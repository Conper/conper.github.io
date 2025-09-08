---
title: "Beloved - HackMyVM"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Beloved)

We find the victim machine’s IP address using the **arp-scan** tool, looking for a MAC address starting with **08:**, which is typical for virtual machines.

Next, we run an NMAP scan to identify open ports.

![01](/images/writeups/beloved/1.png)

As a result, we observed that ports **22** (**SSH**) and **80** (**HTTP**) are open. Inside the web, we can see the following.

![02](/images/writeups/beloved/2.png)

If we click on any link, we notice that it doesn't load because it's a domain that our local machine is not aware of.

![03](/images/writeups/beloved/3.png)
<h6>Our machine is not aware of that domain</h6>

So, we are going to edit the ***/etc/hosts*** file and add this domain with its corresponding IP:

```bash
sudo nano /etc/hosts
```

![04](/images/writeups/beloved/4.png)

Now, upon entering the victim machine's IP address again in Firefox, we see a website with an improved appearance.

![05](/images/writeups/beloved/5.png)

Clicking on "Hello world!" we encounter a user and a comment.

![06](/images/writeups/beloved/6.png)
<h6>User</h6>

![07](/images/writeups/beloved/7.png)
<h6>Comment</h6>


As we have seen earlier, the website is built with **WordPress**. So, let's use the [**wpscan**](https://github.com/wpscanteam/wpscan) tool to gather more information. Using this tool, we can confirm the existence of the user '**smart_ass**'.

Enumerating users:

```bash
wpscan --url http://beloved/ -e u
```

![08](/images/writeups/beloved/8.png)
<h6>Result of user enumeration</h6>

Now, with the following command, we will enumerate the **WordPress plugins** used by this website:

```bash
wpscan --url http://beloved/ --plugins-detection aggressive -t 50 --api-token='YOUR_API'
```

> 📝 **NOTE**: To get the API key, you must register at [**wpscan.com**](https://wpscan.com/)

As a result, we can observe a plugin called [**wpDiscuz**](https://wpdiscuz.com/) with an old version that is **vulnerable**.

![09](/images/writeups/beloved/9.png)

I go to [**Metasploit**](https://www.metasploit.com/) and find that it has an exploit that we can use for **RCE** (**Remote Code Execution**).

![10](/images/writeups/beloved/10.png)
<h6>Exploit</h6>

![11](/images/writeups/beloved/11.png)
<h6>Setting up the options</h6>

And we execute it with the **run** or **exploit** command, and now we would be inside:

![12](/images/writeups/beloved/12.png)

We use the '**shell**' command to spawn a shell and then configure the **TTY**.

![13.5](/images/writeups/beloved/13.5.png)

## Privilege escalation to beloved

When navigating to the ***/var/www*** directory, we can access the ***.bash_history*** file and review some executed commands.

![13](/images/writeups/beloved/13.png)

Later, after running **sudo -l**, we see that we are allowed to execute the **nokogiri** binary as the user **beloved**.

![14](/images/writeups/beloved/14.png)


So, I decide to copy and execute the command I saw in the ***bash_history***, and an Interactive **Ruby** (IRB) session opens.

![15](/images/writeups/beloved/15.png)
<h6>In an Interactive Ruby session</h6>

So, we can execute commands with the following structure:

```bash
system 'COMMAND'
```

So, I run a command to **spawn a shell** as **beloved**, and just like that, we become the **beloved** user.

```bash
system '/bin/bash'
```

![16](/images/writeups/beloved/16.png)

And we can obtain the user flag.

![17](/images/writeups/beloved/17.png)

## Privilege escalation to root

For the next privilege escalation, we will use [**pspy**](https://github.com/DominicBreuker/pspy), which is a program that monitors the processes occurring on the machine.

Now, let's transfer **pspy64** from our local machine to the victim machine by creating a web server with **Python** on port **8080**.

![18](/images/writeups/beloved/18.png)

We grant permissions with '**chmod +x pspy64**' and run it with '**./pspy64**'.

```bash
chmod +x pspy64
```

```bash
./pspy64
```

![19](/images/writeups/beloved/19.png)

We notice that a command is executed every minute.

![20](/images/writeups/beloved/20.png)


The **chown** command **changes the owner and group of files or directories**. In this case, it is applied to everything inside the ***/opt directory***. Inside ***/opt***, we found the **root** user's **id_rsa** file (a private **SSH key**) and since we have write permissions in this directory, it could be exploited.

![21](/images/writeups/beloved/21.png)

Now, we will exploit **chown** to change the owner of the **root**'s **id_rsa** to **beloved**. (Link where it explains how to exploit this vulnerability: [**Wildcard Injection**](/cybersecurity/vulnsandexp/linux/))

We can do it by executing the following commands in ***/opt***:

```bash
touch reference
```

```bash
touch -- --reference=reference
```

After waiting a minute, we will see that the **id_rsa** file becomes owned by us. In case the **id_rsa** is not in that directory but, for example, in /root/.ssh/id_rsa, we should create a symbolic link pointing to it like this:

```bash
ln -s /root/.ssh/id_rsa id_rsa
```

With this approach, we can obtain any file owned by **root** without any issues. But since we already have the **root**'s **id_rsa**, we don't need anything else.

![22](/images/writeups/beloved/22.png)
<h6>The root's id_rsa is now under our ownership</h6>

We copy the contents of the **id_rsa** file and save it to our local machine. Next, we escalate privileges by running:

```bash
chmod 600 id_rsa
```

And we log in via **SSH** using the **root** user and this **id_rsa**, like this:

```bash
ssh -i id_rsa root@IP
```

Finally, we obtain the root flag.

![23](/images/writeups/beloved/23.png)
