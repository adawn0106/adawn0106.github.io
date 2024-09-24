---
layout: post
read_time: true
show_date: true
title: "Fundamental Knowledge of V8 engine (V8, Pipeline, Garbage collection, Ubercage)"
date: 2024-09-25
img: posts/20240925/v8.png
tags: [V8 , prior knowledge , pipeline , Garbage collection , Ubercage]
category: prior knowledge
author: adawn0106
description: "Before explaining the V8 exploit, I would like to briefly introduce some prior knowledge you need to understand, including V8, Pipeline, Garbage Collection, and Ubercage."
comments: true
tocs: yes

---


The following content was originally written for our team blog.
At the time, I didn't have a personal GitHub blog, so I'm posting it now after setting one up. 
This research was not conducted alone but was a joint effort between two people. 
To gain more experience, I plan to redo the entire process by myself from the beginning and will add any new insights I discover along the way.




## what is v8?

Chrome V8 is an open-source JavaScript engine used in the Google Chrome web browser and the Node.js server environment.
V8 is designed to efficiently execute JavaScript code, aiming for fast performance and optimization.
Here is a table showing the global web browser market share, and as of December 2023, we can see that the Chrome browser holds a staggering 64.7% share.

The engine used by the Chrome browser, which I'm currently introducing, is the V8 engine.

<center> <img src="https://github.com/user-attachments/assets/0a823c86-6e55-41b4-a956-28a7ab7cd5b5" /> </center>
<p align="center"> 
    [source : <a href="https://gs.statcounter.com/browser-market-share#monthly-202312-202312-bar/">Browser Market Share Worldwide</a>]
</p>


It seems Chrome might need to make more of an effort so that searches for 'V8 engine' immediately bring up the Chrome V8 engine instead.

<br>

 ### V8 engine components (Pipeline)

The V8 engine is composed of various components. To make it easier to understand, I have attached two images below.

The components of V8 are as follows:

Parser, Ignition, Sparkplug, Maglev, TurboFan

<center> <img src="https://github.com/user-attachments/assets/88795d10-86c5-45f2-9bc8-0ad99580b759" /> </center>
<p align="center"> 
    [source : <a href="https://medium.com/dailyjs/understanding-v8s-bytecode-317d46c94775">V8_pipeline</a>]
</p>

<center> <img src="https://github.com/user-attachments/assets/6aaf0bad-08c3-4520-966f-7f8f8f6ed823" /> </center>
<p align="center"> 
    [source : <a href="https://docs.google.com/document/d/13CwgSL4yawxuYg3iNlM-4ZPCB8RgJya6b8H_E2F-Aek/edit#heading=h.dmhxljs5hbh">Pipeline with Maglev</a>]
</p>
I will briefly explain each component.

- Parser: Converts the JavaScript source code to an AST (Abstract Syntax Tree) structure.

You can view it easily through the site below.

