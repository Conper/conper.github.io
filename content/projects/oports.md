---
title: "Opened Ports"
summary: "TicTacToe game created in Android Studio with Kotlin"
date: 2023-08-06
showToc: true
TocOpen: true
cover:
    image: "/images/projects/oports.png"
    hiddenInSingle: true
---

[GitHub Repository](https://github.com/Conper/Opened-Ports)

## Description
**Opened-Ports** is a Linux script that uses **Nmap** to scan a target and extract only the most relevant information:

- Open ports
- Services running on those ports
- Service versions
- Detected operating system

It can:
- Analyze a previously saved Nmap scan result file.
- Perform a new scan from scratch and then analyze the results.

> 📝 **Note:** Designed for Linux. Functionality on other operating systems is not guaranteed.

<div style="display: flex; justify-content: center;">
  <img src="/images/projects/oports.png" alt="CTF Machines" width="200" style="border-radius: 10%; box-shadow: 0 0 10px rgba(0,0,0,0.3); margin-bottom: 50px;"/>
</div>


---

## Installation (Linux)

1. Download the script:

```bash
wget https://raw.githubusercontent.com/Conper/Opened-Ports/main/oports.sh
mv ./oports.sh /usr/bin/oports
chmod +x /usr/bin/oports
```
