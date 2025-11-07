---
title: "Warez - HackMyVM"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Warez)

When the machine starts, it displays its IP address; in this case, it is 192.168.18.190.

![1](/images/writeups/warez/1.png)
<h6>The victim machine's IP address</h6>

To begin, we will perform a scan using the **NMAP** tool, which will help us identify open ports on the target machine. First, we will use the following command for a **quick scan** that informs us about the **open ports**. We will scan all ports (**65535**) without depth.

```bash
sudo nmap -sS -p- --min-rate 4500 -n -Pn 192.168.18.190
```

![2](/images/writeups/warez/2.png)

And now, with the **open ports** identified, we will perform a more **detailed scan**:

```bash
nmap -sCV -p22,80,6800 -n -Pn 192.168.18.190 -oN target
```

![3](/images/writeups/warez/3.png)

We observe that port 6800 is running [**Aria2**](https://github.com/aria2/aria2), which is an **open-source downloader** that supports various protocols such as **HTTP**, **FTP**, **BitTorrent**.

By accessing **port 80**, we obtain the username **Carolina**.

![4](/images/writeups/warez/4.png)
<h6>User carolina</h6>

Next, we notice that by selecting '**By URLs**' within '**Add**', we can **upload files** to any directory. So, I am going to copy my **id_rsa.pub** to a file named ***authorized_keys*** and upload it, allowing the victim machine to recognize our machine as authorized, specifically as the user **Carolina**.

```bash
cp id_rsa.pub authorized_keys
```

If you don't have the ***id_rsa.pub*** file, you can create it with the following command:

```bash
ssh-keygen
```

And with **Python**, we start a web server on **port 8080**:

```bash
python3 -m http.server 8080
```

![5](/images/writeups/warez/5.png)

Now, we put the URL where our file is located and change the destination directory to ***/home/carolina/.ssh***.

![6](/images/writeups/warez/6.png)

We send it, and now we can access via **port 22** (**SSH**) as the user **carolina**.

![7](/images/writeups/warez/7.png)
<h6>We are now Carolina</h6>

Let's check how many users are on the machine. It appears that there are only two users: **carolina** and **root**.

```bash
grep /bin/bash /etc/passwd
```

![8](/images/writeups/warez/8.png)
<h6>User on the machine</h6>

We get the user flag, and now it’s time to move on to privilege escalation.

## Privilege escalation to Root

We enumerate the **SUID binaries** on the machine, and there is one that we can use to escalate to **root** very easily. The relevant binary is **rtorrent**.

![9](/images/writeups/warez/9.png)

We search in [**GTFObins**](https://gtfobins.github.io/), and it provides a way to escalate to **root** using this **SUID binary** by running the following command:

![11](/images/writeups/warez/11.png)
<h6>Command for privilege escalation</h6>

```bash
echo "execute = /bin/sh,-p,-c,\"/bin/sh -p <$(tty) >$(tty) 2>$(tty)\"" >~/.rtorrent.rc 
/usr/bin/rtorrent
```

![12](/images/writeups/warez/12.png)
<h6>We are root</h6>

Now that we have **root** access, let’s grab the **root flag** and complete the machine

![13](/images/writeups/warez/13.png)