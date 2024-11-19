---
layout: post
read_time: true
show_date: true
title: "Intigriti-CTF-2024-write-up"
date: 2024-11-19
img: posts/20241119/Intigriti_CTF.png
tags: [CTF,Write up]
category: review
author: adawn0106
description: "Intigriti-CTF-2024-write-up"
comments: true

---




Last weekend, I participated in the Intigriti-CTF-2024, and it was really fun because there were so many problems of various types.
I’d like to take some time to review the challenges I solved and the ones I attempted during the competition.

The first category is the warmup. Since the scores were low, I thought I would solve them easily, but they turned out to be not that straightforward.


## Warm Up
<br>

### Sanity Check 

<br>

![Sanity Check](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/sanity1.png)

<br>
<br>

This challenge was a simple one. Clicking the link redirected to the competition's Discord server.
By checking the Discord channel, the flag could be found, making it an easy and straightforward challenge.

<br>
<br>

![Sanity Check](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/sanity2.png)


<br>

### Social

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/social1.png)

<br>

This challenge required checking the CTF's Twitter, YouTube, and Reddit to find parts of the flag, which needed to be combined to solve the challenge.

At first, I thought it would be easy, but in the case of Twitter, I couldn't access certain pages without logging in.
I wasted some time assuming the challenge was designed to be solvable without logging in.
In the end, logging in was necessary.
For Twitter, the required value was found in the mentions after logging in.

<br>

```
 0110100000110000011100000011001101011111011110010011000001110101
```

<br>

Next is YouTube.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/social2.png)

<br>


There was an issue where the required part could be found by sorting the comments by recent, but I managed to solve it this way.


Next is Reddit.

<br>


![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/social3.png)

<br>

It’s not just about combining the parts; you also need to apply the correct decoding method to solve it.

<br>

```
1.0110100000110000011100000011001101011111011110010011000001110101
2.5f336e6a30795f
3.ZDRfYzdm
```

<br>

If you convert the first part to ASCII,

<br>


![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/social4.png)


<br>

Second

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/social5.png)

<br>

Last

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/social6.png)

<br>

So the flag becomes INTIGRITI{h0p3_y0u_3nj0y_d4_c7f}.
































