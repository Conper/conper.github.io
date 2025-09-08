---
title: "Method - HackMyVM"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Method)

To find the IP of the victim machine, I use the **arp-scan** command to discover the devices connected to my local network and locate the one whose** MAC address** starts with **08:**, indicating it is a VirtualBox virtual machine.

```bash
sudo arp-scan -I eth0 --localnet
```

Next, we proceed with a quick scan using **NMAP** to see **which ports are open**.

```bash
sudo nmap -sS --min-rate 4500 -n -Pn 192.168.18.163
```

![1](/images/writeups/method/1.png)
<h6>Ports 22 and 80 are open</h6>

And now we perform a more detailed scan on those **open ports** and save the result in a file called ***target.txt***.

```bash
nmap -sCV -p22,80 -n -Pn 192.168.18.163 -oN target.txt
```

![2](/images/writeups/method/2.png)
<h6>Detailed scan of the open ports</h6>

We observe a website hosted on this machine via **port 80**, but only the default **Nginx** homepage is visible.

I use [**Gobuster**](https://github.com/OJ/gobuster) to enumerate **hidden files** and **directories**. We find a file called ***note.txt*** that tells us the solution is to enumerate.

```bash
gobuster dir -u http://192.168.18.163/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -s 200 -b "" --exclude-length 3690 -x html,txt,xml,bak
```

![3](/images/writeups/method/3.png)

![4](/images/writeups/method/4.png)

I decide to continue enumerating more **directories** and **files** (**zip**, **htm**, **php**, **c**). 

![5](/images/writeups/method/5.png)

I come across the directory **index.htm**, which contains the following.

![6](/images/writeups/method/6.png)

If you view the **page source** (Ctrl + U), you will see a **hidden form** that sends information using the **GET method** to a file called ***secret.php***. This form has an input field named **HackMyVM**, and whatever is typed there is sent to the server when the form is submitted.

![7](/images/writeups/method/7.png)
<h6>Content of the hidden form</h6>

Now, I am going to use [**Burp Suite**](https://portswigger.net/burp/communitydownload) to send a request including all the data we collected earlier.

![8](/images/writeups/method/8.png)
<h6>Send a request</h6>

It tells us to use another method. To do this, I will use the **POST method**. In **Burp Suite**, we have an option to **convert from GET to POST** and **from POST back to GET** with just the click of a button.

![9](/images/writeups/method/9.png)
<h6>Send a request with POST method</h6>

It says we already found it, so let's put some value in the request.

![10](/images/writeups/method/10.png)

And we see that we have access to the victim machine as the user **www-data** (the web server user). I am going to send myself a **Reverse Shell** and set up the **TTY**.

![11](/images/writeups/method/11.png)

Now that we are inside, let's enumerate the existing users on the machine.

```bash
grep /bin/bash /etc/passwd
```

![12](/images/writeups/method/12.png)
<h6>Existing users</h6>

We find two users: **prakasaka** and **root**. 

## Privilege escalation to Prakasaka

I decided to navigate to **prakasaka**’s home directory, and as the **www-data** user, we can read the **User Flag**.

![13](/images/writeups/method/13.png)

The next step to escalate privileges is to read the file ***secret.php***, where we find the **password** for the user **prakasaka**.

![14](/images/writeups/method/14.png)

![15](/images/writeups/method/15.png)
<h6>We are prakasaka</h6>

## Privilege escalation to Root

Next, we run the command **sudo -l** to list the commands that the user **prakasaka** is allowed to execute with elevated **sudo** privileges.

![16](/images/writeups/method/16.png)
<h6>Privilege escalation with the ip binary</h6>

We have permission to execute ***/bin/ip*** with **root privileges**. To escalate to **root**, let’s consult [**GTFObins**](https://gtfobins.github.io/) and use the recommended commands for ***/bin/ip***. Specifically, we will run the following commands to gain **root** access:

```bash
sudo ip netns add foo
sudo ip netns exec foo /bin/sh
sudo ip netns delete foo
```

We are finally the **root** user and can obtain the **Root Flag**.

![17](/images/writeups/method/17.png)
