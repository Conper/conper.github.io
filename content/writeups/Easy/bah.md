---
title: "Bah - HackMyVM"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Bah)


When starting the machine, it displays its **IP address**.

![1](/images/writeups/bah/1.png)
<h6>IP of the victim machine</h6>

Let's begin with a quick **NMAP** scan to identify the **open ports** on the target machine.

```bash
sudo nmap -sS -p- --min-rate 4500 -n -Pn 192.168.18.184
```

![2](/images/writeups/bah/2.png)

Now that we have identified the **open ports**, we will perform a more detailed scan to determine the **services** running on each port along with their respective **versions** and other relevant details.

```bash
nmap -sCV -p80,3306 -n -Pn 192.168.18.184 -oN target
```

![3](/images/writeups/bah/3.png)

Visiting the site hosted on **port 80** (**HTTP**) brings us to a **login page**.

![4](/images/writeups/bah/4.png)
<h6>Login screen</h6>

I have found that the **qdPM** version is **vulnerable** and the password is exposed.

![5](/images/writeups/bah/5.png)
<h6>I search for vulnerabilities in qdPM on searchsploit</h6>

The username and password can be found in this directory:

![6](/images/writeups/bah/6.png)
<h6>Path where the username and password are located</h6>

![7](/images/writeups/bah/7.png)
<h6>We obtain the file</h6>

We read the file and find that it is a **YAML configuration** for a database, containing the **username** and **password** needed to access it. **YAML files** like this are commonly **used to store configuration data** in a simple, readable format **with key-value pairs** and indentation.

![8](/images/writeups/bah/8.png)

Now we log in, but not on the website since we don't have an email, but through **port 3306** (**MySQL**) to access the database hosting the victim machine.

![9](/images/writeups/bah/9.png)
<h6>We are inside the database</h6>

I select the '**hidden**' database:

![10](/images/writeups/bah/10.png)

We find 2 tables: '**url**' and '**users**':

![11](/images/writeups/bah/11.png)

On one hand, in the '**users**' table, we find **IDs**, **usernames**, and **passwords**.

![12](/images/writeups/bah/12.png)

On the other hand, in the '**url**' table, we obtain a list of **URLs**.

![13](/images/writeups/bah/13.png)

We will save all the information obtained in a file.

To start, we will use [**WFUZZ**](https://github.com/xmendez/wfuzz) to find which **URLs** can serve as **subdomain names**.

![14](/images/writeups/bah/14.png)

```bash
wfuzz -c -w URLs.txt -u 192.168.18.184 -H "HOST: FUZZ"
```

![15](/images/writeups/bah/15.png)

We notice that '**party.bah.hmv**' is different from the rest. So, we add it to the ***/etc/hosts*** file to make our machine recognize that **subdomain** with the corresponding **IP**.

```bash
sudo nano /etc/hosts
```

![16](/images/writeups/bah/16.png)

And now we access.

![17](/images/writeups/bah/17.png)

It prompts us to log in. I decide to use the **username** and **password** we obtained earlier.

```
User: qpmadmin
Password: qpmpazzw
```

![18](/images/writeups/bah/18.png)

And we are in. Now, I am going to set up a **reverse shell** for a better interface, and also configure the **TTY** for improved control.

Let's enumerate the users on the victim machine:

```bash
grep /bin/bash /etc/passwd
```

![19](/images/writeups/bah/19.png)
<h6>Users in the machine</h6>

## Privilege escalation to rocio

We see a user named **rocio**, which is present in the table we downloaded earlier from the database.

![20](/images/writeups/bah/20.png)
<h6>Rocio password</h6>

We switch to the **rocio** user and obtain the **user flag**.

![21](/images/writeups/bah/21.png)

## Privilege escalation to root

To do this, we will use [**pspy**](https://github.com/DominicBreuker/pspy), which is a monitoring tool for Linux systems that helps detect important activities and processes discreetly.

Then, using **netcat**, we will transfer the program from our local machine to the victim machine.

-----
**Victim machine:**

```bash
nc -lvnp 8888 > pspy64
```
-----
**Local machine:**

```bash
nc 192.168.18.184 8888 < pspy64
```
-----

And we grant permissions to **pspy64**.

![22](/images/writeups/bah/22.png)


We run **pspy64** and observe the following.

![23](/images/writeups/bah/23.png)


It is a command line that executes **shellinaboxd**, a server that provides a browser-based web terminal. This service **allows command line access** through a web browser using the HTTP protocol. We observe that the **root** is running what is in ***/tmp/dev*** when someone accesses the ***/devel*** directory. Here in the **shellinaboxd** manual, we can see how it works:

![24](/images/writeups/bah/24.png)
<h6>Manual for shellinaboxd</h6>

So, I am going to create a file named **dev** in the ***/tmp*** directory where it sends a **Reverse Shell**, allowing quick access to the **root** user.

![25](/images/writeups/bah/25.png)
<h6>Contents of the 'dev' file</h6>

And we **grant permissions**:

```bash
chmod +x dev
```

First, we set up a listener on **port 4444** to catch any incoming connections, then we access the ***/devel*** directory.

![26](/images/writeups/bah/26.png)

And we receive the **Reverse Shell** without any issues.

![27](/images/writeups/bah/27.png)
<h6>We are root</h6>

We obtain the **root flag**, and there you go, we have successfully completed hacking the machine.