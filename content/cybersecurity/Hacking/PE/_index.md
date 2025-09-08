---
title: "🔝 Privilege Escalation"
summary: "Gaining higher-level permissions on compromised systems by exploiting misconfigurations, vulnerabilities, or insecure practices in Linux and Windows environments."
weight: 6
---

<style>
    img {
        transition: transform 0.3s ease;
    }
    img:hover {
        transform: scale(1.05);
    }
</style>

---

## Table of Contents

<div style="background-color: rgba(255, 255, 255, 0.1); padding:10px; border-radius:8px;">
<details>
  <summary>🔎 Manual Enumeration Commands (Linux)</summary>

- [1. Find binaries with special permissions](#1-find-binaries-with-special-permissions)
- [2. Check binary capabilities](#2-check-binary-capabilities)
- [3. List all running processes](#3-list-all-running-processes)
- [4. List listening TCP sockets and their associated processes](#4-list-listening-tcp-sockets-and-their-associated-processes)
- [5. Review scheduled tasks (cron jobs)](#5-review-scheduled-tasks-cron-jobs)
- [6. Find writable directories and files](#6-find-writable-directories-and-files)
- [7. Find files that may contain passwords](#7-find-files-that-may-contain-passwords)
- [8. Find files belonging to a specific group](#8-find-files-belonging-to-a-specific-group)

</details>

<details>
  <summary>🛠 Useful Tools for Privilege Escalation on Linux</summary>

- [🐾 LinPEAS](#-linpeas)
- [🧰 Linux Smart Enumeration (LSE)](#-linux-smart-enumeration-lse)
- [👀 pspy](#-pspy)
- [📚 GTFOBins](#-gtfobins)
- [⚠️ Linux Exploit Suggester](#-linux-exploit-suggester)

</details>
</div>

## 🔎 Manual Enumeration Commands (Linux)
----

### 1. Find binaries with special permissions
**SUID binaries** (execute with the file owner’s privileges):

```
find / -type f -perm -4000 -ls 2>/dev/null
```

```
find / -perm -u=s 2>/dev/null
```

**SGID binaries** (execute with the group’s privileges):

```
find / -type f -perm -g+s 2>/dev/null
```
----

### 2. Check binary capabilities

```
getcap -r / 2>/dev/null
```

```
/usr/sbin/getcap -r / 2>/dev/null
```
----

### 3. List all running processes

```
ps -faux
```
----

### 4. List listening TCP sockets and their associated processes

```
ss -nltp
```
----

### 5. Review scheduled tasks (cron jobs)

```
cat /etc/crontab
```
----

### 6. Find writable directories and files

**Writable directories**:

```
find / -type d -writable | grep -v -E "proc|dev"
```

**Writable files**:

```
find / -type f -writable | grep -v -E "proc|dev"
```

**Writable files excluding system directories**

```
find / -writable ! -path '/proc*' ! -path '/run*' ! -path '/sys*' ! -path '/dev*' -type f 2>/dev/null
```
----

### 7. Find files that may contain passwords

Search text **files** for **keywords** like **password** or **secret**:

```
grep -r -i "password\|secret" / 2>/dev/null
```

Just list **filenames** that contain these **keywords**:

```
grep -rl -i "password\|secret" / 2>/dev/null
```
----

### 8. Find files belonging to a specific group

Find all **files** and **folders** of a **group**

```
find / -group groupname 2>/dev/null
```

Limit search to **home** and **var** folders:

```
find /home /var -group groupname 2>/dev/null
```


<div style="margin-top: 70px;"></div>

## 🛠 Useful Tools for Privilege Escalation on Linux

---

### 🐾 LinPEAS  
Scans the system for possible escalation vectors, including insecure configurations, SUID binaries, running processes, and more.

🔗 [GitHub Repository](https://github.com/carlospolop/PEASS-ng)

**Download:**  
```
wget -q https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
```

---

### 🧰 Linux Smart Enumeration (LSE)  
A script that enumerates system configurations and potential privilege escalation paths efficiently.

🔗 [GitHub Repository](https://github.com/diego-treitos/linux-smart-enumeration)

**Download:**


```
wget "https://github.com/diego-treitos/linux-smart-enumeration/releases/latest/download/lse.sh" -O lse.sh;chmod 700 lse.sh
```

or

```
curl "https://github.com/diego-treitos/linux-smart-enumeration/releases/latest/download/lse.sh" -Lo lse.sh;chmod 700 lse.sh
```

---

### 👀 pspy  
Monitors running processes in real-time without requiring elevated privileges; useful for detecting scheduled tasks or suspicious activity.

🔗 [GitHub Repository](https://github.com/DominicBreuker/pspy)

**Download:**

```
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
```

---

### 📚 GTFOBins  
Not a tool, but an essential reference listing SUID binaries that can be abused for privilege escalation.

🔗 [Official Site](https://gtfobins.github.io/)

---

### ⚠️ Linux Exploit Suggester  
Helps identify potential kernel exploits applicable to the system version.

🔗 [GitHub Repository](https://github.com/mzet-/linux-exploit-suggester)

**Download:**

```
wget https://raw.githubusercontent.com/mzet-/linux-exploit-suggester/master/linux-exploit-suggester.sh -O les.sh
```