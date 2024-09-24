---
layout: post
read_time: true
show_date: true
title: "Fundamental Knowledge of V8 engine (V8, Pipeline, Garbage collection, Ubercage)"
date: 2024-09-25
img: posts/20240925/v8_picture.png
tags: [V8, prior knowledge, pipeline, Garbage collection, Ubercage]
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

## What is V8?

Chrome V8 is an open-source JavaScript engine used in the Google Chrome web browser and the Node.js server environment.  
V8 is designed to efficiently execute JavaScript code, aiming for fast performance and optimization.  
Here is a table showing the global web browser market share, and as of December 2023, we can see that the Chrome browser holds a staggering 64.7% share.

The engine used by the Chrome browser, which I'm currently introducing, is the V8 engine.

<p style="text-align: center;">
    <img src="https://github.com/user-attachments/assets/0a823c86-6e55-41b4-a956-28a7ab7cd5b5" alt="V8 engine" />
</p>

*Source: [Browser Market Share Worldwide](https://gs.statcounter.com/browser-market-share#monthly-202312-202312-bar)*

It seems Chrome might need to make more of an effort so that searches for 'V8 engine' immediately bring up the Chrome V8 engine instead.

### V8 Engine Components (Pipeline)

The V8 engine is composed of various components. To make it easier to understand, I have attached two images below.

The components of V8 are as follows:

- Parser
- Ignition
- Sparkplug
- Maglev
- TurboFan

<p style="text-align: center;">
    <img src="https://github.com/user-attachments/assets/88795d10-86c5-45f2-9bc8-0ad99580b759" alt="V8 Pipeline" />
</p>
*Source: [V8_pipeline](https://medium.com/dailyjs/understanding-v8s-bytecode-317d46c94775)*

<p style="text-align: center;">
    <img src="https://github.com/user-attachments/assets/6aaf0bad-08c3-4520-966f-7f8f8f6ed823" alt="Pipeline with Maglev" />
</p>
*Source: [Pipeline with Maglev](https://docs.google.com/document/d/13CwgSL4yawxuYg3iNlM-4ZPCB8RgJya6b8H_E2F-Aek/edit#heading=h.dmhxljs5hbh)*

I will briefly explain each component.

- **Parser**: Converts the JavaScript source code to an AST (Abstract Syntax Tree) structure.

  You can view it easily through the site below:

  [AST Explorer](https://astexplorer.net/)

<p style="text-align: center;">
    <img src="https://github.com/user-attachments/assets/1544592a-d3c2-45ff-ab6d-97d689cfa2a6" alt="AST Example" />
</p>

<p style="text-align: center;">
    <img src="https://github.com/user-attachments/assets/d4214fee-b9f1-4543-a9b5-bec505bf1d1d" alt="AST Representation" />
</p>

- **Ignition**: As V8's interpreter, Ignition converts the parsed AST structure into bytecode.  
  It compiles and generates bytecode directly without first compiling the code into machine code, which allows it to be extremely fast.  
  Ignition also collects profiling information for later optimization.

  This image shows a portion of the bytecode:

<p style="text-align: center;">
    <img src="https://github.com/user-attachments/assets/e5b646c6-259a-40cd-bcc5-0e83f93f5841" alt="Bytecode Example" />
</p>

Based on the profiling information collected by Ignition, V8 determines the level of optimization needed using a JIT compiler. There are three types of compilers involved in this process:

- **Sparkplug**: A JIT compiler that performs fast and simple optimizations.
- **Maglev**: A JIT compiler that performs mid-tier optimizations by analyzing the source code.
- **TurboFan**: V8's most powerful optimizing JIT compiler, which conducts dynamic analysis followed by optimization and compilation.

Initially, only Ignition and TurboFan were used, but due to a gap in performance, Sparkplug was developed. Later, Maglev was added to fill the gap between Sparkplug and TurboFan.

## Garbage Collection of V8

V8's Garbage Collection differs from Java's in some ways, though the core concepts are similar.

V8's heap is divided into regions like the **Young space** and **Old space**, and each manages memory differently.

### Young Space

The **Young space** manages newly created objects. Memory here is managed using **Scavenging**, also known as **Minor GC**.

### Old Space

The **Old space** holds objects that have survived in the Young space for a long time. It uses the **Mark-Sweep** and **Mark-Compact** methods for memory management, collectively referred to as **Major GC**.

For a more detailed explanation, check out the following link:

[Memory Management in V8](https://deepu.tech/memory-management-in-v8/)

## V8 Sandbox (a.k.a. Ubercage)

V8 has a protection mechanism called the **Sandbox**, which prevents successful exploits by isolating objects that could be used for exploitation.  
Rather than storing direct addresses of objects, V8 stores indices of tables to protect against exploits.

## References

- [Data and Object in V8](https://www.dashlane.com/blog/how-is-data-stored-in-v8-js-engine-memory)
- [Abstract of Maglev](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/)
- [Explanation about Garbage Collection in Korean](https://medium.com/hcleedev/web-javascript%EC%9D%98-garbage-collection-v8-%EC%97%94%EC%A7%84-9409c5be917c)
