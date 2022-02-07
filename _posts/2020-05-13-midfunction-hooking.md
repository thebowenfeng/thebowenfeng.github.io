---
layout: post
title: Performing a mid-function trampoline hook using C++
subheading: Is this the ultimate solution to bypass anti-tamper?
categories: [Projects, Reverse Engineering]
tags: [C++, Reverse Engineering, Hacking]
---

Hooking, function hooking or function detouring, refers to the act of rerouting a program's code execution in order to modify the behavior of a particular function, or intercept its parameters. It is a very popular technique used by reverse engineers, hackers and the likes, and could be very powerful when used correctly. Trampoline hooking is a newer technique that utilizes a "gateway" in order to bypass the need to write inline ASM and manually perform a detour. However, one disadvantage it has compared to a traditional detour, is it has to be **performed on the first byte** of a given function, in order to maintain stack integrity. In today's article, we will explore an alternative method in order to perform a "mid-function" trampoline hook.

[Full code gist is here](https://gist.github.com/thebowenfeng/1af710c332b75c9195ed06eb9945e265)

### What exactly is trampoline hooking?
As you may already know, a normal detour is simply some clever code that will force the program to redirect its flow of execution, hence **detouring**. Here is a rough diagram of how it is done:
![detour](https://i.imgur.com/9HLPLsx.png)
By changing a few bytes of the original program's code, we can place a jump that jumps to our function, executes our code, then jumps back. This method is straightforward, and very hard to detect. However, it has a few drawbacks:
- Function can only be written as in-line ASM, cannot have prologues/epilogues and must not distrub the stack in any form or way.
- Most compilers do not support x64 inline-ASM (such as MSVC). The ones that do support (clang) it does not support the "naked" attribute (which removes prologues/epilogues), meaning you have to manually remove the prologue, at the very least.

It is for these reasons that many people nowadays often opt in for a newer technique, called **trampoline hooking**. Trampoline hooking will roughly look like this:
![Tramp hook](https://i.imgur.com/vquFLQd.png)
Similar to a standard detour, the code execution flow is redirected via a jump. However, as opposed to a few lines of assembly code, the "hook function" can be a full-fledged function that is able to perform any computation. At the end, it will redirect to a gateway, which will restore the stack by executing the stolen bytes, and jump back to the original function. 

The benefits of a trampoline hook, of course, no need to write assembly (yuck), and overall greater possibility as you do not have to worry about keeping the stack intact; the compiler will automatically clean up the stack for you after executing your hook function. However, the downside is:
- You can only place your hook **on the first byte** of the target function, as the stack has to be *pristine* in order for this to work.

Well, what if I told you we actually don't have to necessarily perform it on the first byte, with some clever tricks? 
But first, let's discuss the motivation behind this
![shocked](https://external-preview.redd.it/BiDC1ZxXOw4lP0nwG8WBEJcJ-paiZeF3qq2msUqTftU.jpg?auto=webp&s=95758fef805cfeb23ee971f1174c666284795dc2)
### Why is mid-function hooking so important
You see, sometimes developers don't necessarily want people messing around with their program. DRMs and AntiCheats are some of the more common measures to prevent people from "hacking" certain programs. Although in most cases their concerns are valid, in rare cases, there might be a legitimate case for someone to "illegally" modify a program. Anyhow, this article isn't an ethics debate, so continuing on...

One of the most common techniques anti-cheats (or similar programs) use is checking the first byte of functions. Anti-cheat developers also realize how lucrative function hooking is, and because most people have to resort to trampoline hooking, they realized that checking the integrity of the first byte is a very efficient and effective solution. It is not computationally demanding (at least compared to checking every byte), and it directly stops most people from performing trampoline hooks. 

So, if one manages to find a way to place their trampoline hook, even just a few bytes down, it would completely circumvent their entire detection vector, at least in theory. There are far more effective ways to prevent WPM (write process memory), but not every anti-cheat is equipped with them, so such knowledge is still relevant.

### So how do we do it?
Stack integrity is everything when it comes to hooking. If you modify the stack without the program's knowledge, it can lead to catastrophic failures. It is precisely the reason why trampoline hooks cannot be performed lower down in the function, as our "hook" function expects a clean stack. So, as long as we can figure out a way to "clean" the stack, so to speak, we can theoretically perform our trampoline hook anywhere. In practice, there are certain soft limitations, which will become apparent later. So, let's get started!

### The victim program
I have developed a very simple testing program for the purpose of this article. However, in practice, the workflow should be extremely similar. Here is what is looks like:

```cpp
#include <iostream>
#include <Windows.h>

int toHook(int myInteger) {
    myInteger = myInteger + 5;
    return myInteger;
}

int main()
{
    int value = 0;
    while (true) {
        if (GetKeyState(VK_SPACE) & 0x8000)
        {
            value = toHook(value);
            std::cout << value;
            std::cout << "\n";
            Sleep(100);
        }
    }
}
```

The goal, by the end, is to hook the "toHook" function (duh) and hopefully also modify its parameter value. Right now it is incrementing the parameter by 5, let's see if we can change that to 100.

### Reversing the function
Firing up a debugger, we can easily see the function's assembly code:
![asm code](https://i.imgur.com/XKtnEYu.png)
It looks like a very standard cdecl prologue (because it is). Normally you just have to count 5 bytes and nop all associated instructions, but since we wanted to place our hook lower down, we ultimately need to find a way to maintain a clean stack.

Now you might say "Hold on a second, can't we just pop the ebp and then place our hook after"? And you would be 100% correct. Like I said, as long as you maintain a clean stack, it doesn't matter where you put your hook. Obviously, maintaining a clean stack gets exponentially more difficult, the lower you go. However, in this case, you can see we could quite easily clean the stack all the way up to `push edi`, as most of these instructions are push, meaning reversing them is as easy as just having a corresponding pop. But, in our case, for this PoC, we will simply just place our function after the `mov` instruction.

Here is what the flow should look like
1. Replace `sub esp, 000000C0` with `pop ebp` and a `jmp` instruction, jumping to your hook function. Make sure you **do not** overwrite the `mov` instruction as that is still required to maintain stack integrity.
2. Execute our hook function, do whatever we want, and eventually jump to our gateway
3. In our gateway, execute all the bytes from the beginning of our function, in order to restore the stack. Typically, we would only need to execute the "stolen bytes" (bytes overwritten for our `jmp`), but in this case, all bytes needs to be overwritten, as we purposely cleaned the stack.
4. Jump back after our `jmp` instruction, just as normal.

Voila, you just performed a trampoline hook mid-function.

### The technical implementation

The PoC code could be found [here](https://gist.github.com/thebowenfeng/1af710c332b75c9195ed06eb9945e265). However, I would encourage attempting to implement this yourself. 

*The above DLL code is written to be injected into a 32-bit (x86) process. This may or may not work on a 64 bit process, due to 64 bit addresses taking up 8 bytes as opposed to 4, which means the relative jump will only work for addresses within the 4GB range.*

We'll start with some fairly standard template code for a DLL, which defining an entry point, and creating a Thread for our code:

```cpp
BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        CreateThread(0, 0, MainThread, hModule, 0, 0);
        break;
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```
Next, we need to create a function that is able to allocate a code cave and write our stolen bytes. The function is as follows:
```cpp
BYTE* trampoline(BYTE* src, BYTE* dest, int len) {
    if (len < 5) return 0;

    BYTE* gateway = (BYTE*)VirtualAlloc(0, len + 5, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);

    memcpy_s(gateway, len, src, len);

    uintptr_t relativeAddr = (src + 3) - gateway - 5;

    *(gateway + len) = 0xE9;
    *(uintptr_t*)((uintptr_t)gateway + len + 1) = relativeAddr;

    detour(src, dest, len);

    return gateway;
}
```
There are a couple things worthy of attention. First, your "length" cannot be less than 5, as that is the size of the `jmp` instructions + a 4 byte address. `VirtualAlloc` is more or less a `malloc` in C, except that it offers more control, as it is a winAPI function. We require read & write permission, hence `PAGE_EXECUTE_READWRITE`. The rest of the code is simply a matter of copying the "stolen bytes" (which is the bytes from the start of the function till the last instruction we overwritten), and then placing a jmp at the end, to jump back to the original function. 

Now you may wonder what the "detour" function is doing. This is a left-over piece of code I had written for a standard function detour. It essentially just writes a `jmp` instruction and calculates the relative address to the destination. `detour` is what will modify the original function and forces it to redirect to our "hook" function. So here is the code:
```cpp
bool detour(BYTE* src, BYTE* dst, int len) {
    if (len < 5) return false;

    DWORD currProtect;
    VirtualProtect(src, len, PAGE_EXECUTE_READWRITE, &currProtect);
     
    *(src + 3) = 0x5D; // pop ebp

    uintptr_t relativeAddr = dst - (src + 4) - 5; // Relative jump offsetted by 4
    *(src + 4) = 0xE9; // Start jump 4 bytes in

    *(uintptr_t*)(src + 5) = relativeAddr;

    VirtualProtect(src, len, currProtect, &currProtect);
    return true;
}
```
You may see some similarities between certain code in this function, and in the trampoline function. Most of it is fairly straightforward, `VirtualProtect` allows us to write to places in memory where we normally cannot write to. In order to restore the stack, we need to `pop ebp`, so we simply overwrite the 4th byte with `0x5D` (instruction for pop ebp). Then, at the 5th byte, we start writing our `jmp` instruction. The relative address will be 5 bytes after the `jmp` instruction, which together with the 4 byte offset, will become 9 bytes. 

Finally, we need to write our hook function, which is actually the easiest part. Here is the code:
```cpp
typedef int(__cdecl* toHook_t) (int a1);
toHook_t origFunc;

int __cdecl hookFunc(int a1) {
    std::cout << "Current value: " << a1;
    std::cout << "\n";
    return origFunc(a1 + 100);
}
```
We'll define a function pointer, to the original hook function. The calling convention (in this case `__cdecl` ) is very important as it has to be consistent. Different calling conventions have different methods of cleaning the stack. Some requires the caller to clean, whilst others requires the callee. In any case, you would most likely need to perform some static analysis using IDA Pro or similar, in order to confirm a function's calling convention. The reason why we are "returning" in our hook function, is to allow the compiler to generate code **as if** we were calling the function, which means once we restore the stack, the original program will have no clue anything even happened. This is the beauty of trampoline hooking. We have the complete freedom to intercept and pass in bogus values, and the program will not have any clue. 

The final step is to execute above said functions, in a main thread. Here is the code:
```cpp
DWORD WINAPI MainThread(LPVOID param) {
    AllocConsole();
    FILE* f;
    freopen_s(&f, "CONOUT$", "w", stdout);

    uintptr_t baseAddr = (uintptr_t)GetModuleHandle(L"midFuncHookVictim.exe");

    origFunc = (toHook_t)(baseAddr + 0x12500);
    origFunc = (toHook_t)trampoline((BYTE*)origFunc, (BYTE*)hookFunc, 9);

    return 0;
}
```
It is worth noting that we changed the address of the origFunc from its address in memory, to our gateway (as that is the return value of the trampoline). We need to ensure our hook function redirects to our gateway in order to prevent "hook recursion". In other words, if we do not specify our gateway address, the hookFunc will simply jump back to the start of the function, and we will have an infinite loop.

### Closing thoughts
The relative ease, with just a little bit more code and attention to reversing, makes this technique a very powerful technique when it comes to dealing with "anti-cheats". Of course, as I have covered above, there are many other ways to prevent WPM aside from byte-checking, and of which are much more powerful and much harder to bypass. However, this is not to say that every single anti-tampering service is willing and able to implement more advanced checks. Byte checking is still a very prevalent techniques used by such programs, simply due to its ease of implementation, and relative efficiency. In any case, regardless of its usefulness, knowing an extra technique will not hurt anyone (maybe except the program you are attempting to modify).




