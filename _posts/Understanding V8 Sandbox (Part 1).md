---
layout: post
read_time: true
show_date: true
title: "Understanding V8 Sandbox (Part 1)"
date: 2025-05-05
img: posts/20240925/v8_picture.png
tags: [V8, Sandbox]
category: Sandbox
author: adawn0106
description: "Understanding V8 Sandbox (Part 1)"
comments: true

---

It’s been a while since I last wrote a blog post because I’ve been busy lately, but I’m planning to get back into it—both as a way to organize my thoughts and to keep up with my personal studies.<br>
I’d like to focus mainly on V8, but I might occasionally post some random or less relevant content as well. <br>

Anyway, this time I’ll be covering the sandbox. I’ve been meaning to study it for a while, so I decided to revisit some documentation I already knew and go through it more carefully.  <br>
This post is based on the following document:
https://docs.google.com/document/d/10ZVrH2m_cbsjhZmjnWd4K5jpEHWCLourq2dulwN8elI/edit?tab=t.0 <br>

<br><br><br>

First, I’ll go over the key terms related to the V8 sandbox.

## General 
<br>


### Sandbox 

Summary: A region of virtual address space (typically 1TB) containing all untrusted V8 objects. <br>
Implementation: This is implemented as a large, contiguous virtual address space reservation using the appropriate operating system primitives. <br>
Address space reservations are cheap on all modern OSes.  <br>
Security Properties: It is assumed that an attacker can arbitrarily read and write memory inside the sandbox address space due to a typical security bug in V8.  <br>
The mechanisms described in this document, in particular the various pointer types, then attempt to prevent an attacker from corrupting other memory inside the process running V8. <br>

Based on the content, although I haven’t gone through the detailed sandbox design yet, it assumes that an attacker can already arbitrarily read and write memory inside the sandbox due to common vulnerabilities. <br>
In other words, it starts from the premise that the security within the sandbox may already be compromised. <br>
Rather than trying to fully protect the inside of the sandbox, the design focuses on preventing the attacker from escaping to the outside. <br>
I used to think of the sandbox as something like a trust zone, but through recent study, I’ve realized that it’s not quite the same. <br>


## Objects 
<br>

### TrustedObject

Summary: A V8 HeapObject containing sensitive data or code, located outside of the sandbox. <br>
Implementation: These are regular HeapObjects with a special instance type and which are allocated in one of the trusted heap spaces, located outside of the sandbox. <br>
Security Properties: As these objects live outside of the sandbox, it can be assumed that they have not been manipulated by an attacker.  <br>
As such, they can safely be read from, but one must be particularly careful when writing to them to avoid any memory corruption in trusted space. <br>

Here, the term instance type refers to objects that are defined as trusted at the type level and are also located within the trusted space. <br>
As for what constitutes sensitive data or code, I think it would be better to go through the detailed documentation and organize that separately later. <br>

The trusted heap itself seems to be a large region, but internally it is divided into multiple spaces. <br>
Lastly, the document mentions that reading is safe, but writing operations must be handled carefully to avoid memory corruption in the trusted space. <br>
This is because there are many dangerous possibilities, such as modifying JIT-compiled code directly or tampering with pointer tables. <br>


### ExposedTrustedObject

Summary: A trusted object directly exposed to objects inside the sandbox.  <br>
Implementation: These objects own an entry in one of the trusted pointer tables and therefore can be referenced from inside the sandbox in a memory-safe way via a trusted pointer (see below).  <br>
Security Properties: Same as TrustedObject.  <br>

ExposedTrustedObject also lives in the trusted heap, but unlike a regular TrustedObject, it can be accessed from within the sandbox.   <br>
That said, this access is not done through raw pointers, but rather through the trusted pointer table.  <br>

## pointers  
<br>

### Compressed Pointer

Summary: A pointer that is guaranteed to point into V8’s 4GB heap area, inside the sandbox. <br>
Implementation: These are stored as 32-bit offsets from the start of the heap area. They were originally developed to reduce V8’s memory footprint. <br>
Security Properties: As this pointer always points into the sandbox, it can always safely be written to, but when reading from it, it must be assumed that the data has been corrupted by an attacker. <br>

The reason it is limited to 4GB is that the value is represented using a 32-bit offset. <br>
The reason write operations are considered relatively safer than read operations is based on the assumption that memory inside the sandbox can already be arbitrarily modified by an attacker. <br>
Therefore, additional writes do not significantly change the situation from that assumption. <br>
In other words, writes do not directly affect memory outside the sandbox, since compressed pointers are restricted to referencing only addresses within the sandbox. <br>

On the other hand, while an attacker reading data may simply result in information disclosure, the real issue arises when V8 performs a read.  <br>
If V8 reads corrupted or attacker-controlled data and trusts it, it may operate based on incorrect values, which can ultimately lead to effects that extend beyond the sandbox.  <br>

### Uncompressed/Full Pointer

Summary: A full (64-bit) pointer to a V8 HeapObject.  <br>
Implementation: These are regular (“raw”) pointers to HeapObjects.  <br>
 They appear for example after decompressing a compressed pointer when loading it into a register or onto the stack, or as fields in objects located outside of the sandbox, for example tracking data structures used by the GC or the compilers, or objects created by the Embedder. <br>
Security Properties: As these are raw pointers, they must not be corrupted by an attacker and so must only be used outside of the sandbox. As they also point into the sandbox, the same considerations as for Compressed Pointers apply, namely that it must be assumed that what they point to has been corrupted. <br>

