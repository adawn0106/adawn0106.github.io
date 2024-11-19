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

---

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

---

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

<br>

---


### Lost Program

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/program1.png)

<br>

TODO: find lots of 😎🐛 on 🥷🥝🎮

Using these hints, I figured out what they meant after some thought. The first emoji represented bug bounty, and the second referred to Ninja Kiwi. 
With that understanding, I constructed the flag accordingly.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/program2.png)

<br>

The correct flag is INTIGRITI{Ninja_Kiwi_Games}.

<br>


---

### IN Plain Sight

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/sight1.png)

<br>

This challenge was one I checked after the competition ended.
When downloading the image, you could see a picture of a cat.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/sight2.png)

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/sight3.png)

<br>

Looking at the hex values, I noticed that the file contained a flag.png and identified the "PK" file signature, indicating that another file might be hidden inside.
To investigate further, I used binwalk to analyze the file.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/sight4.png)

<br>

By running binwalk on the file, you can see the PK file signature, indicating the presence of an embedded file.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/sight5.png)

<br>

A zip file is visible in the results, but the flag.png file itself is not directly shown.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/sight6.png)

<br>

The zip file was password-protected, making it inaccessible.
I couldn't solve it before the competition ended, but according to the write-up, here's what happened:

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/sight7.png)

<br>

In HxD, upon closer inspection, you can find a specific phrase embedded in the file. 
Entering that value as the password allows you to extract the contents of the zip file.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/sight8.png)

<br>

However, when you open the flag.png file, you’ll see that it’s just a blank white image.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/sight9.png)

<br>

If you paint the background black in a program like Paint, the hidden flag becomes visible on the image.

<br>

---

### IrrORversible

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/xor1.png)

<br>

When I first approached this challenge, I thought it might require a brute-force method. 
However, since there were no hints provided, I doubted that brute-forcing was the intended solution and suspected it might involve XOR operations instead.
I attempted this problem but couldn’t solve it within the time limit.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/xor2.png)

<br>

When you run the program, it outputs encrypted values based on the plaintext input you provide.
This challenge involves inputting specific plaintext values and observing their corresponding encrypted outputs.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/xor3.png)

<br>

This challenge requires understanding XOR operations. With XOR, the following properties hold:

<br>

```
plaintext ^ key = encrypt
plaintext ^ encrypt = key
```

<br>

Therefore, by inputting a known plaintext and obtaining its encrypted output, you can XOR the plaintext with the encrypted result to determine the key.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/xor4.png)

<br>

The flag is INTIGRITI{b451c_x0r_wh47?}.

---

### Layers

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/l1.png)

<br>

When you extract the compressed file, you can see that it contains numerous files.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/l2.png)

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/l3.png)

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/l4.png)

<br>

The combined ASCII values were entered into CyberChef for processing.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/l5.png)

<br>

I couldn't find any special values when combining the ASCII values.
When I opened each file individually, I noticed that ASCII characters were present in them. 
Initially, I tried combining the ASCII values in numerical order of the files, but the output didn't make sense. 
I continued working on the challenge but couldn’t solve it within the time limit.
According to the write-up, the solution is as follows:

<br>

The ls -lart command plays a key role. I searched for the explanation of ls -lart using GPT, and here's what it means:

<br>

Explanation of ls -lart
The ls -lart command lists files in a directory with detailed information and sorts them based on specific criteria. The options serve the following purposes:

l (long listing format):

Displays detailed file information, such as:
File permissions (e.g., rw-r--r--)
Owner and group (e.g., crystal crystal)
File size (e.g., 8 bytes)
Last modification time (e.g., Aug 19 17:15)
File name (e.g., 52)
a (all):

Includes hidden files (those starting with .) in the listing.
For example, it will show . (current directory) and .. (parent directory).
r (reverse order):

Reverses the default sorting order.
By default, ls -l sorts in ascending order by name or time. Adding r changes this to descending order.
t (time):

