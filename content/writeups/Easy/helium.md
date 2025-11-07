---
title: "Helium - HackMyVM"
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

[Machine Link](https://hackmyvm.eu/machines/machine.php?vm=Helium)

To start, we perform a basic NMAP scan to find out which ports the target machine has open.

```bash
nmap 192.168.18.54
```

![1](/images/writeups/helium/1.png)
<h6>Simple NMAP scan</h6>



As we can observe, we find ports 80 and 22 open. Port 80 hosts a webpage, and port 22 runs the SSH service.

We open the website and encounter the following:

![2](/images/writeups/helium/2.png)

![3](/images/writeups/helium/3.png)

<h6>We view this with Ctrl + U</h6>

We obtain a possible username: **paul**.

We enter the file ***bootstrap.min.css*** and find the following information that may be useful to us.

![4](/images/writeups/helium/4.png)

We test the address /yay/mysecretsound.wav, and indeed, it exists. We download the audio.

Upon listening to the audio, you realize it might be in Morse code. So, you go to the [**morsecode.world**](https://morsecode.world/) website to upload the audio and have it decoded.

![5](/images/writeups/helium/5.png)
<h6>Result</h6>

I attempted to log in via SSH with the username **paul** and the password **ETAIE4SIET**, but it didn't work. I noticed a word formed in the audio at the bottom, so I used "dancingpassyd" as the password and successfully logged in.

![6](/images/writeups/helium/6.png)

Now, with the **ls** command, we see that we have the **user flag**.

## Privilege escalation

We use the **sudo -l** command to view the actions we can perform with sudo as root while being the user **paul**. As a result, we see that we can execute the ***/usr/bin/ln*** binary with sudo as if we were root without needing to know their password.

We go to the [**GTFOBins**](https://gtfobins.github.io/) website and look for information on how to escalate privileges using the **sudo ln** command.

![7](/images/writeups/helium/7.png)
<h6>Search result on GTFOBins</h6>

We copy the commands and execute them:

```bash
sudo ln -fs /bin/sh /bin/ln
```

```bash
sudo ln
```

![8](/images/writeups/helium/8.png)


We have successfully become the root and we can obtain their **flag**.

```bash
cat /root/root.txt
```