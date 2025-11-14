---
title: "🕵️ Reconnaissance & Enumeration"
summary: "Gathering information about the target through passive and active methods, including OSINT, network scanning, fuzzing, and service discovery."
weight: 1
---

<style>
    img {
        transition: transform 0.3s ease;
    }
    img:hover {
        transform: scale(1.05);
    }
</style>

<div style="margin-top: 70px;"></div>


<h1 style="text-align:center;">Reconnaissance</h1>


<div style="margin-top: 70px;"></div>


## Passive
---

### WHOIS

Provides information about domain registration and ownership.

```
whois example.com
```

Online tools:

- [**domaintools**](https://whois.domaintools.com/)

- [**whois**](https://www.whois.com/whois/)

### DNS Lookup

Retrieves **IP addresses**, **subdomains**, **mail** **servers**, and **DNS** records.

Online tools:

- [**DNSLookup**](https://mxtoolbox.com/DNSLookup.aspx)

- [**dnschecker**](https://dnschecker.org/)

### Social Media

Collect info about **employees**, **technologies**, or **organizational structure**.


**LinkedIn** → find employee roles and structure

**Twitter** → company announcements and tech stack hints

**Facebook** → office locations, events, or technologies

[**TheHarvester**](https://github.com/laramies/theHarvester) → to gather emails & usernames

### Shodan / Fofa.info

Search Internet-connected devices for exposed servers, webcams, IoT, etc.

<div style="display: flex; justify-content: center;">
  <img src="/images/hacking/shodan.png" alt="Mi foto" width=65% style="border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,0,0.3); margin-bottom: 50px;"/>
</div>



Links:

- [**Shodan**](https://www.shodan.io/)

- [**Fofa.info**](https://fofa.info/)

### Google Dorking

Use advanced search operators to discover sensitive info online.

Examples:

```
filetype:pdf site:example.com "confidential"
intitle:"index of" passwords
```

Build and test advanced Google dorks easily with [**DorkSearch**](https://dorksearch.com/)

<div style="margin-top: 70px;"></div>


## Active
---

### Ping

Check if a host is alive and measure latency.  

**Example commands:**

```
ping example.com           # Ping a domain
ping -c 4 192.168.1.1      # Ping a host 4 times
```

> 📝 **NOTE:**Ping sends ICMP echo requests to see if the target responds. Useful for basic network reachability tests.

Online tools:

- [**ping.eu:**](https://ping.eu/)


### Dig / NSLookup

Actively query DNS records of the target. Useful when passive info is incomplete.

```
# Using nslookup
nslookup example.com
nslookup -type=MX example.com   # Check mail servers
nslookup -type=NS example.com   # Check name servers

# Using dig
dig example.com                  # Basic A record (IP)
dig example.com ANY              # All DNS records
dig MX example.com               # Mail servers
dig NS example.com               # Name servers
```

<div style="margin-top: 70px;"></div>

<h1 style="text-align:center;">Enumeration</h1>



<div style="margin-top: 70px;"></div>

## Network & Host
---

### Nmap

Nmap scans a target to find **open ports** and shows which **services** are running on them. It can also tell the **operating system** of the target. This helps to know which parts of the system are reachable and what could be vulnerable. You can use it to see if **firewalls** or filters are blocking connections and to plan further tests.

<div style="display: flex; justify-content: center;">
  <img src="/images/hacking/nmap.jpg" alt="Mi foto" width=65% style="border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,0,0.3); margin-bottom: 50px;"/>
</div>

Example commands:

```
nmap example.com            # Basic port scan
nmap -sV example.com        # Service version detection
nmap -O example.com         # OS detection
nmap -p- example.com        # Scan all ports
nmap -A example.com         # Aggressive scan (OS, services, scripts)
```

Example scan for hacking:

```
nmap -sS -p- --min-rate 4500 -n -Pn -vvv example.com

nmap -sCV -p22,80,3306 -n -Pn example.com -oN target.txt
```

Official website: [**nmap.org**](https://nmap.org/)

## Web Fuzzing
---
### Gobuster

A brute force tool used to find **directories**, **files**, and **subdomains** on web servers. It runs from the terminal using **wordlists**, making it fast since it doesn’t rely on a browser.

**Example Commands:**

```
# Basic directory scan
gobuster dir -u http://example.com/ -w /usr/share/wordlists/dirbuster/common.txt -x txt, php, html, py

# More complete scan with output file
gobuster dir -u http://example.com/ -x txt,php,html,bak --wordlist /usr/share/wordlists/dirb/common.txt -o dir.log

# Add slash at the end of directories to detect them properly
gobuster dir -u http://example.com/ -w /usr/share/wordlists/dirb/common.txt -x txt,php,html,bak -t 200 --add-slash

# Handle response length errors and exclude specific lengths
gobuster dir -u http://example.com/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -s 200 -b "" --exclude-length 3690 -x html,txt,xml,bak
```

### Wfuzz

A web fuzzer that’s very flexible for testing different types of inputs in web applications, such as **GET/POST** parameters, **cookies**, or **headers**. It helps discover **directories**, **files**, and vulnerabilities by trying combinations of words and patterns.

<div style="display: flex; justify-content: center;">
  <img src="/images/hacking/wfuzz.png" alt="Mi foto" width=65% style="border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,0,0.3); margin-bottom: 50px;"/>
</div>

**Example Commands:**

```
# Fuzzing a GET parameter
wfuzz -c -w /usr/share/wordlists/dirb/big.txt -u http://example.com/route/file.php?FUZZ=whoami --hw 0

# Brute force on a POST form parameter
wfuzz -c -w /usr/share/wordlists/rockyou.txt -u http://example.com/ -d password=FUZZ --hw=140

# Brute force login form with username and password
wfuzz -c -w /usr/share/wordlists/rockyou.txt -u 'http://example/admin/' -d 'username=admin&password=FUZZ&login=' --hw=147

# Fuzzing files with custom list
wfuzz -c -z file,/usr/share/wordlists/rockyou.txt -u 'http://example.com/route/file.php?FUZZ=ls' --hl=0
```

### FFUF

A **lightweight** and **fast** web fuzzing tool for discovering **paths**, **files**, and **subdomains** using wordlists. Similar to Gobuster but optimized for speed and simplicity.

**Example Commands:**

```
# Fuzz directories with rate limit
ffuf -u http://example.com/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -fs 25 -e '.php' -rate 1 | grep -v 403

# Fuzz a parameter using POST request file
ffuf -w /usr/share/SecLists/Discovery/Infraestructure/common-http-ports.txt -request post.req -u http://example.com/upload -fs 61

# Brute force POST request with JSON payload (HTTPS)
ffuf -u 'http://example.com/session' -w /usr/share/wordlists/rockyou.txt -d '{"username":"jose","password":"FUZZ"}' -H 'Content-Type: application/json' -fs 31
```

### Feroxbuster

A **recursive** web **directory** and **file** discovery tool. It’s designed to handle large scans efficiently, automatically following discovered paths to uncover deeper hidden content.

**Example Command:**


```
# Scan a target using a custom wordlist with specific file extensions appended.
feroxbuster -u http://example.com/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html

# Recursive scan with status code filtering:
feroxbuster -u http://example.com -r -s 200,301,302
```

### Dirsearch

A CLI tool for brute forcing **directories** and **files** on web servers. It’s simple, effective, and widely used for web content enumeration.

```
dirsearch -u http://example.com/ -x 404
```

<div style="margin-top: 70px;"></div>

## Domains & Subdomains
---

Enumerate **subdomains** and gather DNS-related information. Useful for finding **hidden services**, misconfigured **domains**, or additional attack surfaces.

### Gobuster

**Gobuster** can enumerate **subdomains** using the `vhost` mode.

```
# Enumerate subdomains
gobuster vhost --append-domain -u http://example.com/ -w /usr/share/wordlists/dirb/common.txt

gobuster vhost --append-domain -u https://example.com/ -w /usr/share/wordlists/dirb/common.txt -k
```

> 📝 **NOTE**: The `-k` option (`--no-tls-validation`) tells the tool to ignore TLS/SSL certificate validation. This is useful when scanning HTTPS sites with self-signed or expired certificates, as it allows Gobuster to continue without failing.

### Wfuzz

**Wfuzz** allows brute forcing **subdomains** via the `Host` header.

```
# Enumerate subdomains
wfuzz -c -u IP -H "HOST: FUZZ" -w subdomains.txt

# Example
wfuzz -H "Host: FUZZ.example.com" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://example.com --hw=12

```

### Fuff

```
# Enumerate subdomains
ffuf -c -u "http://example.com" -H "Host: FUZZ.example.com" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -fs 201
```