Pointers located outside the sandbox do not need to use compressed pointers, so full pointers are used instead. <br>
Since these pointers reside outside the sandbox, it is assumed that they cannot be modified by an attacker.  <br>
However, because they point to objects inside the sandbox, the values they reference may still be corrupted. <br>
As mentioned earlier, the sandbox is designed under the assumption that memory within it may already be compromised, so the data being pointed to could potentially be attacker-controlled. <br>

### Sandboxed Pointer

Summary: A pointer that is guaranteed to point into the sandbox.  <br>
Implementation: A sandboxed pointer is a 40-bit offset (when the sandbox is 1TB large) that is added to the base of the sandbox when loaded. When the sandbox is disabled, they are regular full pointers. <br>
Security Properties: As this pointer always points into the sandbox, it can always safely be written to, but when reading from it, it must be assumed that the data has been corrupted by an attacker. <br>

A sandboxed pointer is a pointer that is guaranteed to always point inside the sandbox.  <br>
The difference between a sandboxed pointer and a compressed pointer is that a compressed pointer refers to the V8 heap, while a sandboxed pointer refers to the entire sandbox space. <br>
The address is computed in the form of sandbox_base + offset. <br>

### Protected Pointer

Summary: A pointer that cannot be modified by an attacker. Only used between TrustedObjects. <br>
Implementation: These are implemented as compressed pointers but use the trusted space base instead of the in-sandbox heap base.  <br>
 It’s therefore guaranteed that they will always point into trusted space. As the pointer itself is also located in trusted space, it cannot directly be modified by an attacker. When the sandbox is disabled, they are regular tagged pointers. <br>
Security Properties: Neither the pointer itself nor the pointed-to object can be modified by an attacker as both live in trusted space. These pointers therefore have the strongest security guarantees. <br>

A protected pointer is a pointer that cannot be modified by an attacker and is only used between TrustedObjects. <br>
The reason it is considered unmodifiable is that it exists between objects located outside the sandbox. <br>
Since TrustedObjects themselves reside in trusted space, the pointer is also placed in a location that the attacker cannot directly access.  <br>
Additionally, this pointer is based on the trusted space base. <br>

### External Pointer

Summary: A pointer to an object external to V8, outside of the sandbox and not managed by V8’s GC. <br>
Implementation: These are implemented as indices into the ExternalPointerTable (EPT), which is located outside of the sandbox. The EPT performs type checks on every access to a pointer. <br>
 Conceptually, they are very similar to file descriptors in Unix. When the sandbox is disabled, they are regular full pointers. <br>
Security Properties: The pointer will point to a valid object of the expected type. However, an attacker can swap these pointers as long as the type of the referenced object is the same. <br>

External pointers do not directly reference objects outside of V8. Instead, they are managed through an indirection mechanism using the ExternalPointerTable (i.e., indices).  <br>
While type checks ensure safety, pointer swap attacks are still possible between objects of the same type. <br>
For example, it is possible to swap two objects A and B if they share the same type. <br>

### Indirect Pointer
Summary: A pointer to a V8 HeapObject that goes through a pointer table indirection on access.  <br>
Implementation: The “under-the-hood” implementation of Trusted Pointers and Code Pointers (see below) when the sandbox is enabled. Similar to External Pointers, these are indices into a pointer table and perform type checks on access. These pointers are only available when the sandbox is enabled.  <br>
Security Properties: See Trusted Pointer.  <br>

Since V8 HeapObjects reside inside the sandbox, it might seem unnecessary to introduce an additional level of indirection.  <br>
However, if they were accessed directly, an attacker could potentially modify the pointer values. <br>
To mitigate this, the design uses a pointer table and performs an additional type check when retrieving the object, providing an extra layer of safety. <br>

### Trusted Pointer
Summary: A pointer that is guaranteed to point to a valid TrustedObject in trusted space.   <br>
Implementation: These are implemented as Indirect Pointers using the TrustedPointerTable (TPT) when the sandbox is enabled. Otherwise they are regular tagged pointers.  <br>
Security Properties: The pointer will point to a TrustedObject in trusted space of the expected type. However, as with External Pointers an attacker can perform “pointer swap attacks” so it is not guaranteed that the pointer still points to the “original” object.  <br>

The difference between a Trusted Pointer and a Protected Pointer is that a Trusted Pointer points to a TrustedObject, but since it is accessed indirectly through a pointer table, it is not completely secure and is susceptible to pointer swap attacks.  <br>
In contrast, a Protected Pointer resides entirely within trusted space, so it cannot be modified by an attacker and therefore provides stronger security guarantees. <br>
Pointers do not exist independently; they are stored as fields within objects. The security level of a pointer is determined by the location of the object that contains it, whether it resides inside the sandbox or in trusted space. <br>

### Code Pointer
Summary: A special kind of Trusted Pointer that always points to a Code object. <br>
Implementation: Similar to Trusted Pointers, but these use the CodePointerTable (CPT) instead of the TPT. They are essentially a performance optimization as the CPT also directly points to the code’s entrypoint. <br>
Security Properties: Same as Trusted Pointers, but will always point to a Code object. <br>

A Code Pointer refers to a Code object generated by JIT compilation, and unlike a regular Trusted Pointer, it is optimized to directly return the entrypoint of the code.   <br>
Pointer swapping between Code objects is possible within the same CodeKind.   <br>
Since it directly affects which code gets executed, it is a pointer that has an impact on control flow.  <br>














