---
title: A small click goes a long way (Part 1/2)
description: Have you ever wondered what actually happens when you type something in the google search bar and press 'Enter'? In this series, I will explain the complex path between you and your search results!
date: 2021-10-14 08:57:00+0530
image: header.png
categories:
    - The Click Saga
    - Overview
tags:
    - the-click-saga
    - overview

math: true
toc: true
---

Have you ever wondered what actually happens when you type something in the google search bar and press 'Enter'? It's a pretty massive collection of some very complicated sets of systems interacting with one another. In this article, I will break down each of them for you.

For the sake of simplicity, I will assume some things. For example, I will assume that you are on an android phone using a chrome browser and are doing a google search. The assumption will mostly be stated whenever there is one.

With that, let's dive into what happens. Each subtopic is formatted as "Event (Education field related to it)."

## Touch Input (Digital Electronics)

As soon as you press the "Google Search" button on your screen, the following things happen:

1. Your phone has what is called a capacitive touch screen. As soon as you tap on the screen, there is a change in the voltage in that area.
2. This change in voltage is detected by some hardware sensor that is connected to the display. This hardware itself may determine the location of the click on the screen, but in other cases, like, say, a mouse, it reports changes in the location and not the actual location.
3. This sensor fires something called an interrupt. This 'interrupt' interrupts the CPU, essentially informing it that there is 'some' input.

## Recognising the input (Operating Systems)

1. Now the CPU has received the interrupt and knows that some data is waiting for it. It may also know that this data is from the display, but it may not know that at this point.
2. Now, it reads the data at its leisure. This is the reason why slow phones react slowly to your touch input and not instantaneously. The processor determines when to service the interrupt.
3. Associated with each interrupt is something called an ISR (an Interrupt Service Routine). Think of it as a code that is triggered every time a particular interrupt comes in. The ISR, in this case, can trigger the driver.
4. The 'driver' is a piece of code in the 'kernel' of your Operating System (Windows, Mac, Linux, Android, iOS etc.). The driver knows the exact format of input that the hardware device, in this case, the display, is sending. It's like knowing the language that the device will talk in. And yes, just like humans, not all hardware talks in the same language.
5. This driver determines the exact location of the click. If you use a mouse or a similar device, the driver determines the new position based on the current position and the change reported by the mouse.
6. The Operating System, more specifically the GUI manager of the OS that you use, then determines the application running in the foreground and sends that application the input of the click, preferably in local coordinates. 
    - The application in a laptop etc., might not be full screen, so its coordinate system might differ.

## Determining the task to do (Web Dev and Compilers)

1. Now that chrome has received where the click happened, it is up to chrome to determine which tab was open when the click occurred.
2. Your website is almost always a combination of HTML, CSS and JS. HTML is a human-readable text which determines the elements of the website. CSS is essentially the make-up on HTML. It formats it and beautifies it. JS is the code that is triggered whenever you do something. Like a scroll, click etc.
3. So, when you click, chrome determines from the positioning derived from HTML and CSS that you clicked on the "Google Search" button. It then sees what action has to be performed on it.
4. In this case, there would be a Javascript code associated with the button that would be triggered when the button is clicked.
5. Now Javascript is an interpreted language. So chrome's JS engine will compile each line of the Javascript that it needs to run into assembly code. This code then executes on the processor.
    - I have left out the part of scheduling algorithms and related stuff when dealing with the OS itself.
    - You can find more information on compiled vs interpreted languages in my other blog post
6. In this case, the code determines that it needs to send a network request to the Google servers to get the results.
7. But first, it needs to determine the IP address of google.com. `google.com` is a human-readable name and is more complex for the computer to parse. On the other hand, the IP address is a series of bits (or numbers) identifying particular computer/s. E.g. `192.168.0.1`
8. For determining this, there is a service called DNS or Domain Name Service. A set of servers worldwide have a database of Domain names (e.g. `google.com`) mapped to their IP address/es. They are referred to as DNS Servers.
9. You send the first request to that DNS server, whose IP you know because it is sent to you by your ISP when you connect to the internet. And once you know the IP of google.com, you send a second request to google for the search results.

## Life of a packet (Computer Networks)

1. Now, chrome will create an HTTP request (the 2nd request). This request has about 3 layers in it. The top 3 layers of the OSI model. For the DNS request, all that changes is that there is DNS instead of HTTP. Other layers in the [OSI model](https://www.geeksforgeeks.org/layers-of-osi-model/) remain intact for the most part.
    - Think of the OSI layer as a box inside a box. You are in the United States and want to ship something to your friend in India, then you make a box with your contents in it and some label on the top of it (in OSI's case, this is called headers)
    - Now your box can be put inside another box labelled with the name of the city your friend is in, and that box in a box labelled with the state he is in and so on.
    - The only thing is, unlike real-life objects, messages can be broken into multiple parts at any point. So a lower layer box can be subdivided without any problems.

2. Note that HTTPS (the 'S' stands for Secure) has an additional layer of encryption, but that deserves a post of its own.
3. This request goes to the kernel mentioned above. This kernel fulfils the subsequent 3 layers of the OSI model, viz. Transport, Network and Data-link.
4. Another driver in the kernel, which knows how to talk to your LAN or WiFi Hardware, gets informed that the OS wants to send some data. 
5. There is some memory on the LAN/WiFi hardware called a buffer. If the buffer is full, the OS has to wait. Otherwise, it'll put the data in that buffer.
6. The LAN/WiFi hardware will package it into the final layer (Physical layer) and send it out at its discretion.
    - All WiFi/LAN hardware has to take care that they don't interrupt other similar devices. Like your phone and laptop might both be connected to the WiFi.
    - If you studied waves in Physics, you'd know that 2 waves can interfere (either constructively or destructively), but anything that changes data is destructive in our world. And so we avoid that at all costs.
    - Most hardware sends some data and sees if they receive the same data back. If not, there is some interference and your hardware pauses and does not send data. It waits an arbitrary amount of time so ask to not clash again with the same stream it clashed with before.

7. This data is then sent to your router (which works at Network Layer). From there, it goes to the routers and switches (which work at the Data-link layer) of your Internet Service Provider (ISP). And from there, maybe all the way across the globe to some other ISP, providing service to Google.

Now, this packet made it all the way to google.com's data centre. We will continue what happens there in Part 2 of this blog!