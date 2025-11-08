---
title: "Bingus"
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

[Crackme Link](https://crackmes.one/crackme/68a153668fac2855fe6fb67b)

## ⚙️ Characteristics

| Characteristic | Value        |
| -------------- | ------------ |
| **Language**   | C / C++      |
| **Arch**       | x86-64       |
| **Platform**   | Unix / Linux |


## 🗒️ Writeup

When trying to run the binary in different ways, it only returns the message "***Bingus exploded***".

![1](/images/crackmes/bingus/1.png)

What we'll do is open the binary in [**Ghidra**](https://github.com/NationalSecurityAgency/ghidra) to inspect the code and find the function that checks the input.

![2](/images/crackmes/bingus/2.png)

**We find the function** that decides whether it has been exploited or not. We’ll **analyze the function to understand the program’s logic** and figure out how to make it print "***Bingus survived***".

Before we start, we need to make it clear that `param_1` is the **number of arguments** and `param_2` is the **list of program arguments**.

### First condition:

![3](/images/crackmes/bingus/3.png)

The conditional is triggered if any of these conditions are true:

- There isn’t exactly one argument besides the program name.

- The two characters of the argument are not the same.

- The argument does not have exactly 2 characters.

In any of these cases, it prints "***Bingus exploded***".

Therefore, the **hint** to move forward is:

- Pass exactly 1 argument.

- The argument must be 2 identical characters.

> 📝 **NOTE**: `param_2` + 8 points to **argv[1]**, the first argument of the program, because in memory each pointer takes up **8 bytes** in **64-bit systems**.


### Loop:

For this part, I’ll **rename the variables** to make it **simpler** and **easier** to read.

![4](/images/crackmes/bingus/4.png)

What this loop does is **add each character** of the string "***This is a red herring***" to `num`, **summing their ASCII** values since in **C** a character is treated as an integer in arithmetic operations.

To see the result after the loop finishes, we recreate the code:

```
#include <stdio.h>
#include <string.h>

int main() {
    int num = 0x66;  /* num = 102 */
    int i;
    size_t x;
    const char *str = "This is a red herring";

    x = strlen(str);  /* x = 21 */

    for (i = 0; i < x; i++) {
        num = num + str[i];
    }

    printf("%d", num);
    return 0;
}
```

And we get `num = 2021`.

### Final condition:

![5](/images/crackmes/bingus/5.png)

This last condition **adds `num` and the ASCII values of the first two characters** of the parameter string. If the total is **2245** (**0x8c5**), it prints "***Bingus survived***".

<html>
<head>
  <script src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
</head>
<body>
  <p>
    $$
    \begin{align*}
    2021 + 2x &= 2245 \\
    2x &= 224 \\
    x &= 112
    \end{align*}
    $$
  </p>
</body>
</html>


Therefore, the resulting character in **ASCII** is **112**, which corresponds to the letter `p`, and since the program **expects two identical characters**, the correct result is `pp`.

![6](/images/crackmes/bingus/6.png)
