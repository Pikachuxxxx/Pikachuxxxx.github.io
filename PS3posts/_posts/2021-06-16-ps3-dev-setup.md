---
title: Setting Up your PS3 for Development
tags: [PS3, C++, Homebrew]
style: fill
color: danger
description: How to setup the SDK and and your environment to develop for the PS3 system
blog: [PS3]
published: true
permalink: /ps3/:title
---

Well Welcome! If you're here it means you are in for some nostalgic adventure. The PlayStation™3 system is one of the most hardest and intriguing hardware ever built. It's unique cell architecture makes things challenging, which also makes the entire process of development even more exciting. In the coming blog posts we'll dive deeper into the Architecture and detailed tutorial series on how to develop and take full advantage of the SDK and use other industry standard frameworks.  

Before we get all technical let us first setup out environment and explore some samples, samples are a great way to stay motivated and explore the SDK. We'll see what all tools we will be needing along with the SDK and configure everything to start our journey deep down this rabbit hole. I will also explain the structure of the SDK in detail in the coming posts.

Good news! You don't really need an actual PS3 Development kit which looks like this :
![](https://pikachuxxxx.github.io/assets/images/blog/ps3/dep-setup/ps3devkit.png)

You can convert your own PS3 into a dev kit. But there's a catch, If you're lucky enough and have either a **PS3 Phat** or one of the **PS3 Slim(CECH-200xx, 250x)** models you can jailbreak them!. I hate to disappoint you but it doesn't work on any of the PS3 super slim models and slim models above 250xx. Yes! you'll need a jailbroken console running on a specific custom firmware (Rebug DEX) to actually develop. We will also be using a lot of leaked and deprecated stuff, but that's the fun of it right!

***

# Jailbreaking your console
There are many amazing youtube videos explaining this so I'm going to link to some of them here. You can continue reading this after you have jailbroken your console.

[How to Jailbreak Your PS3 on Firmware 4.87 or Lower!](https://www.youtube.com/watch?v=Eckd06nFReY) by [Mr.Mario2011](https://www.youtube.com/channel/UC-YlkP3c1zKUPfyMMurARAQ)

[How to Downgrade CFW on a Jailbroken PS3](https://www.youtube.com/watch?v=WoMsVT_ewW4) by [Mr.Mario2011](https://www.youtube.com/channel/UC-YlkP3c1zKUPfyMMurARAQ)

[How to Update CFW on a Jailbroken PS3](https://www.youtube.com/watch?v=ar2-eCzT2wI) by [Mr.Mario2011](https://www.youtube.com/channel/UC-YlkP3c1zKUPfyMMurARAQ)

**Remember to use the Rebug Custom firmware, because we will be using the DEX(developer firmware) version of it!.**

If you need to downgrade your system no worries! Here is a video on it. (It will probably be necessary if you're on 4.88 because Rebug-DEX is only available up to 4.84)

***

# Converting from CEX to DEX

I would suggest you to follow this amazing youtube tutorial on [How to Convert a Jailbroken PS3 from CEX to DEX](https://www.youtube.com/watch?v=tmpexUf9eK0) by [Mr.Mario2011](https://www.youtube.com/channel/UC-YlkP3c1zKUPfyMMurARAQ).

#### CEX vs DEX What's the difference?

CEX is the retail version firmware and DEX is the developer firmware with additional debug features.  

***

**Congratulations**! Now you have an **unofficial PS3 Development Kit**! Hooray🎉!! (probably illegal too idk)  

Don't worry the feds don't care.  

***

# Setting up your development Environment
Now you can move to your PC and get ready be a software pirate. First you'll need to download the PS3 leaked SDK from [here](https://archive.org/details/ps3-4.75-sdk). It comes packed with a lot of samples, official documentation, tools and libraries for the PS3. I will explain the SDK structure in details so don't be confuse if things doesn't make sense.

## SDK Installation
Let's get on with the SDK installation process. Unzip the SDK and double-click the SDK installer.
