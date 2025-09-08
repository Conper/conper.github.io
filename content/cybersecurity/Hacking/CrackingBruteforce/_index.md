---
title: "🔐 Cracking & Brute Force"
summary: "Breaking passwords and encryption through dictionary attacks, brute-force methods, hash cracking, and credential stuffing techniques."
weight: 4
---

## Table of Contents

<div style="background-color: rgba(255, 255, 255, 0.1); padding:10px; border-radius:8px;">

<details>
  <summary>🔑 Password Cracking</summary>

- [Tools](#tools)
- [Hash types](#hash-types)
- [Cracking Modes](#cracking-modes)

</details>

<details>
  <summary>💥 Brute Force / Dictionary</summary>

- [Tools](#tools-1)
- [Hydra Examples](#hydra-examples)
- [Medusa Examples](#medusa-examples)
- [Ncrack Examples](#ncrack-examples)

</details>

<details>
  <summary>📂 Cracking Specific Files and Systems</summary>

- [Decrypting id_rsa SSH private keys](#decrypting-id_rsa-ssh-private-keys)
- [Decrypting passwords from /etc/shadow](#decrypting-passwords-from-etcshadow)
- [Decrypting KeePass database passwords](#decrypting-keepass-database-passwords)
- [Decrypting ZIP file passwords](#decrypting-zip-file-passwords)
- [Decrypting protected PDF files](#decrypting-protected-pdf-files)

</details>

</div>


## Password Cracking

**Password cracking** is the process of recovering passwords from data that has been stored in or transmitted by a computer system.


---

### Tools

- **Hashcat**: Advanced password recovery tool supporting GPU acceleration.

- **John the Ripper**: Popular CPU-based password cracking tool with support for multiple hash types.

<div style="display: flex; justify-content: center;">
  <img src="/images/hacking/johntheripper.png" alt="Mi foto" width=65% style="border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,0,0.3); margin-bottom: 50px;"/>
</div>

---

### Hash Types

<div style="display: flex; justify-content: center;">
  <img src="/images/hacking/hashingalg.png" alt="Mi foto" width=65% style="border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,0,0.3); margin-bottom: 50px;"/>
</div>

**MD5**: Fast but insecure, widely used for legacy systems.

```
Example: 5f4dcc3b5aa765d61d8327deb882cf99 → "password"
```

**SHA1**: Slightly more secure than MD5, but also vulnerable.

```
Example: 5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8 → "password"
```

**NTLM**: Used by Windows systems; weak against modern attacks.

```
Example: 8846f7eaee8fb117ad06bdd830b7586c → "password"
```

**bcrypt**: Strong hashing algorithm with built-in salt and slow computation.

```
Example: $2y$12$EXRkfkdmXn2gzds2SSitu.JG3r8sG3sJ3JDZf4F9nYq5YyNUI4/9e → "password"
```

Use [**mattw.io**](https://mattw.io/hashID/types) to find the correct hash **mode number** (**Hashcat**) or **format name** (**John the Ripper**) before starting the attack.

Hash identifier: [**dCode**](https://www.dcode.fr/hash-identifier)

---

### Cracking Modes

**Dictionary Attack**: Tests each word from a wordlist against the hashes. Very fast if the password is a common or weak word.

- **Hashcat**:

```
hashcat -m 0 -a 0 hashes.txt wordlist.txt
```

Uses MD5 (-m 0) in dictionary mode (-a 0) with wordlist.txt.

- **John the Ripper**:

```
john --wordlist=wordlist.txt hashes.txt
```

Uses the specified wordlist to try matching hashes.

---

**Brute-force Attack**: Tries all possible combinations of characters until the correct one is found. Very slow for long passwords.

- **Hashcat**:

```
hashcat -m 0 -a 3 hashes.txt ?a?a?a?a
```

Brute-forces all printable ASCII characters (?a) for 4-character passwords.

- **John the Ripper**:

```
john --incremental=All hashes.txt
```

Tests all character combinations; “All” means all printable characters.

---

**Mask Attack**: Similar to brute-force but uses a known pattern to reduce the search space. Much faster when part of the password format is known.

- **Hashcat**:

```
hashcat -m 0 -a 3 hashes.txt ?u?l?l?l?l?d?d
```

Pattern: 1 uppercase letter (?u), 4 lowercase letters (?l), and 2 digits (?d).

- **John the Ripper**:

```
john --mask='?u?l?l?l?l?d?d' hashes.txt
```

Same mask pattern, but using John’s syntax.


<div style="margin-top: 70px;"></div>

## Brute Force / Dictionary

**Brute force** attacks involve systematically trying **every possible combination** of credentials until the correct one is found. **Dictionary attacks** are similar but use a **predefined list** of likely passwords (a **dictionary**) instead of trying every combination.

---
### Tools:

- [**Hydra**](https://github.com/vanhauser-thc/thc-hydra): Fast and flexible password-cracking tool for multiple protocols.

- [**Medusa**](https://github.com/jmk-foofus/medusa): Parallelized login brute-forcer with wide protocol support.

- [**Ncrack**](https://github.com/nmap/ncrack): High-speed network authentication cracking tool, designed for large-scale network audits.

---

### Hydra Examples

**SSH:**  
```
hydra -l admin -P rockyou.txt ssh://example.com
```

**HTTP POST login form:**

```
hydra -l user -P rockyou.txt example.com http-post-form "/login:username=^USER^&password=^PASS^:Login failed"
```

**FTP:**

```
hydra -l root -P passwords.txt ftp://example.com
```

**SMB:**

```
hydra -L users.txt -P passwords.txt smb://example.com
```

---

### Medusa Examples

**FTP:**

```
medusa -h example.com -u root -P passwords.txt -M ftp
```

**RDP:**

```
medusa -h example.com -u administrator -P passwords.txt -M rdp
```

---

### Ncrack Examples

**SSH:**

```
ncrack -p 22 -u admin -P rockyou.txt example.com
```

**RDP:**

```
ncrack -p 3389 -u administrator -P passwords.txt example.com
```

<div style="margin-top: 70px;"></div>

## Cracking Specific Files and Systems

This section shows how to crack passwords from different types of files and systems using John the Ripper. 

---
### Decrypting `id_rsa` SSH private keys


```
ssh2john id_rsa > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
john --show hash
```
---
### Decrypting passwords from `/etc/shadow`


- **Option A**: With ***/etc/passwd*** (**recommended**)

```
unshadow passwd.txt shadow.txt > unshadowed.txt
john --wordlist=/usr/share/wordlists/rockyou.txt unshadowed.txt
```

or specifying format:

```
john --wordlist=/usr/share/wordlists/rockyou.txt unshadowed.txt --format=crypt
```

- **Option B**: Only ***/etc/shadow***

```
john --wordlist=/usr/share/wordlists/rockyou.txt shadow.txt
```

> 📝 **NOTE**: Using only ***/etc/shadow*** works, but **John** will not have the **full usernames**; output may show only **hashes**.

---
### Decrypting `KeePass` database passwords


```
keepass2john dataset.kdbx > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

> 🚨 IMPORTANT: If you want to create a vulnerable .kdbx for testing, use this version:

```
wget https://github.com/keepassxreboot/keepassxc/releases/download/2.3.4/keepassxc-2.3.4-x86_64.AppImage
```
---
### Decrypting `ZIP file` passwords


```
zip2john archivo.zip > hash
john hash
```
---
### Decrypting protected `PDF files`


```
pdf2john protected.pdf > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```