Sorts files by their modification time.
The most recently modified files appear first.
Combined Effect of ls -lart
Time-based Sorting: The t option sorts files by modification time.
Reversed Order: The r option changes the order from descending (default) to ascending.
Includes Hidden Files: The a option ensures that hidden files are listed as well.
Detailed Output: The l option provides additional details about each file, such as permissions, ownership, and size.
When applied to the challenge files, the result would look something like this:

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/l6.png)

<br>

The files are sorted by their creation times, and by converting the binary values from these files in this order, you can obtain the flag.
According to the official write-up, the solution was achieved using the following source code.

<br>

```
import zipfile
import os
from datetime import datetime

ARCHIVE_NAME = "layers.zip"
EXTRACT_DIR = "files"

def binary_to_char(binary_str):
    return chr(int(binary_str, 2))

with zipfile.ZipFile(ARCHIVE_NAME, 'r') as zipf:
    for info in zipf.infolist():
        extracted_path = zipf.extract(info, EXTRACT_DIR)
        date_time = datetime(*info.date_time)
        mod_time = date_time.timestamp()
        os.utime(extracted_path, (mod_time, mod_time))

file_data = []
for file_name in os.listdir(EXTRACT_DIR):
    file_path = os.path.join(EXTRACT_DIR, file_name)
    mod_time = os.path.getmtime(file_path)
    with open(file_path, "r") as f:
        binary_data = f.read().strip()
        char = binary_to_char(binary_data)
    file_data.append((mod_time, char))

file_data.sort()
reconstructed_string = ''.join([char for _, char in file_data])

print("Reconstructed String:")
print(reconstructed_string)

for file_name in os.listdir(EXTRACT_DIR):
    os.remove(os.path.join(EXTRACT_DIR, file_name))
os.rmdir(EXTRACT_DIR)
```

<br>

By using it, you can obtain the corresponding string.

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/l7.png)

<br>

![social](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/l8.png)

<br>

Then the flag becomes INTIGRITI{7h3r35_l4y3r5_70_7h15_ch4ll3n63}.

<br>

---

### BabyFlow

<br>

![BabyFlow](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/b1.png)

<br>

I ran the file.

<br>

![BabyFlow](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/b2.png)

<br>

![BabyFlow](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/b3.png)

<br>

The program said the password was incorrect, so I opened it in Ghidra to analyze it.

<br>

![BabyFlow](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/b4.png)

<br>

The structure of the function is as follows:
When the input value is SuPeRsEcUrEPaSsWoRd123, it prints correct. However, if local_C == 0, it prints:
Are you sure you are admin? o.O.
To get the flag, local_C needs to hold a value other than 0.
To achieve this, local_C must be influenced to take a different value.
Analyzing the value-checking part, it uses strncmp, which only compares the specified number of characters.
It does not check the remaining characters beyond the specified limit.
This allows reading additional values that follow the compared string.

<br>

![BabyFlow](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/b5.png)

<br>

![BabyFlow](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/b6.png)

<br>

This is the part where local_C is declared. Its location is as follows:

<br>

![BabyFlow](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/b7.png)

<br>

is 0 

<br>

![BabyFlow](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/b8.png)

<br>

After entering the input, checking the value reveals the following:

<br>

![BabyFlow](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/b9.png)

<br>

![BabyFlow](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/b10.png)

<br>

Since the value becomes aaa, which is not 0, the program prints the flag.
I solved it using this payload.

<br>

![BabyFlow](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/b11.png)

<br>

---

### Rigged Slot Machine 1

<br>

![Rigged Slot Machine 1](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/j1.png)

<br>

This challenge seemed strange because, according to the write-up, the provided code was very similar to the one I wrote while solving it.
However, I couldn't solve it, and even using the code provided in the write-up, the flag wasn't revealed.
Starting with the source code, here's what I found:

<br>

