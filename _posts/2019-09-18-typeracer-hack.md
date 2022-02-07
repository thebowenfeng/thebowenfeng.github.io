---
layout: post
title: Using web drivers to automate typing races in Python
subheading: Can webdrivers be used to cheat in a typing game?
categories: [Projects, Web Automation]
tags: [Python, Selenium, Webdrivers, Web Automation]
---

The power of web automation is apparent, whether it is automating a tedious task, or performing automated testing on a website. However, with great power comes great responsibility. Web automation tools can be easily abused, spam being the main issue. Today we will examine the use of web automation in a slightly less nefarious manner, automating [TypeRacer](https://play.typeracer.com/) using Python, Chrome WebDriver and Selenium.
[Full source code here](https://github.com/thebowenfeng/typeracer-hack)

### What is typeracer and how does it work
Typeracer is a famous typing competition-styled game. Each person controls a virtual car, and the faster you can type-out a random paragraph of text, the faster you car moves. Mis-typed words/characters are not counted and you need to delete them before continuing. 

The idea is to use Selenium to extract the text, and automatically type out the words for us. **I do not condone cheating and this was done purely as an experiment**. But before we can do that, there are a few things we need to resolve...

### Typeracer's anti cheating mechanisms
Before you think you can type 1000 words per minute and be on the number 1 stop on the leaderboards:
![Stop](https://i.kym-cdn.com/entries/icons/facebook/000/027/242/vault.jpg)
Typeracer is not new to people using unfair means to gain an advantage. For starters, they have a CAPTCHA system where you are required to type out an image-based text if your WPM (words per minute) exceeds a certain threshold, or if your WPM has drastically improved suddenly. *We will not be discussing how to bypass this CAPTCHA in this article.* In any case, there are certain limits you can push in terms of your typing speed, before you are inevitably presented with this CAPTCHA test. 

In addition, typeracer also has its own backend algorithm to flag suspicious users. For instance, if someone's typing speed is consistent and their accuracy is 100%, then it would be highly suspicious as no human is able to maintain 100% consistent typing speed. We will address this later, in the implementation section.

Finally, the ultimate "anti-cheat" is in fact the webdrivers themselves. You see, Google and other companies realized the potential for webdriver abuse, so they have implemented certain mechanisms to identify itself to the website, and any website can easily use this and ban webdrivers from accessing their site. There are, as with anything, ways to bypass such detections by modifying the source code. However, again, this will not be part of the scope of this article.

### How is this made
 Simulating user input is quite trivial in Selenium. All you need is to find the HTML element for the "textbox" that receives user input, and use a function "send_keys" to simulate a user typing. However, there is a slight wait before each race, in order to have enough people in a single race. To bypass this issue, we simply repeatedly search for the "countdown popup" until it is gone, which signifies the start of the race. 

The most interesting part of this project, is the algorithm to circumvent "soft" cheating detections, as I have mentioned above. Namely

 - Perfect consistent typing speed across a single/multiple games
 - Perfect accuracy.

If we are able to implement something that can remove these two conditions, we have effectively "humanized" our bot, at least in the eyes of the Typeracer anti-cheating algorithm. So, what's the solutions?

![Random](https://upload.wikimedia.org/wikipedia/commons/thumb/3/36/Two_red_dice_01.svg/1200px-Two_red_dice_01.svg.png)
Yup, you guessed it, random number generator!

Well, to be more technical, we will randomize the typing speed across the entire game, and have randomized "typing mistake" events throughout the game.

First, let's discuss randomized typing speeds. Since we are essentially looping through the target text, and computers have become fast enough that each iteration in a while or for loop has almost zero delay, we can assume no "natural" delay between each character to the next. This will obviously lead to an inhuman typing speed. So, to combat this, we need to implement some form of artificial delay, which is time.sleep() in Python. For those who aren't familiar, time.sleep() will pause the current thread of execution for a specified amount of time. Now, as discussed previously, we would need to randomize this delay, in order to achieve a certain level of unpredictability. So, we simply need to use the "random" library in Python to generate random numbers at each pass, and use that number as part of our delay, thereby yielding non-constant typing speeds.

Implementing a typing error is slightly more difficult. We would need to make a deliberate typing error, delete the erroneous text, and re-type the correct text in order to progress. Of course, the probability of causing this event needs to be randomized (again, using random library). But, we also need to randomized the "error length", or in other words, how many characters do we type before we realize our mistakes and begin backtracking? Now you may say 5, or 10, or 15, but as we have discussed again and again, humans are unpredictable and as such, we need to make our bot as unpredictable as possible. Therefore, the most optimal way is to randomize the "error length". So in one mistake, we could type 3 characters before backtracking, in the next one, perhaps 10. The next problem is, well what do we type for our erroneous output? Some might opt for some constant, such as typing 3 "a"s or 5 "b"s. Some might say "how about we type random gibberish"? Although that is a valid point, we also need to factor in the human factor. You see, typically when people mistype, they would mistype one character and continue on typing the rest, correctly, but off-setted by one (or how ever many they typed wrong). So, my algorithm will simply skip 1 character, start from the next character, and continue typing, which is a totally reasonable human error. Then, after a random amount of character being typed, our bot will "realize" and starts using backspace to delete the wrong characters, and continue typing correctly. 

### Conclusion
By no means is this a perfect typing bot. You will most likely be banned if you use this in an actual race, and I have no desire to spend time to patch the webdriver, just to cheat in a typing game. However, it is an interesting journey to see just how useful web automation can be, even in the most unlikeliest of places. 
