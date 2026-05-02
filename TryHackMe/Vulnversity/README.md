# Vulnversity - TryHackMe Writeup

## Overview

This room is a training machine designed to practice penetration testing techniques, including enumeration, vulnerability assessment, exploitation, and privilege escalation.

During this assessment, I performed an Nmap scan using default scripts and version detection in order to identify open services and potential entry points.

I discovered a web application running on a non-standard port and identified a file upload functionality vulnerable to extension filtering bypass.

By exploiting this file upload vulnerability, I was able to gain initial access to the system.

Finally, I achieved root privileges by exploiting a misconfiguration in systemd, allowing execution of a malicious service as root.

## Reconnaissance

To begin the assessment, I performed an Nmap scan to identify open ports and running services on the target machine.

##Nmap Scan

I used the following command:

```bash
nmap -sV -sC -Pn 10.113.168.221 -oA vulnversity
```

The `-sV` flag was used to detect service versions, which helps identify potential vulnerabilities.

The `-sC` flag enabled default Nmap scripts to gather additional information about the services.

The `-Pn` flag was used to skip host discovery and treat the target as alive, ensuring the scan would proceed even if ICMP responses were blocked.

The scan revealed several open ports, including FTP (21), SSH (22), SMB (139/445), a proxy service (3128), and a web application running on port 3333.


<img width="789" height="542" alt="image" src="https://github.com/user-attachments/assets/f000b01f-6b3f-49a8-8159-d2c624929b73" />


## Enumeration

After identifying the web service running on port 3333, I accessed the application through a web browser to manually inspect its functionality.

### Web Application Analysis

The application appeared to be a basic website with limited visible functionality, so I proceeded with directory enumeration to uncover hidden paths.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/06ab982d-0837-4748-84a8-40a6ade7256f" />

### Directory Enumeration

I used Gobuster with a common wordlist:
```bash
gobuster dir -u http://10.113.168.221:3333 -w /usr/share/wordlists/dirb/common.txt
```
This revealed several directories, including `/internal`, which was not directly accessible from the main page.

Upon accessing the `/internal` directory, I discovered a file upload functionality that allowed users to upload files to the server.

<img width="728" height="459" alt="image" src="https://github.com/user-attachments/assets/3664569c-ede9-4ea3-b869-299a4f7f6833" />

## Exploitation

While testing the file upload functionality in the `/internal` directory, I initially attempted to upload `.txt` and `.php` files, but both were rejected with an "Extension not allowed" error.

This indicated that the application was enforcing file extension filtering.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/fe97eba5-d66b-495b-8b2b-f78cf303e64d" />

To bypass this restriction, I tested alternative PHP-related extensions and successfully uploaded a file with a `.phtml` extension, which is also interpreted as PHP by the server.

I then created a reverse shell payload and saved it as `shell.phtml`, which allowed execution of system commands on the target.

After uploading the payload, I started a listener on my machine and accessed the uploaded file via:

http://10.113.168.221:3333/internal/uploads/shell.phtml

This triggered the reverse shell, giving me initial access to the target system as a low-privileged user.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/8ea650fb-26e3-427e-9985-35401e4c724d" />

## Privilege Escalation

After gaining initial access as a low-privileged user, I began enumerating the system for privilege escalation vectors.

I searched for SUID binaries using:
```bash
find / -perm -u=s 2>/dev/null
```
During this process, I identified `/bin/systemctl` as an interesting binary.

<img width="727" height="569" alt="image" src="https://github.com/user-attachments/assets/6457c355-7e51-4d0d-9e14-918376314054" />

Further investigation revealed that it could be leveraged to execute a custom service. I created a malicious systemd service file that executed a reverse shell command.

After reloading systemd and starting the service, the payload was executed with elevated privileges, resulting in a reverse shell as the root user.

This confirmed a misconfiguration that allowed abuse of systemd for privilege escalation.

<img width="682" height="294" alt="image" src="https://github.com/user-attachments/assets/d8266237-acd9-41ec-a342-ffab45c04bca" />

## Conclusion

This machine demonstrated that file upload restrictions based solely on file extensions can often be bypassed by using alternative extensions such as `.phtml`.

It also highlighted the importance of thorough enumeration, both before and after gaining initial access, as it is critical for identifying attack vectors and potential privilege escalation paths.

Overall, this exercise reinforced the need to think creatively when bypassing security controls and to maintain a structured methodology throughout the assessment.