```
void main(void)

{
  time_t tVar1;
  long in_FS_OFFSET;
  uint local_20;
  int local_1c;
  __gid_t local_18;
  int local_14;
  undefined8 local_10;
  
  local_10 = *(undefined8 *)(in_FS_OFFSET + 0x28);
  setvbuf(stdout,(char *)0x0,2,0);
  local_18 = getegid();
  setresgid(local_18,local_18,local_18);
  tVar1 = time((time_t *)0x0);
  srand((uint)tVar1);
  setup_alarm(0xb4);
  local_20 = 100;
  puts("Welcome to the Rigged Slot Machine!");
  puts("You start with $100. Can you beat the odds?");
  do {
    while( true ) {
      while( true ) {
        local_1c = 0;
        printf("\nEnter your bet amount (up to $%d per spin): ",100);
        local_14 = __isoc99_scanf(&DAT_0010222e,&local_1c);
        if (local_14 == 1) break;
        puts("Invalid input! Please enter a numeric value.");
        clear_input();
      }
      if ((local_1c < 1) || (100 < local_1c)) break;
      if ((int)local_20 < local_1c) {
        printf("You cannot bet more than your current balance of $%d!\n",(ulong)local_20);
      }
      else {
        play(local_1c,&local_20);
        if (0x20a6e < (int)local_20) {
          payout(&local_20);
        }
      }
    }
    printf("Invalid bet amount! Please bet an amount between $1 and $%d.\n",100);
  } while( true );
}
```
<br>

You start with $100 and can bet up to $100 based on the user's input.
If your total money exceeds 133742, the payout function is executed.

<br>

```
void play(int param_1,uint *param_2)
{
  uint uVar1;
  long lVar2;
  int iVar3;
  long in_FS_OFFSET;
  int local_1c;
  
  lVar2 = *(long *)(in_FS_OFFSET + 0x28);
  iVar3 = rand();
  iVar3 = iVar3 % 100;
  if (iVar3 == 0) {
    local_1c = 100;
  }
  else if (iVar3 < 10) {
    local_1c = 5;
  }
  else if (iVar3 < 0xf) {
    local_1c = 3;
  }
  else if (iVar3 < 0x14) {
    local_1c = 2;
  }
  else if (iVar3 < 0x1e) {
    local_1c = 1;
  }
  else {
    local_1c = 0;
  }
  uVar1 = param_1 * local_1c - param_1;
  if ((int)uVar1 < 1) {
    if ((int)uVar1 < 0) {
      printf("You lost $%d.\n",(ulong)-uVar1);
    }
    else {
      puts("No win, no loss this time.");
    }
  }
  else {
    printf("You won $%d!\n",(ulong)uVar1);
  }
  *param_2 = *param_2 + uVar1;
  printf("Current Balance: $%d\n",(ulong)*param_2);
  if ((int)*param_2 < 1) {
    puts("You\'re out of money! Game over!");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  if (lVar2 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return;
}
```
<br>

With a certain probability, you can either win more money, lose money, or break even.

<br>

```
void payout(int *param_1)

{
  FILE *__stream;
  long in_FS_OFFSET;
  char local_58 [72];
  undefined8 local_10;
  
  local_10 = *(undefined8 *)(in_FS_OFFSET + 0x28);
  if (*param_1 < 0x20a6f) {
    puts("You can\'t withdraw money until you win the jackpot!");
                    /* WARNING: Subroutine does not return */
    exit(-1);
  }
  __stream = fopen("flag.txt","r");
  if (__stream == (FILE *)0x0) {
    puts(
        "Flag File is Missing. Problem is Misconfigured, please contact an Admin if you are running  this on the shell server."
        );
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  fgets(local_58,0x40,__stream);
  printf("Congratulations! You\'ve won the jackpot! Here is your flag: %s\n",local_58);
  fclose(__stream);
                    /* WARNING: Subroutine does not return */
  exit(-1);
}
```
<br>

When the payout function is executed, it seems to read and print the contents of flag.txt.
Locally, the process runs very quickly, but when running it remotely, it seems to take over 3 minutes, preventing successful completion.

<br>

![Rigged Slot Machine 1](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/j2.png)

<br>

So, looking at the flags of those who solved it...

<br>

