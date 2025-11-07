---
title: "Gift - HackMyVM"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Gift)

We can see that when we open the machine it tells us its IP:

![01](/images/writeups/gift/1.png)

So knowing that the IP of the machine is 192.168.18.28, we are going to do a simple NMAP scan to see what ports it has open and what service they are running:

![02](/images/writeups/gift/2.png)
<h6>NMAP simple scan</h6>


We can see that this machine hosts a web server on port 80 and that port 22 (ssh) is open.

This is what we see when we open the web server:

![03](/images/writeups/gift/3.png)

It tells us not to think too much. So we decided to try to enter through port 22 by brute force with the hydra tool.

We are going to set the user as root and as the password we will set a dictionary (in this case common.txt) so that it goes word by word trying to see if any word in this dictionary is the correct password:

> 🚨 **IMPORTANT:** Normally, you cannot access as the root user via SSH for security reasons. Also, the rockyou.txt file is usually used for password cracking. This is an exception because it’s a very easy difficulty machine.


![04](/images/writeups/gift/4.png)
<h6>Result after using hydra</h6>

And we found it! The user is "root" and the password is "simple". Now we are going to enter through port 22 with this username and password.

![05](/images/writeups/gift/5.png)

Now that we are inside, with the "ls" command we see what there is, and we find that we already have the user flag and the root flag. 

![06](/images/writeups/gift/6.png)

So machine completed!