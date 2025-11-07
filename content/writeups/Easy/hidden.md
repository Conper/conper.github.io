---
title: "Hidden - HackMyVM"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Hidden)

To start, we will perform a port scan to identify which ports are open.

![1](/images/writeups/hidden/1.png)
<h6>NMAP port scan</h6>

So the first thing I do is go to the website hosting this machine. And we come across a challenge:

![2](/images/writeups/hidden/2.png)
<h6>Challenge 1</h6>

![3](/images/writeups/hidden/3.png)
<h6>What is observed when inspecting the page</h6>

I decide to search on [**dCode**](https://www.dcode.fr/) for the type of symbol encryption the **level 1 puzzle** could be, and I finally find it:

![4](/images/writeups/hidden/4.png)
<h6>Cipher with Rosicrucian Cipher</h6>

![5](/images/writeups/hidden/5.png)
<h6>Decryption</h6>

We decrypt it and obtain **SYS.HIDDEN.HMV**, which seems to be like a domain. So, I decide to save it in ***/etc/hosts***:

![6](/images/writeups/hidden/6.png)


Now I search again for the website, and we come across **level 2**:

![7](/images/writeups/hidden/7.png)
<h6>Challenge 2</h6>

As I don't see anything on the page, I decide to use **Gobuster** to enumerate subdirectories.

![8](/images/writeups/hidden/8.png)
<h6>Enumeration of subdirectories with Gobuster</h6>

It gives us this result, but ***/users*** and ***/members*** are just a rabbit hole, and the directory that will be useful is ***/weapon***. Although it is initially empty, when scanning it again with Gobuster, we find that it contains a **PHP file**.

![9](/images/writeups/hidden/9.png)
<h6>We found an interesting PHP file</h6>

After trying for a while to view the code within **loot.php** and not making any progress, along with other futile attempts, I decide to use **wfuzz** to check if we can **execute any command**s with this **PHP file**.

![10](/images/writeups/hidden/10.png)
<h6>We found the keyword to execute commands</h6>

And we find the keyword that we should use to **execute commands**. This way, we can now access the machine by running a ***Reverse Shell***.

![11](/images/writeups/hidden/11.png)
<h6>Executing commands as www-data</h6>

As we can see, we can execute commands as ***www-data***, so we set up a listener on port **4444** and execute the ***Reverse Shell***.

![12](/images/writeups/hidden/12.png)
![13](/images/writeups/hidden/13.png)
<h6>We are inside the machine</h6>

I decide to set up the **TTY** for convenience, and it would look something like this.

![14](/images/writeups/hidden/14.png)

Let's see which users exist on this machine.

```bash
grep /bin/bash /etc/passwd
```
![15](/images/writeups/hidden/15.png)
<h6>System users</h6>

And we observe that there are three users: 
- ***toreto***
- ***atenea***
- ***root***

Now, by running **sudo -l**, we see that we can execute **Perl** as the user ***toreto***. 

![16](/images/writeups/hidden/16.png)


This allows us to escalate privileges to become ***toreto*** quite easily. The first thing we need to do is go to a directory where we have write permissions, such as the ***/tmp*** directory, and there we will create a **Perl file** and execute it as the ***toreto*** user. 

Here is the ***Perl code***:

```bash
echo -ne '#!/bin/perl \nuse POSIX qw(setuid); \nPOSIX::setuid(0); \nexec "/bin/bash";' > script.pl
```

And we execute it like this:

```bash
sudo -u toreto /usr/bin/perl script.pl
```

This will allow us to obtain the shell of ***toreto***.

![17](/images/writeups/hidden/17.png)
<h6>We are the user toreto</h6>

I've tried **sudo -l**, looking to see if any **SUID binaries** could be useful, and I've even checked the **capabilities**, but found nothing. So, I decided to search through directories, and in the end, I found a file that will help us escalate privileges to the ***atenea*** user.

![18](/images/writeups/hidden/18.png)
<h6>We found a text file</h6>

The text file is a dictionary.

![19](/images/writeups/hidden/19.png)

So I use **Hydra** to perform a ***brute-force attack*** on the **SSH port** (22) with the user ***atenea***, using this dictionary to check if any combination is the correct password.

![20](/images/writeups/hidden/20.png)
<h6>We found the password for atenea</h6>

And the password is **sys8423hmv**, so we log in via **SSH**:

![21](/images/writeups/hidden/21.png)
<h6>We are atenea</h6>

Now we can obtain the **user flag**:

![22](/images/writeups/hidden/22.png)

Next, let's ***escalate privileges*** to become the ***root*** user. By running **sudo -l**, we see that we can execute **socat** as if we were ***root*** without password.

![23](/images/writeups/hidden/23.png)

I search on [**GTFOBins**](https://gtfobins.github.io/) and find that we can easily ***escalate privileges***.

![24](/images/writeups/hidden/24.png)

```bash
sudo socat stdin exec:/bin/sh
```

We run it, and there you go, we are now ***root***.

![25](/images/writeups/hidden/25.png)

And we obtain the ***root flag***.

![26](/images/writeups/hidden/26.png)

Machine pwned!!