![Rigged Slot Machine 1](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/j3.png)

<br>

It seems to be displayed like this.

<br>

---

## REV

<br>

---

### Secure Bank

<br>

![Secure Bank](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/s1.png)

<br>

![Secure Bank](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/s2.png)

<br>

When running the challenge file, it prompts for a PIN.
I opened it using Ghidra for analysis.

<br>

```
bool main(void)

{
  undefined4 local_14;
  int local_10;
  undefined4 local_c;
  
  banner();
  login_message();
  printf("Enter superadmin PIN: ");
  __isoc99_scanf(&DAT_001021ea,&local_10);
  if (local_10 == 1337) {
    local_c = generate_2fa_code(1337);
    printf("Enter your 2FA code: ");
    __isoc99_scanf(&DAT_001021ea,&local_14);
    validate_2fa_code(local_14,local_c);
  }
  else {
    puts("Access Denied! Incorrect PIN.");
  }
  return local_10 != 1337;
}
```
<br>

I found that the first PIN can be successfully entered as 1337, and then a second prompt for a 2fa_code appears.

<br>

```
uint generate_2fa_code(int param_1)

{
  int local_14;
  uint local_10;
  uint local_c;
  
  local_10 = param_1 * 0xbeef;
  local_c = local_10;
  for (local_14 = 0; local_14 < 10; local_14 = local_14 + 1) {
    local_c = obscure_key(local_c);
    local_10 = ((local_10 ^ local_c) << 5 | (local_10 ^ local_c) >> 0x1b) +
               (local_c << ((char)local_14 + (char)(local_14 / 7) * -7 & 0x1fU) ^
               local_c >> ((char)local_14 + (char)(local_14 / 5) * -5 & 0x1fU));
  }
  return local_10 & 0xffffff;
}
```
<br>

```
uint obscure_key(uint param_1)

{
  return ((param_1 ^ 0xa5a5a5a5) << 3 | (param_1 ^ 0xa5a5a5a5) >> 0x1d) * 0x1337 ^ 0x5a5a5a5a;
}
```
<br>

When the 2fa_code is entered, the program processes the input and determines whether it is

<br>

```
void validate_2fa_code(int param_1,int param_2)

{
  if (param_1 == param_2) {
    puts("Access Granted! Welcome, Superadmin!");
    printf("Here is your flag: %s\n","INTIGRITI{fake_flag}");
  }
  else {
    puts("Access Denied! Incorrect 2FA code.");
  }
  return;
}
```
<br>

The program outputs the flag when the correct 2fa_code is entered.
There are two ways to solve this challenge, but since I originally solved it using static analysis, I decided to try the dynamic approach mentioned in the write-up.
Here’s what I observed in this straightforward challenge:

<br>

![Secure Bank](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/s3.png)

<br>

After setting a breakpoint at the section following the FA function call, examining the value shows that this specific value is stored:

<br>

```
0x568720 → 5670688
```

<br>


The reason for converting it to decimal is...

<br>

![Secure Bank](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/s4.png)

<br>

...because the program uses %u to receive the value, and later, the generated FA value will be compared as a decimal.

<br>

![Secure Bank](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/s5.png)

<br>

---

### TriForce Recon

<br>

![TriForce Recon](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/t1.png)

<br>

When you extract the compressed file,

<br>

![TriForce Recon](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/t2.png)

<br>

You can see that it consists of three files.
Each file has a different format:

One is an .exe file.
One is an .elf file.
One is a Mach-O file.

<br>

![TriForce Recon](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/t3.png)

<br>

When I executed the ELF file, it seemed to require an input value.
According to the write-up, the suggested approach appeared simpler, so I’ll introduce that method.
(Initially, I had used static analysis to write code and figure out the value myself.)
Looking at the code in IDA, it’s clear that a simple XOR operation can be performed to solve it.

<br>

![TriForce Recon](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/t4.png)

<br>

You can identify the string to XOR and the key value.

<br>

![TriForce Recon](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/t5.png)

<br>

![TriForce Recon](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/t6.png)

