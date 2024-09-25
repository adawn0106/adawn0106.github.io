---
layout: post
read_time: true
show_date: true
title: " Fundamental Knowledge of JavaScript (JS object structure, Allocation folding, TypedArray)"
date: 2024-09-26
img: posts/20240925/v8_picture.png
tags: [V8, prior knowledge, JS object structure , Allocation Folding , TypedArray]
category: prior knowledge
author: adawn0106
description: "Before explaining the V8 exploit, I would like to briefly introduce some prior knowledge you need to understand, including JS object structure , Allocation Folding , TypedArray."
comments: true
tocs: yes
---

The following content was originally written for our team blog.  
At the time, I didn't have a personal GitHub blog, so I'm posting it now after setting one up.  
This research was not conducted alone but was a joint effort between two people.  
To gain more experience, I plan to redo the entire process by myself from the beginning and will add any new insights I discover along the way.





# [JavaScript] Background Knowledge - Java Object Structure, Allocation Folding




## JS object structure

<center> <img src="https://github.com/user-attachments/assets/cb1bd7c2-38f4-47e3-aef5-6e5b736272ef" /> </center>

A JavaScript object is a data structure composed of key-value pairs. The V8 engine handles metadata and actual data differently depending on the structure and properties of the object.

### Map

The value in the first field is the map, also known as the hidden class. The hidden class contains metadata about the property names, types, and the position of each property within the object.
Let's take a closer look at hidden classes.

<center> <img src="https://github.com/user-attachments/assets/8b9b8fc2-6988-496a-b504-39417d8419a6" /> </center>


When an object is created, a map (or hidden class) is generated. Each time new properties are added to the object, a new map is also created.

Additionally, the map (hidden class) points to both the previous map and the next map, allowing it to navigate to either one.

<center> <img src="https://github.com/user-attachments/assets/82974c1a-e29d-473a-a78c-83e18962f9c2" /> </center>


You can see that it's structured this way.

If you have any further questions or want to learn more, please refer to the provided references.

### Properties

The second field is the Properties, which are composed of key-value pairs.

For example, it would look like this.


```jsx
let person = {
    name: "John",
    age: 30,
    isEmployed: true
};

```

If a person object is defined like this, for example:

name → Property, john → Value

age → Property, 30 → Value

isEmployed → Property, true → Value

### Elements

The third field is the Elements. This refers to the space where the array elements of a JavaScript object are stored.

Unlike regular objects, array objects store data using sequential numeric indices.

Elements are responsible for storing and managing these array elements.

### In-Object Properties

The fourth field, in-Object Properties, refers to properties that are stored directly within the object itself.

They are used for fast access, but if the number of properties exceeds a certain limit, they can no longer be used.

To help with understanding, I've brought in the Debugprint of the `x` object and the `a` array, which we'll use later in the PoC (Proof of Concept).

It might be helpful to compare this with the information provided above.

<center> <img src="https://github.com/user-attachments/assets/a4bec213-f773-4842-8e89-f9222bf67d20" /> </center>

(a Array’s DebugPrint)

<center> <img src="https://github.com/user-attachments/assets/7679ef28-4c9d-44bb-bb64-d9fac04a37bc" /> </center>

(x object’s DebugPrint)

## Allocation folding

Allocation Folding is a memory allocation technique that reduces unnecessary allocation attempts by allocating memory in bulk, rather than making multiple allocations.

<center> <img src="https://github.com/user-attachments/assets/bb16d73f-8ffd-4b9d-a259-f4f3d9518615" /> </center>


For example, when allocating object of Class B and Array a, as shown in the diagram, each would be allocated to memory once, resulting in a total of two allocations. However, if the allocation folding technique is used, the combined size of Class B and Array a (size(x+y)) is allocated as a single block of memory, which is then shared by both Array a and Class B.
 

## JS Array

TypedArray is a JavaScript object provided to handle arrays of a specific type.

In other words, all elements in the array have the same data type (e.g., int8, Uint8, int16, etc.).

<center> <img src="https://github.com/user-attachments/assets/ce184978-674a-4dce-afb6-a0bc99215d8f" /> </center>

(TypedArray Types)

The reason for using TypedArray is that, since all elements have the same type, they have a fixed size, which makes them advantageous for performance optimization.

---

Starting with Packed Array: 

it is an array where all elements are defined in a contiguous block of memory, with no gaps between the elements. 

---

Additionally, there are concepts like Packed Array and Holey Array. 

I’ll explain Packed Array firstly. It is an array in which all elements are defined sequentially, without any gaps.

This means that every index in the array is fully occupied.

This type of array is stored contiguously in memory, which allows for faster access speeds.

Next is the Holey Array. In this type of array, there are undefined elements in the middle (e.g., `undefined` or certain indices without values).

In this case, the array is stored non-contiguously in memory, which leads to reduced performance.

To illustrate with code:

```jsx
let array = [0,1,2]; 
array[3] = 3.1
array[3] = 4
array[4] = "five"
array[6] = 6
```

The first line refers to `PACKED_SMI_ELEMENTS`. Here, "Smi" stands for "Small Integer," an internal data type in the V8 engine representing small integers of 31-bit (on x32) or 63-bit (on x64) size. The array is labeled as "PACKED" because all its elements are fully populated.

The second line also refers to a "PACKED" array, but it's labeled as `PACKED_DOUBLE_ELEMENTS`. Originally, only SMIs were present, but since a float value has been added, both Smi and Float now coexist within the same array.

The third line is still labeled as `PACKED_DOUBLE_ELEMENTS`. This is because, even if the last element becomes an SMI value again, the array doesn't revert to `PACKED_SMI_ELEMENTS`. Once an array's type has changed, it doesn't go back to its previous state.

 
That's why it remains `PACKED_DOUBLE_ELEMENTS`.

The fourth line is labeled as `PACKED_ELEMENTS`. Although the indices are still fully occupied, the `DOUBLE` designation disappears because there are now three or more different types of elements in the array.

The fifth line is labeled as `HOLEY_ELEMENTS`. This is because index 6 was added suddenly, but the previous index, `array[5]`, doesn't have a value. As a result, a gap was created, making the array "holey."

This will be important later when we analyze exploits, so it's a good idea to keep this in mind.

Reference 

Js Object Structure: [https://devkly.com/nodejs/javascript-array/](https://devkly.com/nodejs/javascript-array/)

V8 Hiddenclass : [https://v8.dev/docs/hidden-classes](https://v8.dev/docs/hidden-classes)

v8 TypedArray:[https://v8docs.nodesource.com/node-7.10/d5/dbb/classv8_1_1_typed_array.html](https://v8docs.nodesource.com/node-7.10/d5/dbb/classv8_1_1_typed_array.html)
