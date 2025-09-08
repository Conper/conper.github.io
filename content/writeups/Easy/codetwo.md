---
title: "CodeTwo - Hack The Box"
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

[Machine Link](https://app.hackthebox.com/machines/CodeTwo)

To begin working on the machine, we first perform a quick scan to identify open ports using **Nmap**:

```
sudo nmap -sS --min-rate 4500 -n -Pn -vvv 10.10.11.82
```

![1](/images/writeups/codetwo/1.png)

Once we know which ports are open, we perform a more detailed scan on them:

```
nmap -sCV -p22,8000 -n -Pn 10.10.11.82 -oN target
```

![2](/images/writeups/codetwo/2.png)

We can see that port **8000** is hosting a web server.


![3](/images/writeups/codetwo/3.png)
<h6>Home page</h6>

Clicking on the **Download App** button downloads a **ZIP file**.

![4](/images/writeups/codetwo/4.png)

This archive contains the **default structure of the web application**.

![5](/images/writeups/codetwo/5.png)

Here we can see the files it contains, and after checking the requirements, we notice that it uses a vulnerable version of **js2py**, which can be exploited through **RCE** (Remote Code Execution). Exploit: [**CVE-2024-28397**](https://github.com/waleed-hassan569/CVE-2024-28397-command-execution-poc)

![6](/images/writeups/codetwo/6.png)


![7](/images/writeups/codetwo/7.png)

To exploit this, we will run the provided **JavaScript** code inside an interpreter. Let’s register on the victim’s web application and see which programming language we can execute.

![8](/images/writeups/codetwo/8.png)

Luckily, we can execute **JavaScript** directly from the web interface.

![9](/images/writeups/codetwo/9.png)

It works perfectly, we can execute commands as the user **app**.
Next, we will establish a **Reverse Shell** to gain access inside the machine.

We start by setting up a listener with **Netcat** on port **4444**, and then execute the **Reverse Shell** payload in the exploit.

![10](/images/writeups/codetwo/10.png)

![11](/images/writeups/codetwo/11.png)

Now that we are inside, let’s remember that when we downloaded the **ZIP** earlier, it contained a file named ***users.db***. At that time, it was empty since it was just the default database. Knowing its location and purpose, let’s inspect it again.

![12](/images/writeups/codetwo/12.png)

![13](/images/writeups/codetwo/13.png)

We **transfer the file** to our local machine for easier handling.

![14](/images/writeups/codetwo/14.png)

![15](/images/writeups/codetwo/15.png)


This is where the **passwords** of all **users** registered in the web editor are stored, including ours.

To save time, we will first check which users exist on the system so we only attempt to **crack relevant passwords**.

![16](/images/writeups/codetwo/16.png)

Perfect, we see that **marco** exists on the machine.
We will use **Hashcat** to crack **Marco**’s password and see if it is reused for system login (**John The Ripper** didn’t work for me here).

![17](/images/writeups/codetwo/17.png)

We manage to log in without any issues, which allows us to retrieve the **User Flag**.

![19](/images/writeups/codetwo/19.png)

## Privilege escalation to Root

Running `sudo -l`, we discover that **Marco** can **run a backup binary as root** without needing the **root** password.

![20](/images/writeups/codetwo/20.png)

![21](/images/writeups/codetwo/21.png)

This binary requires a **configuration file** to work. Fortunately, we already have one in **Marco**’s home directory. Although it is owned by root, Marco has read permissions.

(With the `-h` option we can see all the **available parameters**.)

![22](/images/writeups/codetwo/22.png)

From this, we learn that the **root** user employs this configuration file to back up the web application we’ve been exploiting earlier.
To **escalate**, we can **create a copy** of the **configuration file** and modify it so that the backup targets the ***/root*** directory.

![23](/images/writeups/codetwo/23.png)

We then run the **backup** using:

```
sudo /usr/local/bin/npbackup-cli -c ./test.conf -b
```

![24](/images/writeups/codetwo/24.png)

Next, we **list all snapshots** to identify the one we need:

```
sudo /usr/local/bin/npbackup-cli -c ./test.conf --snapshots

```

![25](/images/writeups/codetwo/25.png)

Once we know the **snapshot ID**, we can **list its contents**:

```
sudo /usr/local/bin/npbackup-cli -c ./test.conf --snapshot-id ID --ls
```

![26](/images/writeups/codetwo/26.png)

Finally, we obtain the **Root Flag** and **complete the machine**.

```
sudo /usr/local/bin/npbackup-cli -c ./npbackup.conf --snapshot-id 0121d0a3 --dump /root/root.txt
```

