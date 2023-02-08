---
layout: post
title: Bypass code detection with polymorphic code engine
subheading:  Writing a polymorphic engine to bypass signature-based detection of programs.
categories: [Projects, Reverse Engineering]
tags: [C++, Hacking, Reverse Engineering]
---

[Link to project on github](https://github.com/thebowenfeng/PolymorphicEngine)

Recognizing malware is one of, if not the most important part of 
protection software such as Anti-Virus or Anti-Cheat softwares. Much like
how the human immune system works, these softwares will often look for
identifiable features of suspected programs, which can then by used to
identify future copies of the same program. Today we will be looking at
techniques which effectively bypass code detection by utilizing 
polymorphic code via a polymorphic code engine.

## Brief overview of how anti-virus software work

Before we get into how polymorphic code works, it is important to
rationalize the importance of polymorphism, by recognizing how most
anti-virus/anti-cheat software work. 

Amongst other things, one of the jobs of an AV is to recognize malware.
Modern AV is quite sophisticated in its ways of malware detection, but 
one of the main ways it is able to retain memory of a particular malware, so
that it can recognize the malware in case it pops up in the future, is by utilizing
a technique called **signature scan**. It is a very old, yet very
effective method of recognizing malware and it is what polymorphic code
aims to combat.

### So what is signature scanning?

A lot of what anti-virus does draws parallels with the human immune system.
One of the primary ways our immune systems is able to recognize viruses is
because it has seen it before. In other words, once the immune system
sees a virus, it'll remember its key characteristics so that next time,
if the same virus shows up, it'll know what it is and is able to effectively
target it.

In the same way, **signature scanning** is all about "remembering" what different
programs looks like so that anti-virus softwares can identify certain malware.
In this case, anti-virus will typically use a program's code to identify it.

In practice, typically what will happen is once a malware is identified by someone,
most likely a researcher, the person who identified it will publish the malware's
signature. This could be the hash of the program's binary content, or specific
lines of code that the person thinks can uniquely identify this piece of malware.
The signature will then be published to some sort of database, and once an anti-virus
sees a program that matches the signature, it can then promptly remove the malware. It is
fast, efficient and most importantly easy to distribute without having
to perform manual updates to the anti-virus.

Similarly, anti-cheat softwares works the same way, except that instead of identifying 
malware, it primarily targets cheating software.

## How does polymorphic code defeat signature scanning

Polymorphism comes from a greek word, meaning "many shaped". It is a rather
widely used term in Computer Science and software development. In this particular case,
what polymorphism, or polymorphic code means, is a program that can alter
its own code.

This poses a serious threat to the aforementioned technique of signature scans. If
an anti-virus relies on signatures to detect a piece of malware, then if the malware
changes its code (and thereby its own signature), then it suddenly becomes a brand new
program and the anti-virus is none the wiser. If the malware developer can
somehow automate the process, in other words develop a polymorphic code engine, then the malware
can essentially evade signature scans indefinitely by just keep mutating
itself once its been detected.

That being said, mutating code is not easy. The key challenge is to
change the code without changing its behaviour. To be more specific, 
imagine writing a piece of code. Now someone asks you to write that same piece of code,
but in an entirely different way. Now imagine if you have to do it again, and again, eventually
you will most likely run out of ways. This is what a polymorphic code engine has to do, which is
why writing a **good** polymorphic engine is extremely difficult. That being said,
I will discuss a relatively simple method of mutating code.

## How does simple polymorphic engine work

At its core, this polymorphic engine seeks out functions that are
not reachable from the code's entry point, and mutate them with random bytes.

Some compilers (such as Microsoft's MSVC) will bake in certain functions,
such as functions that interact with the OS, into every program. Not every one
of these functions will be actually utilized, so when they aren't, they are
effectively junk code, which means changing them will not actually affect
the program's behaviour. Furthermore, users can deliberately write so called "junk functions",
into their own program to facilitate code polymorphism.

The way the engine knows which functions are junk, is by tracing 
every possible paths from the program's entry point. It will then
disassemble the program, iterate over every identified function
and checks against the list of functions that was traced previously.
If a given function is not in the list of traced functions, then we can
safely assume that this function is not used, and mutate it.

### Shortcomings

Although this method is relatively easy to implement, its major shortcoming
is that it actually doesn't mutate every function. Any signatures built
on any function that is actually being used (and hence not mutated) is still going
to be valid. 

However, in practice, most anti-virus often build multiple signatures against
a single binary, and will not typically aggressively target a binary for matching
only one or two signatures. This means that even mutating a couple signatures is
sometimes enough for a malware or program to "fly under the radar".
