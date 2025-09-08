---
title: "Thirteen - HackMyVM"
summary: "Difficulty: Very Easy 🔵"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Thirteen)

To begin with, we perform a **quick scan** with [**Nmap**](https://nmap.org/) to see which **ports** are **open**.

```
sudo nmap -sS -p- --min-rate 4500 -n -Pn -vvv 192.168.1.233
```

![01](/images/writeups/thirteen/1.png)

Now with these **open ports**, we will run a more **intensive scan** to check which **services** are running behind them and their respective **versions**, saving the result in a file called ***target***.

![02](/images/writeups/thirteen/2.png)

This version of **FTP** is **not vulnerable to anonymous login**, so what we’ll do is access the website hosted on the victim machine.

![03](/images/writeups/thirteen/3.png)

When entering, we find this page, and when we click on any of the buttons at the bottom, we **can view the contents of a file**.

![04](/images/writeups/thirteen/4.png)

Other than that, I don’t see anything interesting, so with the [**Gobuster**](https://github.com/OJ/gobuster) tool we will **enumerate subdirectories** and **files** to see what we can find.

![05](/images/writeups/thirteen/5.png)

Now we can start to understand what might be happening behind the "***themes***" parameter we saw earlier, because when we access ***/config.txt***, we see **the same content**.

![06](/images/writeups/thirteen/6.png)

The file name passed through the parameter **is encoded** in some way, and if we pay attention, we see that the letters correspond to others and with a certain logic.
In the end, we discovered that it is encoded with [**ROT13**](https://simple.wikipedia.org/wiki/ROT13).

![07](/images/writeups/thirteen/7.png)

I try reading a system file like ***/etc/passwd***, and it works. It is vulnerable to [**LFI**](https://en.wikipedia.org/wiki/File_inclusion_vulnerability)(Local File Inclusion).

![08](/images/writeups/thirteen/8.png)

We can’t access the ***/logs*** directory, but if we try enumerating inside it with **Gobuster** to see what's there, we find a file.

![09](/images/writeups/thirteen/9.png)

They are the **FTP server logs**, where we see that two files were uploaded, which could be very useful for us.

![10](/images/writeups/thirteen/10.png)

The first file is a **Python** program that starts the **FTP server** and **logs** all connections and uploaded or downloaded files to a file.

![11](/images/writeups/thirteen/11.png)

It also gives us the password to access.

![11.5](/images/writeups/thirteen/11.5.png)

The second one is a reverse shell.

![12](/images/writeups/thirteen/12.png)

It looks like the ***rev.sh*** file is trying to give us a hint of where this is going.
What we could do is insert a **Reverse Shell** inside the ***ftp_server.py*** itself, so when the machine boots up and runs the script to start the **FTP server**, the **Reverse Shell** executes as well, granting access to the machine.

![13](/images/writeups/thirteen/13.png)
<h6>Revshell created in: <a href="https://www.revshells.com/">Revshells.com</a></h6>

I **create the script with the payload** included on my local machine, access via **FTP**, and replace it.

![14](/images/writeups/thirteen/14.png)
<h6>Insert the payload</h6>

![15](/images/writeups/thirteen/15.png)

We also see that the files belong to the **root** user.

Now, just start listening on ***port 4444***, **restart** the victim machine, and wait for the **shell** to connect.

![16](/images/writeups/thirteen/16.png)

We get direct access as **Root** and capture the flags.

![17](/images/writeups/thirteen/17.png)

Machine completed!! 💻🔥
