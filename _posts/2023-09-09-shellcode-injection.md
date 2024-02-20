---
layout: post
title: Loading encrypted shellcode at runtime - A CTF writeup
subheading:  Analyzing dynamic shellcode injection to avoid static analysis
categories: [Projects, Reverse Engineering]
tags: [Reverse Engineering]
---

Recently I came across a rather interesting problem during the [ASD CTF](https://www.anu.edu.au/capture-the-flag-2023), 
which was jointly hosted by ANU and ASD (Australian Signals Directorate). It was a reveres engineering 
challenge but what caught my eye was that it utilizes a somewhat sophisticated technique that many malware and
anti-cheats/AVs use to dissuade static analysis during reverse engineering, namely, streaming (encrypted) shellcode
directly into memory.

## A brief primer on how programs thwart reversing attempts

For the most part, we can define a program to be an executable encoded in binary form. Inside the executable is 
a section of code (amongst other things) that gets run once the program is loaded into memory. 

Reverse engineers takes advantage of this fact and attempts to decompile code segments within
a program in order to figure out how it works. The task of reverse engineering can be broadly classified into two forms:
static and dynamic analysis. As the name implies, static analysis is the act of analyzing the raw bytes of a given program's
file on disk, whereas dynamic analysis refers to the act of analyzing a program whilst it is being
run.

There are numerous ways developers employ in order to dissuade their programs of being statically analyzed. Obfuscation 
is a technique that involves masking the source code and adding "junk code" in order to make the code seem convoluted,
thereby making it harder to reverse engineer. Another technique involves virtualizing the code itself using something like VMProtect,
converting the code into a proprietary form, making it hard for people to decompile it into pseudocode.

## What is shellcode injection and how does it work?

Not a lot of people realize but you can actually allocate executable memory. Much like how malloc allocates a section of
memory for data, Windows' `VirtualAlloc` and Linux's `mmap` is able to allocate
a section of executable memory, that is, you can run code which resides in the section of allocated memory.

Shellcode injection relies on this fact. Raw, compiled code (shellcode) is copied into a section of 
executable memory (injection) and can be either voluntarily or forcibly executed.

## How does shellcode injection prevent static analysis

With shellcode injection, program logic does not necessarily have to exist at runtime. It is entirely possible
to store large parts of the program logic remotely and simply load them in when necessary. The other added benefit is that
code does not necessarily have to be computer readable. Parts of the program can be encrypted, and when needed can be dynamically
decrypted and loaded into memory.

Sophisticated malware and anti-cheats/AVs use a combination of both techniques to protect their code. Typically, large portions
of the program logic is encrypted and stored on a remote server, and streamed directly into memory where it is decrypted and ran.
This makes it impossible for anyone to analyze the distributed executable (without running them).

## CTF Walkthrough

When you first decompile the main function, it'll look something like this (note that most of the functions were unamed 
initially and comments were added on later):

![main](https://raw.githubusercontent.com/thebowenfeng/asdctf-2023-files/master/1.PNG)

The first thing that caught my eye, which also hinted towards its mode of operation, is "// flag function" line. This means
that `addr` is in fact a function pointer and is likely not a static function. The second thing is the `evp_decrypt` function,
which IDA helpfully annotated for me. This implies that there is some encryption involved within this program.

Since the return value of `addr` (`flag`) is instrumental to whether or not the CTF flag is shown, I know the key to overcoming
this challenge relies on whatever function `addr` is pointing to.

I was interested to know where the `addr` is coming from, so I looked up and noticed `add_function_to_memory` (again unamed at first).
The function returns an address, which then turns out to be a function pointer, so the function must have something to do with
loading in said function. A closer look at `add_function_to_memory` shows this:

![loading in the function](https://raw.githubusercontent.com/thebowenfeng/asdctf-2023-files/master/4.PNG)

Its now pretty obvious what the function does. It takes the buffer `plaintext` from its argument and copies its content 
to a executable section of memory, which it also kindly allocates, then returning a pointer to said section, which is then
later executed.

Following this trail, I looked up the `plaintext` buffer and saw that it is passed in to evp_decrypt, which leads me to believe
that `ciphertext` is in fact a section of encrypted shellcode. A closer look led me to `read_ciphertext` function shows that
the encrypted shellcode is actually embedded within the executable image itself.

![read_ciphertext](https://raw.githubusercontent.com/thebowenfeng/asdctf-2023-files/master/3.PNG)

Now that I have access to the encrypted shellcode, I am faced with two options: Either I can decrypt it myself, or I can 
intercept the decrypted `plaintext` at runtime. Option 1 ended up being a bit finnicky as I did not have the right Linux distro
which contained the same version as OpenSSL that is used to compile this program so I ended up going with option 2. Plus,
it is much easier to piggyback off of work that's already been done for you anyways.

After placing a couple breakpoints, I managed to decompile the encrypted function:

![flag function](https://raw.githubusercontent.com/thebowenfeng/asdctf-2023-files/master/2.PNG)

The program was nice enough to hardcode a large portion of the flag as a plain ASCII string, so all I had to do is trace back
and look up a few ASCII values to complete the remainder of the flag. Alternatively, you can try to reverse the logic and 
find a valid "key" which will result in the program spitting out the flag for you. I've also attached the signature for the
key [here](https://github.com/thebowenfeng/asdctf-2023-files/raw/master/pin.txt). If everything goes right, the flag should look something like:

![final](https://raw.githubusercontent.com/thebowenfeng/asdctf-2023-files/master/6.PNG)

### Final thoughts

This CTF challenge is a good example of a commonly used technique to protect softwares, especially distributable executables.
If in the future you need to protect a piece of software that you are publicizing, this is certainly a very effective technique
to have in your arsenal.