<br>

Therefore, it’s a straightforward challenge where you simply input the XOR key and the string to solve it.

<br>

![TriForce Recon](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/t7.png)

<br>

This challenge involved combining the flags from three programs with different formats.

<br>

The final flag is:
Intigriti{s7reaMCircUm574NcEgrAV3UnpL3A54ntRE5i6N4T10nSatI5fAct1on}.

<br>

---

<br>

## OSINT

<br>

---

<br>

### TrackDown1,2

<br>

![TrackDown1,2](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/tr1.png)

<br>

![TrackDown1,2](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/tr2.png)

<br>

This challenge involves identifying the location where the given photo was taken.

<br>

![TrackDown1,2](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/tr3.png)

<br>

The above photo is from the first challenge, and if you search for it on Google, you will find...

<br>

![TrackDown1,2](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/tr4.png)

<br>

By searching for the building from the photo on Google, I was able to identify it.
Using Google Street View and examining the photo, I noticed the table in the image had a restaurant-like feel, so I tried the name of a nearby bar or building.
This resulted in the flag being validated:

INTIGRITI{Si_Lounge_Hanoi}.

<br>

![TrackDown1,2](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/tr5.png)

<br>

This is the second photo, and if you search for it using Google Lens, you will find...

<br>

![TrackDown1,2](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/tr6.png)

<br>

The search results showed a photo that appears to be taken at night, likely from the same location as the previous one. Based on this, I identified the hotel and the flag was successfully validated:

INTIGRITI{Express_by_M_Village}.

<br>

---

<br>

### NO Comment

<br>

![NO Comment](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/n1.png)

<br>

This was another challenge I couldn't solve within the time limit. However, when you download the photo, you can see...

<br>

![NO Comment](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/n2.png)

<br>

The image looks like this, and since it's an OSINT challenge, I initially thought it was similar to a trackdown problem, so I wasted a lot of time searching on Google.
I'll follow the write-up to proceed.
By using an image metadata tool like exiftool, I was able to confirm the following details:

<br>

![NO Comment](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/n3.png)

<br>

By checking the comment, I found a link. Upon reviewing the write-up, it seems that the format is related to imgur.com.

<br>

![NO Comment](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/n4.png)

<br>

When checking the location, I found the following encoded string:

<br>

```
V2hhdCBhICJsb25nX3N0cmFuZ2VfdHJpcCIgaXQncyBiZWVuIQoKaHR0cHM6Ly9wYXN0ZWJpbi5jb20vRmRjTFRxWWc=
```

<br>

I input this value into CyberChef, and...

<br>

![NO Comment](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/n5.png)

<br>

in link

<br>

![NO Comment](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/n6.png)

<br>

The site appears, and it prompts for a password.
The password, which we decoded earlier as long_strange_trip, should be entered.
Once entered, the following string can be obtained:

<br>

![NO Comment](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/n7.png)

<br>

```
25213a2e18213d2628150e0b2c00130e020d024004301e5b00040b0b4a1c430a302304052304094309
```

<br>

After entering the password, it seems that you need to access the profile to gather further information.

<br>

![NO Comment](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/n8.png)

<br>

![NO Comment](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/n9.png)

<br>

You can then infer from the profile that the encrypted data was XORed.

<br>

![NO Comment](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/n10.png)

<br>

INTIGRITI{instagram.com/reel/C7xYShjMcV0}

<br>

---

<br>

### Bob L'éponge

<br>

