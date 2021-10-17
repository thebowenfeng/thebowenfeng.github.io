---
layout: post
title: Writing a custom malloc and free implementation using C
subheading: How does malloc actually work under the hood?
categories: [Projects, Dynamic Memory]
tags: [C, Dynamic Memory, Malloc, Heap]
---

# Writing a custom malloc and free implementation using C

Dynamic memory and malloc have been a staple feature in the C programming language. It is both feared and respected by people, as it provides great power but is also very easy to screw up. However, most people have never wondered what goes on under `malloc`, and just take things for granted. In this article, we will be exploring "under the hood" mechanisms of malloc, as well as coming up with our very own algorithm to allocate and de-allocate memory.

The full project source code can be found [here](https://github.com/thebowenfeng/memalloc).

### What is heap, malloc and free?
![magic words](https://i.kym-cdn.com/photos/images/newsfeed/001/909/636/a8c.png)

For anyone who is unfamiliar with C, or just dynamic memory in general, the **heap** is simply a section in memory that is reserved for you, the programmer, to put stuff in it. You can imagine it as an invisible, flexible global array that you can read/write anytime, anywhere, within your program. The heap is not to be confused with the heap data structure, they are completely two different concepts. Whereas the "stack" is built using a stack data structure, the heap is **not** built using a heap data structure.

Malloc, then, would simply be a function that allocates (reserves) some memory on the heap for you, should you need it. For instance, if I need to store an integer on the heap, I would simply call `malloc`, give it the size of an integer, and `malloc` will give me a pointer, pointing to the location of my integer variable. Of course, malloc isn't restricted to only basic datatypes. For instance, you could also malloc custom structs. 

Free is the anti-malloc. Whereas malloc reserves memory space for you, `free` takes away that reservation and returns memory back to the OS. Obviously once we no longer need something that we malloc'd on the heap, it would be ideal that we "throw away" that variable and let something else use the available space. If we do not, then a "memory leak" would occur, where your program will continuously consume memory until your OS crashes. 

### Memory fragmentation and the difficulty of a good malloc algorithm
So imagine if we had a global, array of bytes (char) that act as our very own heap. Since it is global, functionally speaking, it would be very similar to the actual heap, where we can access it anywhere in our program and store information on it.

Now imagine if we have a naive malloc algorithm that simply linearly scan through the heap, start to finish, and chuck things in there if it finds a big enough space. Similarly, the free algorithm will simply overwrite existing bytes with null bytes if needed.

Now, say we want to store a variable of size 5 bytes to the heap. Since the heap is empty, the algorithm simply put it at the start of the heap, like this:

![First heap assign](https://i.imgur.com/tu9gi4u.png)

Ok great, now we want, let's say, 3 bytes of data on the heap. Logically, the algorithm will scan through the heap, go to the end of the 5 bytes data, and put the 3 bytes there, like this.
![Second heap assign](https://i.imgur.com/rfSpaCB.png)

Now, let's say we want to have, another 5 bytes assigned. So we put it after the 3 bytes, like this:
![Third heap assign](https://i.imgur.com/nxOrmWf.png)

Now the interesting stuff happens. We want to *free* the middle "3 bytes" of data. Okay, fairly straightforward, we simply nullify the 3 bytes of data.

![Free 3 bytes](https://i.imgur.com/cnNAOPP.png)

Some of you might already know where I'm going with this (hint: memory fragmentation) but to solidify my point, let's now attempt to assign 10 bytes of data.

The algorithm scans through the heap, finds the 3 bytes of free space. **However**, because it is too small to contain the 10 bytes of data, it gives up and continues on until the very end, and put the 10 bytes **after** the last 5 bytes of data, like this:

![Fragmentation](https://i.imgur.com/BvyDyBl.png)

So, now we basically have this 3 byte gap in the memory, which will forever remain there unless something smaller or equal than 3 bytes is to be assigned to the heap. 

This is called **memory fragmentation**, where memory in your program allocated in non-contiguous (continuous) blocks, essentially leaving unusable gaps in-between. In other words, not utilizing the full capacity of your memory, or wasting memory. It is a very fundamental problem that malloc has to solve in order to be space efficient. 

In order to solve this, there are two basic methods:
1. You could dynamically re-organize the heap once something is free'd. In our case, we would push the 2nd "5 byte" data backwards in order to fill in the gap.
2. You could store data non-contiguously. In our case, for the 10 byte data, we could store 3 bytes of it in the tiny gap, and the rest (7 bytes) after, with information linking the two "blocks" together. 

![Method 2](https://i.imgur.com/gAdJzwn.png)
(Diagram demonstrating method 2)

The first method is inefficient, slow, and impractical. If we forget the computational resource needed to re-organize the heap, changing the location of data means we also have the change the location of the pointers allocated to those data, otherwise the pointers will be pointing at garbage value. 

The second method is a bit slow, requires extra space, but much faster than the first alternative, and completely solves memory fragmentation. Most modern compilers will use a similar idea (in terms of dividing memory into blocks and looking for free blocks/reserving blocks), and it will be the basis for my own malloc algorithm. 

### The malloc algorithm

As discussed above, the core concept behind this algorithm is to divide memory into blocks. One piece of data could be split between multiple blocks, and linked together using a linked list. When reading/writing the data, we simply need to traverse the linked list. 

Each memory block will contain the following data:
- Address: The index location of the first byte on the heap covered by the current block. Remember, the heap is simply a char array.
- Size: The size of the current memory block. In other words, how many bytes does this block cover.
- Next: A pointer to the next block, should a piece of data be split between multiple blocks. This can be NULL. 
- isFree: A flag variable that marks whether or not this memory block is free to be used, or reserved for something.

In this case, malloc would be responsible for assigning, creating and splitting memory blocks, so it is the most complicated part of the entire project. For each memory block encountered, there are three possibilities:
1. The given data is bigger than the memory block's capacity
2. The given data is smaller than the memory block's capacity
3. The given data has the same size as the memory block's capacity
4. No free memory block exists.

#### Case 3 (same size)

Let's start off with the simpliest case, where the given data perfectly matches the size of a memory block. In this case, our work is easy. We simply need modify the heap at the address (see block struct definition above), with the size of the block. Nullify the next pointer of the block, as the data is only contained in this block alone, and return the address of the block.

#### Case 4 (no free block)

This case is also relatively easy to handle. This means that every single memory block is filled/reserved, and we need to create a new memory block. We would simply need to find the "max address", or the block that is the furthermost to the right of the heap (as the heap is filled from left to right), then create a new block that is right after the last block. The size of the block will be the data size, and obviously next pointer will be NULL. Return the address, and add the block to a global memory block array (or you can malloc it on the heap but that's a bit ironic, considering we are trying to write our own malloc).

#### Case 1 (bigger than)

Now we move onto a more interesting case. What if the data size is bigger than the size of the current memory block? Well, as demonstrated in the example scenario above (10 bytes scenario), we would simply "split" the data into 2 parts. The first part would be stored in the current memory block, and the next part would be stored in the next available position. Except there is one more important detail: The "next" part can also be subsequently split, if the next available memory block has a smaller size than the "next" part. So, for instance, in our above example, let's say the next available memory block is also 3 bytes, then the remaining "7 bytes" would be again split into 3 bytes, and now the remaining would be 4 bytes. 

In the actual program, once we have found an available memory block, we would copy the first N bytes of the data into the heap, at the current given address. Then, we would have a marker variable that marks the start of the 2nd (next) partition, essentially, splitting the data into 2 parts. The function will then continue scanning for memory blocks until it finds another free one (or cannot find one, in which case we go to **Case 4**), and applies the same processes depending on the circumstances. The partition could either be split into 2 partitions again, or it could be stored entirely in a block that's big enough (**Case 2 or 3**). In any case, we would also need to link the current memory block to the next memory block. We can accomplish this by saving the current memory block somewhere, and when we do find an appropriate block, edit the saved block and link it.

Basically, we "recursively" apply the function to the 2nd partition (except there is no need for actual recursion), until we have successfully stored all parts of the data.

#### Case 2 (smaller than)

This is, in my opinion, the most interesting and also the most complicated case. What if our data is smaller than the current block's capacity? Well, I suppose you could just store it in there and call it a day, but doing so will defeat the purpose of the entire algorithm, as it will cause memory fragmentation. The "leftover" bits in the block will be sitting there, inaccessible to any other data as the entire block is reserved. 

So, the solution is to split the block. You might be observing a pattern here. Previously, we split the data because it is bigger, now we split the actual memory block because it is bigger. So how do we actually split it?

Well, we can't actually physically "split" something. What we do, is we can adjust the current block, and create a new block, representing the leftover space of the current block. The reason why we need to specifically create a new block, is to ensure the entire heap is covered in memory blocks. If we do not, then again, there will be a gap, inaccessible as it is not mapped to any existing memory block. 

So, in our function, we would calculate the size difference. We would adjust the current memory block, by decreasing its size to the data's size. Then, we would create a new block, much like how we created a new block in **Case 4**, except the size will instead be the "size difference" we have calculated just then. The new block will be marked available for use, and a new piece of data in the future could utilize that block.

#### Analysis of speed and efficiency

So now that we have covered the algorithm, we could examine its time and space complexity. Let's start off with time

Obviously, at the very start, the time complexity would be very close to O(1), since there are no memory blocks (or not too much) yet. However, as we go on to use the heap, the complexity will gradually increase, where the worst case scenario would be O(n), where each block has been sub-divided so many times that its size is 1 byte. However, such case is unlikely, as it requires a lot of tiny data being malloc'd onto the heap. In general, the time complexity would be O(n / k), where n represents the heap size, and k represents the average data size. Obviously, k cannot be larger than n, so as k grows larger (a lot of huge data), less memory blocks is needed and less traverse time.

Space complexity is essentially the same. At the start, there will be very little space used to store memory blocks, but as time goes on, the space required to store all the blocks will inevitably grow. Similarly, frequent use of smaller data will cause the memory blocks to be sub-divided further, thus increasing space to store blocks.

The average case? Hard to say, because it really depends on how you are using the heap. The main issue with this algorithm is the space complexity, not necessarily the time complexity. Even a O(n) time complexity would be extremely fast for most use cases. However, a O(n) space complexity would be quite disastrous, as there will literally be more space used to store these blocks, than the actual size of the heap, again defeating the purpose of the algorithm. 

### The free algorithm

The free algorithm, in our case, is very straightforward, given the foundation we have built in the malloc algorithm. Given a valid address to the heap, we linear scan all memory blocks, find the block with the corresponding address, then traverse the entire linked list of blocks, and mark each and every block as "free". That's it, not special tricks needed. 

### Read/write to heap

What's the use of the heap if we can't access it? Of course we need functions that are able to extract the information that is stored on the heap. Luckily, given the foundation built by malloc, reading and writing is very easy.

Similarly to `free`, we need to scan through memory blocks until a matching block is found, with the correct address. But instead of marking blocks as "free", we simply copy the content on the heap, at each block's address, into a buffer variable and traverse to the next block. 

For writing, the reverse is done, where we would copy the stuff **from** the buffer **to** the heap at the given address at the current block. 

A possible problem is with the current implementation is that it is reliant on the end-user to supply our R/W functions (read/write) with a valid address. What if they supplied a garbage address, yet it coincidentally matches a memory block? The program have no way to distinguish and will treat it as a "starting block". Although the fix for this would be quite trivial. Simply add another flag to the block, `isStarting`, that marks whether or not the current given block is the starting block. 

### Final thoughts

No doubt modern, robust compilers like GCC will have a way more optimal algorithm for malloc and free, yet the core concept between most modern malloc algorithms stay the same. They all rely on the concept of dividing memory into different blocks. This is precisely the reason why sometimes malloc feels sluggish and slow. Whilst you can comfortably sit there and enjoy memory being magically given to you, malloc has to do the hard work of managing your heap to minimize memory being wasted. As with most algorithms in Computer Science, there will often be a trade-off between time and space. In this case, "space" is your heap and time is the speed of malloc. A naive malloc will no doubt be speedy, but the speed trade-off is huge. Conversely, a "space-optimized" algorithm like this requires time. So, in the end, it is about finding the right *balance*. As Thanos would say "Perfectly balanced, as all things should be". 

![Thanos](https://www.meme-arsenal.com/memes/de7e96af20765406d4b81d81cffc6268.jpg)
