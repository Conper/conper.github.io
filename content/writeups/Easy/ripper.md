---
title: "Ripper - HackMyVM"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Ripper)

To begin, let's find the IP of our target machine using arp-scan by executing the following command:

```bash
arp-scan -I eth0 --localnet
```

> 📝 **NOTE:** Normally, the MAC addresses of VirtualBox virtual machines start with 08:


Now that we know its IP **192.168.18.169**, let's perform a port scan using **NMAP** to identify which ports are **open**, their **services**, and more information.

![01](/images/writeups/ripper/1.png)

<h6>NMAP scan</h6>

So now that we know it has ports **22** and **80** open, let's access the web page hosted on this machine to see if we can find any useful information.

![02](/images/writeups/ripper/2.png)

It tells us that the website is under maintenance, nothing more. So, I decide to use [**Gobuster**](https://github.com/OJ/gobuster) to enumerate subdirectories, and I come across a text file named ***staff_statements.txt***:

![03](/images/writeups/ripper/3.png)
<h6>Gobuster enumeration</h6>

![04](/images/writeups/ripper/4.png)
<h6>staff_statements.txt</h6>

The site is not yet repaired. Technicians are working on it by using **old SSH connection files**. Although the message doesn't specify which files, I inferred it could involve older key files like ***id_rsa.bak*** rather than the current ***id_rsa***. I searched for these old files, found them, and proceeded to download them.

![05](/images/writeups/ripper/5.png)
<h6>We download id_rsa.bak</h6>

Now that we have ***id_rsa.bak***, we still need to know the **username** to log in. We already know the username since it is displayed when the machine starts:

![06](/images/writeups/ripper/6.png)
<h6>User Jack</h6>

Before anything else, we need to grant permissions to the ***id_rsa.bak*** file with the following command:

```bash
chmod 600 id_rsa.bak
```

But when trying to access via **SSH**, it asks for a **password**, preventing us from entering.

![07](/images/writeups/ripper/7.png)
<h6>It prevents us from accessing.</h6>

So, we can use **John the Ripper** to crack the password.

```bash
ssh2john id_rsa.bak > hash
```

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

![08](/images/writeups/ripper/8.png)
<h6>John the Ripper result</h6>

Now that we know the password, we can access.

![09](/images/writeups/ripper/9.png)
<h6>We are the user Jack</h6>

I decide, first of all, to check the users on this machine.

```bash
grep /bin/bash /etc/passwd
```

![10](/images/writeups/ripper/10.png)
<h6>Users on the machine</h6>

And we find 3 users: **jack**, **helder**, **root**. 

Now it's time for privilege escalation. In this case, we need to perform 2 privilege escalations to become the root user.

## Privilege escalation to helder

Now, let's use [**LinPEAS**](https://github.com/peass-ng/PEASS-ng), a tool for enumeration and security auditing. It focuses on identifying files with the **SUID** (Set User ID) and **SGID** (Set Group ID) permission bits set, which can potentially be exploited by attackers to escalate privileges and gain unauthorized access.

We download **LinPEAS** with the following command:

```bash
wget -q https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
```

![11](/images/writeups/ripper/11.png)


We grant permissions for its execution and run it, saving the results in a file named ***LinpeasLog***.

```bash
./linpeas.sh > LinpeasLog
```

And after some time of searching, I come across the following:

![12](/images/writeups/ripper/12.png)

> 📝 **NOTE:** The **opasswd** file stores users' old passwords in Linux to prevent them from reusing them when changing their password.


Although it says it was **Jack**’s password, since some people reuse passwords (which is bad practice) we decided to check if it works for **Helder**. I tried it, and it worked. We obtained the user flag.

![13](/images/writeups/ripper/13.png)

Upon realizing that [**LinPEAS**](https://github.com/peass-ng/PEASS-ng) didn't reveal anything new, I decide to use [**pspy**](https://github.com/DominicBreuker/pspy). It is a **process monitoring tool** that provides real-time visibility into running processes without requiring elevated privileges. I downloaded it to my local machine and transferred it to the target machine by hosting a simple **Python** web server.

![14](/images/writeups/ripper/14.png)

Now we grant permissions and execute it.

![15](/images/writeups/ripper/15.png)

![16](/images/writeups/ripper/16.png)
<h6>Result</h6>

We discovered a script that runs every minute as the **root** user (UID 0) and uses **netcat** to connect to localhost on **port 10000**, saving the data exchanged over the connection to a file named out in ***/root/.local/***. Afterwards, it compares the contents of ***/root/.local/helder.txt*** and ***/home/helder/passwd.txt***. If they match, it sets the **SUID** bit on the executable located in ***/usr/bin/***, with the filename specified by the content of the out file.

To achieve this, the first step is to create the ***passwd.txt*** file in ***/home/helder*** as a symbolic link pointing to ***/root/.local/helder.txt***. This ensures that ***passwd.txt*** contains the same content as ***helder.txt***, so the condition will be true.

```bash
ln -s /root/.local/helder.txt /home/helder/passwd.txt
```

> ⚠️ **IMPORTANT**: A symbolic link is like a shortcut that points to another file. When a program accesses the link, it actually reads the file it points to. So even if you don’t own the original file, the program can access it through the link.


Then, we check with the following command to confirm that it indeed points to ***/root/.local/helder.txt***.

```bash
ls -la /home/helder/passwd.txt 
```

Now, we listen on **port 10000** and send the text '**bash**'. When the script runs, it will save this input to the out file and set the **SUID** bit on the corresponding executable in **/usr/bin/**. In this case, we're **granting SUID permissions to bash**.

```bash
echo 'bash' | nc -lvnp 10000
```

![17](/images/writeups/ripper/17.png)
<h6>What I mentioned can be observed in the code</h6>

After executing the exploit, we notice a brief connection. Now, ***/usr/bin/bash*** has the **SUID** bit set. Simply running the command **bash -p** grants us root privileges.

```bash
bash -p
```

![18](/images/writeups/ripper/18.png)
<h6>All the previous process</h6>

Finally, we navigate to the ***/root*** directory and obtain the **root flag**.

![19](/images/writeups/ripper/19.png)