![Bob L'éponge](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/bob1.png)

<br>

I couldn't solve this challenge within the time limit either.
When you enter the link, the following video appears.

<br>

![Bob L'éponge](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/bob2.png)

<br>

There is this strange video, and I'm not sure where or how to use it...

<br>

![Bob L'éponge](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/bob3.png)

<br>

When checking the profile's playlist, I saw this list, but when I played it, there was nothing useful, so I couldn't solve it.
According to the write-up, it seems there is a tool for reading YouTube data.
By using that tool to extract data from the second video, the flag appeared, which felt anticlimactic.

<br>

![Bob L'éponge](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/bob4.png)

<br>

INTIGRITI{t4gs_4r3_m0stly_0bs0l3t3_zMlH7RH6psw}

<br>

---

<br>

### Private Github Repository

<br>

![Private Github Repository](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/git1.png)

<br>

I almost solved it, but due to using the wrong approach, I couldn’t complete the challenge.
First, if you search for the user on GitHub, you will find...

<br>

![Private Github Repository](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/git2.png)

<br>

You can find one user.

<br>

![Private Github Repository](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/git3.png)

<br>

You can find an email in this format. Initially, I thought it might be the key value, but upon further investigation, I realized it was actually a PK (zip file).

<br>

![Private Github Repository](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/git4.png)

<br>

When you open the zip file, you can find the private key.

<br>

![Private Github Repository](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/git5.png)

<br>

So, I added the SSH key and...


<br>

```
git clone git@github.com:bob-193/1337up.git

```

<br>

Through that, I was able to download the repository.

<br>

![Private Github Repository](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/git6.png)

<br>


I got stuck at this part during the CTF competition.
Then, upon checking the write-up, it mentioned using the following command:

<br>

```
ssh -T git@github.com
```

<br>

When I asked GPT about the command, it responded as follows:

<br>

Execution Result
If the SSH key is correctly registered:

<br>

```
Hi <GitHub-username>! You've successfully authenticated, but GitHub does not provide shell access.
```

<br>

Here, <GitHub-username> refers to the GitHub account name.
This message indicates that the SSH key authentication was successful.

<br>

![Private Github Repository](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/git7.png)

<br>

Through this, I was able to discover Tiffany's account name.
Then, I proceeded to fetch the repository along with Bob.

<br>

![Private Github Repository](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/git8.png)

<br>

After that, I tried to clone the repository, but...

<br>

![Private Github Repository](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/git9.png)

<br>

It didn’t proceed as expected, possibly due to the download, but in any case, the goal was to search for the flag among the git logs.

<br>

---

<br>

## Misc

---

<br>

### Quick Recovery

<br>

![Quick Recovery](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/qr1.png)

<br>

When you open the file, you will find...

<br>

![Quick Recovery](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/qr2.png)

<br>

![Quick Recovery](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/qr3.png)

<br>

Looking at the image, you can see a fragmented QR code. It seems that the goal is to reconstruct the QR code.

<br>

```
from PIL import Image, ImageDraw
from itertools import permutations
import subprocess

qr_code_image = Image.open("qr_code.png")
width, height = qr_code_image.size
half_width, half_height = width // 2, height // 2

squares = {
    "1": (0, 0, half_width, half_height),
    "2": (half_width, 0, width, half_height),
    "3": (0, half_height, half_width, height),
    "4": (half_width, half_height, width, height)
}


def split_square_into_triangles(img, box):
    x0, y0, x1, y1 = box
    a_triangle_points = [(x0, y0), (x1, y0), (x0, y1)]
    b_triangle_points = [(x1, y1), (x1, y0), (x0, y1)]

    def crop_triangle(points):
        mask = Image.new("L", img.size, 0)
        draw = ImageDraw.Draw(mask)
        draw.polygon(points, fill=255)
        triangle_img = Image.new("RGBA", img.size)
        triangle_img.paste(img, (0, 0), mask)
        return triangle_img.crop((x0, y0, x1, y1))

    return crop_triangle(a_triangle_points), crop_triangle(b_triangle_points)


triangle_images = {}
for key, box in squares.items():
    triangle_images[f"{key}a"], triangle_images[f"{key}b"] = split_square_into_triangles(
        qr_code_image, box)

a_order = ["1", "2", "3", "4"]  # UPDATE ME
b_order = ["1", "2", "3", "4"]  # UPDATE ME

final_positions = [
    (0, 0),
    (half_width, 0),
    (0, half_height),
    (half_width, half_height)
]

reconstructed_image = Image.new("RGBA", qr_code_image.size)

for i in range(4):
    a_triangle = triangle_images[f"{a_order[i]}a"]
    b_triangle = triangle_images[f"{b_order[i]}b"]
    combined_square = Image.new("RGBA", (half_width, half_height))
    combined_square.paste(a_triangle, (0, 0))
    combined_square.paste(b_triangle, (0, 0), b_triangle)
    reconstructed_image.paste(combined_square, final_positions[i])

reconstructed_image.save("obscured.png")
print("Reconstructed QR code saved as 'obscured.png'")
```

<br>

The content of the source code is as follows:
The solution code is as follows:
(Provide the source code or solution code here.)

<br>

```
from PIL import Image, ImageDraw
from itertools import permutations

qr_code_image = Image.open("obscured.png")
width, height = qr_code_image.size
half_width, half_height = width // 2, height // 2

squares = {
    "1": (0, 0, half_width, half_height),
    "2": (half_width, 0, width, half_height),
    "3": (0, half_height, half_width, height),
    "4": (half_width, half_height, width, height)
}

def split_square_into_triangles(img, box):
    x0, y0, x1, y1 = box
    a_triangle_points = [(x0, y0), (x1, y0), (x0, y1)]
    b_triangle_points = [(x1, y1), (x1, y0), (x0, y1)]

    def crop_triangle(points):
        mask = Image.new("L", img.size, 0)
        draw = ImageDraw.Draw(mask)
        draw.polygon(points, fill=255)
        triangle_img = Image.new("RGBA", img.size)
        triangle_img.paste(img, (0, 0), mask)
        return triangle_img.crop((x0, y0, x1, y1))

    return crop_triangle(a_triangle_points), crop_triangle(b_triangle_points)

triangle_images = {}
for key, box in squares.items():
    triangle_images[f"{key}a"], triangle_images[f"{key}b"] = split_square_into_triangles(qr_code_image, box)

# 모든 순열을 시도
for a_order in permutations(["1", "2", "3", "4"]):
    b_order = a_order[::-1]  # a_order의 역순으로 b_order 설정
    
    final_positions = [
        (0, 0),
        (half_width, 0),
        (0, half_height),
        (half_width, half_height)
    ]
    
    reconstructed_image = Image.new("RGBA", qr_code_image.size)
    
    for i in range(4):
        a_triangle = triangle_images[f"{a_order[i]}a"]
        b_triangle = triangle_images[f"{b_order[i]}b"]
        combined_square = Image.new("RGBA", (half_width, half_height))
        combined_square.paste(a_triangle, (0, 0))
        combined_square.paste(b_triangle, (0, 0), b_triangle)
        reconstructed_image.paste(combined_square, final_positions[i])
    
    # 결과 이미지 저장
    filename = f"reconstructed_{''.join(a_order)}.png"
    reconstructed_image.save(filename)
    print(f"Reconstructed QR code saved as '{filename}'")
```

<br>


Since it tries all permutations, all possible versions of the image are generated.
By running the source code, you can see various images, and from those, the correct one was selected.

<br>

![Quick Recovery](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/qr4.png)

<br>

![Quick Recovery](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/qr5.png)

<br>

---

<br>

Here's the end of the write-up, and there's a small episode I want to share. While solving a problem, a hint popped up mentioning "v8 version," which made me think it was related to a v8 issue. I thought I’d check the write-up later. After the competition ended, I checked and found that one of the authors of the problem was a teammate from the CVE-2024-0517 analysis I worked on this summer. His blog was referenced in the write-up, which was really surprising. It also made me realize how important it is to continue working on v8 analysis. I’m still doing v8 analysis these days, but since it doesn’t always yield immediate results like CTFs, I plan to post my findings once the analysis is fully completed. Looking forward to more challenges ahead!
I should make sure to be mentioned next time too 🤭🤭

<br>

![episode](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/v1.png)

<br>

![episode](https://github.com/Adawn0106/Adawn0106.github.io/raw/main/assets/img/posts/20241119/v3.png)

<br>