[https://astexplorer.net/](https://astexplorer.net/)

<center> <img src="https://github.com/user-attachments/assets/1544592a-d3c2-45ff-ab6d-97d689cfa2a6" /> </center>

<center> <img src="https://github.com/user-attachments/assets/d4214fee-b9f1-4543-a9b5-bec505bf1d1d" /> </center>

<br>
- Ignition: As V8's interpreter, Ignition converts the parsed AST structure into bytecode.
Since it compiles and generates bytecode directly without first compiling the code into machine code, it has the advantage of being extremely fast.
In addition to executing the code, Ignition also plays a role in collecting profiling information that will be used for later optimization.

This image shows a portion of the bytecode.
<center> <img src="https://github.com/user-attachments/assets/e5b646c6-259a-40cd-bcc5-0e83f93f5841" /> </center>

<br>
Based on the profiling information collected by Ignition, V8 determines the level of optimization needed to improve performance and execution speed using a JIT compiler. 
There are three types of compilers involved in this process.

- Sparkplug: A JIT compiler that performs very fast and simple optimizations.
  It doesn't perform extensive optimizations but simply converts bytecode to native code quickly.
- Maglev: A JIT compiler that performs mid-tier optimizations. It conducts static analysis of the source code to achieve this level of optimization.
  To analyze the optimizations done by Maglev, one can use the Maglev-IR-Graph to see the process of generating machine code.
- TurboFan: V8's most powerful optimizing JIT compiler, which features dynamic analysis followed by optimization and compilation.

The reason these three compilers were developed is that initially, there were only Ignition and TurboFan. 
However, a performance and execution speed gap emerged between these two, leading to the release of Sparkplug to bridge that gap.
Later, another speed difference arose between Sparkplug and TurboFan, which prompted the release of Maglev.

Which compiler performs the optimization is determined based on the profiling information collected by Ignition. 
Generally, this decision is made according to the number of times the code is called, so it's good to understand that as the number of calls increases and the code is recognized as "hot," optimizations will be applied.

## Garbage Collection of V8

I intentionally titled it "V8's Garbage Collection" because V8's Garbage Collection differs from Java's Garbage Collection.

The core concepts are quite similar, so if you're already familiar with one, it will help you understand the other more easily.

This Garbage Collection function manages the heap in memory by identifying unused objects and reclaiming the memory occupied by them.

<center> <img src="https://github.com/user-attachments/assets/0d579d8a-59ff-4a3c-a161-1508caa6db58" /> </center>

<br>
V8's heap is basically divided into different regions, with the areas where we allocate memory being the Young space and the Old space. These two regions manage memory in different ways.

### Young space

This region is where newly created objects are located. Memory is managed here using a method called Scavenging, which is also known as Minor GC.

The Young space is divided into two halves, known as semi-spaces, which are used alternately.

- Newly created objects are allocated to one of the semi-spaces.
- When this semi-space becomes full, the scavenger is triggered.
- At this point, only the objects that are still referenced are moved to the other semi-space, and the remaining memory is cleared.
- Objects that survive two scavenger runs are promoted to the old space.

Note: During scavenging, as objects are moved to the other semi-space, this process inherently results in a compaction effect.
Therefore, there is no separate compaction phase, but even in the Young space, objects are moved to the front of the memory.

### Old space

The old space is divided into a pointer space and a data space. The pointer space stores objects that contain pointers to other objects. 
The data space, on the other hand, stores only data.(such as strings, boxed numbers, and arrays of unboxed doubles)

This region is where objects that have survived the Young space and have been retained for a long time are stored.
It is managed using the Mark-Sweep and Mark-Compact methods, which are collectively referred to as Major GC.

- The Mark-Sweep method marks the objects that are in use and then clears the objects that are unmarked (i.e., no longer in use). This process can lead to memory fragmentation.
- The Mark-Compact method was designed to address the fragmentation issue that can occur with Mark-Sweep. 
  It does so by moving the surviving objects to the front of the memory, thereby compacting the memory and eliminating gaps.

If you want to study this in more detail, it's a good idea to check out the link below. It provides a very detailed explanation.
[https://deepu.tech/memory-management-in-v8/](https://deepu.tech/memory-management-in-v8/)

## V8 Sandbox ( a.k.a. Ubercage )

In V8, there is a protection mechanism that prevents a successful exploit even if a vulnerability is triggered in the heap.
This mechanism is known as the 'Sandbox,' which is a separate, isolated space where objects that could be used for exploitation are collected.
This approach has evolved from storing direct addresses of objects in the heap to storing indices of tables instead.

Later on, WebAssembly will be used to bypass this Sandbox.

## Preferences

[Data and Object in V8](https://www.dashlane.com/blog/how-is-data-stored-in-v8-js-engine-memory) <br>
[Abstract of Maglev](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/) <br>
[Explanation about Garbage collection in korean](https://medium.com/hcleedev/web-javascript%EC%9D%98-garbage-collection-v8-%EC%97%94%EC%A7%84-9409c5be917c) <br>





