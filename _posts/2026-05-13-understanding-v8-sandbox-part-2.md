---
layout: post
read_time: true
show_date: true
title: "Understanding V8 Sandbox (Part 2)"
date: 2026-05-13
img: posts/20240925/v8_picture.png
tags: [V8, Sandbox]
category: Sandbox
author: adawn0106
description: "Understanding V8 Sandbox (Part 2)"
comments: true
---

This post is based on the following document: <br>
[High-Level Design Doc](https://docs.google.com/document/d/1FM4fQmIhEqPG8uGp5o9A-mnPB5BOeScZYpkHjo0KKA8/edit?tab=t.0) <br>

<br><br>

## Summeary

Objective: build a low-overhead, in-process sandbox for V8. <br>
Motivation: V8 bugs typically allow for the construction of unusually powerful and reliable exploits. <br>
Furthermore, these bugs are unlikely to be mitigated by memory safe languages or upcoming hardware-assisted security features such as MTE or CFI. <br>
As a result, V8 is especially attractive for real-world attackers. <br>
Design: The proposal assumes that an attacker can arbitrarily corrupt memory on the V8 heap, where JavaScript objects are located. <br>
This primitive can be constructed from typical V8 vulnerabilities. <br>
To protect other memory within the same process from corruption, and by extension to prevent the execution of arbitrary code, the V8 heap is moved into a pre-reserved region of virtual address space: the sandbox. <br>
Then, all memory accesses performed by V8 must either be restricted to the sandbox address space (e.g. by using offsets instead of raw pointers to reference objects) or be validated in some way (e.g. by using a pointer table indirection). <br>

In particular, full 64-bit pointers must be banned entirely from the V8 heap. <br>

<br>

Before talking about MTE and CFI, let’s first understand what MTE and CFI actually are.
Both are security mitigation techniques designed to make it harder for memory vulnerabilities to lead to actual code execution.

<br>

### MTE

First, MTE (Memory Tagging Extension) is a hardware feature available on ARM-based CPUs. It is a memory tagging extension mechanism. MTE attaches tags both to pointers and to memory regions, and whenever a program reads from or writes to memory, the CPU checks whether the two tags match.

For example, when an object is allocated on the heap, the memory region may receive tag A, and the pointer referencing that object will also receive tag A. Later, when the pointer is used to access memory, the CPU verifies whether the pointer tag and the memory tag are equal. If the tags do not match, the CPU treats it as an invalid memory access and raises an error.

<br>

The types of bugs MTE mainly tries to detect are usually UAF, BOF, OOB, and similar memory safety issues.

The tag itself is typically just a small integer value, and the allocator, runtime, or kernel usually selects it randomly or pseudo-randomly during memory allocation.

In other words, the tag value itself is unrelated to the object type.

ARM MTE tags are typically 4-bit values, meaning they range from 0 to 15.

<br>

For example, suppose the following code executes:

```C++
char* p = malloc(32);
```

The allocator would then:<br>

Allocate the memory region<br>
Set the tag of the memory granules to tag = 5<br>
Record tag = 5 in the upper bits of the returned pointer<br>

Later, when the memory is accessed, the CPU checks whether the tag values match, i.e., whether `5==5`.

<br>

Another important point is that although MTE stores tags per granule (16-byte units), if the allocated size is larger than 16 bytes, the allocator typically fills the entire allocation with the same tag value so that the tags remain consistent.

On the other hand, if the allocation size is smaller, for example 8 bytes, the granule size is still 16 bytes. As a result, two separate 8-byte objects A and B may share the same tag. This is referred to as a sub-granule overflow issue.

<br>

Tags are chosen randomly primarily to invalidate stale pointers.

For example:

```C++
p = malloc(...);   // tag 5
free(p);

q = malloc(...);   // same address reused
```

In this case, `q` may be allocated at the same address previously used by `p`.

If the new object reused tag 5 as well, then the old pointer `p` could still appear valid even though the allocator considers it freed. To prevent this, allocators typically assign a different tag during reallocation.

However, because the number of possible tag values is small, tag collisions can still occur by chance. For this reason, MTE is often considered a probabilistic mitigation technique.

<br>

For memory tags, MTE stores tags both inside the pointer itself and inside memory-side metadata.

For pointer tags:

ARM64 does not use the full 64-bit address space, so the upper bits can be utilized like this:

`| tag | virtual address |`

Memory tags, on the other hand, are stored in hidden metadata associated with memory.

<br>

There are also situations where MTE is not sufficient.

Case 1 is simple: if an OOB access occurs but the out-of-bounds region happens to have the same tag value, the access will still succeed because the tags match.

Case 2 is the sub-granule overflow issue mentioned earlier.

Since MTE assigns tags in 16-byte granules, anything occurring within the same granule shares the same tag. As a result, overflows occurring inside a single granule cannot be detected.

<br>

### CFI

CFI stands for Control Flow Integrity. It is a technique designed to protect the integrity of a program’s control flow.

Programs execute according to control flow operations such as function calls, returns, branches, and virtual function calls. CFI is designed to prevent attackers from abusing memory corruption to redirect execution flow to unintended locations.

<br>

Before explaining further, let’s first look at `fp();.`

Suppose we have:

```C++
void (*fp)();
```

Then doing:

```C++
fp = hello;
```

is equivalent to:

```C++
fp = &hello;
```

In other words, the address of the `hello` function is stored inside `fp`.

Therefore, when `fp` is called, the `hello` function executes.

<br>

Now let’s consider how attacks normally happen.

Suppose the following code exists:

```C++
class Animal {
public:
    virtual void speak() {
        printf("animal\n");
    }
};

void run(Animal* a) {
    a->speak();
}
```

Internally, a virtual call follows a flow similar to:

`vtable -> read function address -> jump`

If a vulnerability such as UAF exists, an attacker may corrupt the vtable pointer itself and redirect execution to an arbitrary address.

<br>

When CFI is applied, the compiler transforms:

`a->speak();`

into something conceptually similar to:

```C++
if (target in allowed_targets)
    jump(target);
else
    crash();
```

In other words, the indirect jump is checked to determine whether the destination is legitimate.

When applying CFI, the compiler generates metadata at compile time such as:

`allowed_set = {hello, bye}`

and performs checks like:

```C++
if (fp not in allowed_set)
    abort();
```

<br>

CFI generally comes in two forms.

First is Forward-edge CFI.

This protects:

virtual calls
function pointers
indirect jumps

and similar control flow transfers.

For example:

```C++
func_ptr();
obj->virtual_method();
```

are restricted so they can only call functions within an allowed set.

<br>

Next is Backward-edge CFI, which protects control flow returning backward.

Examples include:

function returns
return addresses stored on the stack

One common mechanism is the shadow stack.

When a function is called, the return address is stored both on the normal stack and on a separate protected stack called the shadow stack.

Then, when the function returns, the system compares the return address on the normal stack with the value stored in the shadow stack to ensure they match.

<br>

Although this explanation took a bit of a detour, the reason why MTE and CFI are considered insufficient for mitigating V8 vulnerabilities is the following:

MTE is strong at detecting invalid memory accesses, but as discussed earlier, it can still miss type confusion and data manipulation occurring within seemingly valid objects whose tags match.

CFI is strong at preventing control flow hijacking, but attackers do not always need to jump to invalid addresses. For example, attackers may perform data-only attacks by invoking legitimate functions within valid control flow while manipulating only data. Similarly, manipulation of JIT internal states is difficult to stop using CFI alone.

Additionally, because JIT engines generate code dynamically at runtime, defining what constitutes “legitimate control flow” is significantly harder than in ordinary static programs.

<br>

The sandbox design itself assumes that the V8 heap has already been compromised. Therefore, it reserves a large virtual memory region and places all JavaScript objects inside it so that memory outside the sandbox cannot be accessed.

Furthermore, if pointers inside the sandbox were stored as full 64-bit pointers, attackers could overwrite them with arbitrary addresses outside the sandbox. Instead, the sandbox represents addresses using offsets:

```C++
offset = 0x1234
real_address = sandbox_base + offset
```

By representing addresses as offsets, no matter what value an attacker writes, it becomes impossible to construct an address outside the sandbox range because the offset itself has a size limitation.

<br>

There are still cases where pointers to objects outside the sandbox must be used. However, instead of storing raw pointers directly, the system uses a structure like:

```
heap object
  ↓
index
  ↓
pointer table
  ↓
real pointer
```

This ensures that raw pointers do not exist directly inside the heap.

<br>

Finally, the reason raw pointers must not exist is, as discussed repeatedly above, because attackers could manipulate them to point to addresses outside the sandbox, ultimately enabling sandbox escape.

<br>

## Objective

Build an in-process sandbox for V8 to prevent an attacker who successfully exploited a V8 vulnerability, and thus is able to corrupt objects inside the V8 heap, from corrupting other memory in the process and thus from executing arbitrary code. In essence, this will turn arbitrary writes originating from V8 vulnerabilities into bounded writes. <br>
Preventing reads (direct or speculative) outside of the V8 heap is not a goal, however. The performance overhead should be minimal, with a rough target of around 1% overall on real-world workloads. This sandbox should eventually become a supported security boundary. <br>

Although this follows the same reasoning discussed earlier, the V8 Sandbox considers AAW (Arbitrary Address Write) significantly more critical than AAR (Arbitrary Address Read). <br>

While information disclosure alone can still enable attackers to perform actions such as address leaks, ASLR bypasses, and object layout discovery, it does not immediately lead to control flow hijacking. Once arbitrary write becomes possible, however, attackers may corrupt memory outside the sandbox and eventually achieve arbitrary code execution. <br>

For example, with only AAR, attackers may leak addresses or inspect memory layouts, but they generally cannot perform attacks such as RIP overwrite, vtable overwrite, JIT code corruption, or function pointer overwrite. <br>

Therefore, the primary goal of the sandbox is to prevent writes to memory outside the sandbox, effectively transforming arbitrary writes into bounded writes. <br>

## Motivation

Many V8 vulnerabilities exploited by real-world attackers are effectively 2nd order vulnerabilities: the root-cause is often a logic issue in one of the JIT compilers, which can then be exploited to generate vulnerable machine code (e.g. code that is missing a runtime safety check). <br>
The generated code can then in turn be exploited to cause memory corruption at runtime. This appears to be a somewhat natural problem of JIT compilers for dynamic languages, as one of their major purposes is to remove (redundant) runtime checks that would otherwise be performed by the interpreter. <br>
As an example, consider the case of a [JIT compiler incorrect side effect modeling vulnerability](http://phrack.org/issues/70/9.html#article): here, the compiler will model the side effects of an operation incorrectly and can then be tricked into emitting machine code that is lacking a runtime type check after such an operation (because the compiler believes that the object could not have changed its type during the operation).<br>

As such, the emitted machine code is now vulnerable to a type confusion, and the attacker can exploit that to cause (fairly arbitrary) memory corruption at runtime.

These types of issues are uniquely attractive for attackers for a number of reasons:

The attacker has a great amount of control over the memory corruption primitive and can often turn these bugs into highly reliable and fast exploits

Memory safe languages will not protect from these issues as they are fundamentally logic bugs

Due to CPU side-channels and the potency of V8 vulnerabilities, upcoming hardware security features such as memory tagging will likely be bypassable most of the time

Due to the nature of these vulnerabilities, and their uniqueness to JavaScript engines, it seems desirable to build a custom sandboxing mechanism for V8. <br>

Originally, I thought that during JIT compilation, runtime checks or guards were simply used temporarily or tested during execution. But now I understand that the generated machine code itself actually contains those runtime check instructions.

<br>

In other words, a JIT compiler generates machine code that already includes safety checks. However, because the JIT itself is expensive and performance-sensitive, if there is a part where it assumes something like:

<br>

“This condition will probably always be true.”

<br>

then it may optimize the code by removing some checks.

<br>

Usually, the generated machine code follows a pattern like:

<br>

`fast path + runtime guard`

<br>

So if a bug exists in the JIT compiler, it may fail to generate checks that were originally supposed to be there.

“Incorrect side effect modeling” refers to how the compiler reasons about and models what kinds of state changes (side effects) can occur when a particular operation executes.

For example, it concerns questions like:

“How can operation A modify memory, objects, or type state?”

To think about it more intuitively, in order to optimize code, a JIT compiler may assume that when accessing:

`obj.x`

the object `obj` will continue to have the same type.

However, in the middle of execution, a function call such as:

`foo(obj)`

might actually trigger object structure changes, map transitions, prototype modifications, type changes, and so on. These are considered side effects.

If the compiler models `foo(obj)` as something that does not change the type of `obj`, but in reality the type does change, then that becomes incorrect side effect modeling.

As a result, the JIT may conclude that additional type checks are unnecessary and eliminate them, because it assumes the type of `obj` cannot change.

Ultimately, this can lead to a wrong-type object being accessed in an invalid way, which is essentially how type confusion occurs.

A side channel refers to a path that was not intended as the official data flow.

For example, in a normal program, the intended channel may only reveal whether a password is correct or not. But if an attacker can infer the password by observing execution time, cache states, power consumption, branch predictor state, and so on, then that becomes a side-channel attack.

## Attack model


This proposal assumes that an attacker is capable of repeatedly performing arbitrary reads and writes inside the V8 heap as well as potentially performing reads outside of the V8 heap, for example due to speculative side-channel attacks. 
This reflects common initial exploitation primitives gained from many V8 vulnerabilities, such as vulnerabilities in the JIT compilers, the runtime, or the garbage collector.

The ability to corrupt memory outside of the V8 heap is then considered to be an escape from this sandbox. This definition also covers arbitrary code execution.

The key point discussed here is the threat model of Ubercage (V8 Sandbox).

The sandbox assumes that an attacker is already capable of performing arbitrary read/write operations within the V8 heap. However, once the attacker is able to corrupt memory outside of the heap, it is considered a sandbox escape.

This definition also includes arbitrary code execution.


## Design

Since early 2020, V8 has implemented pointer compression in its heaps. With pointer compression, every reference from an object in the V8 heap to another object in the V8 heap (“on-heap”) becomes a 32 bit offset from the base of the heap, leaving only a few objects with raw pointers to objects outside the v8 heap (“off-heap”). Compressed pointers are then only valid inside a 4GB virtual memory region, referred to as the pointer compression cage. Pointer compression can be visualized with an instance of an ArrayBuffer in memory. The following shows the in-memory layout of a hypothetical ArrayBuffer object without pointer compression:


![sandbox](/assets/img/posts/20260527/sandbox-2_1.png)

The difference between an on-heap pointer and an off-heap pointer depends on whether the target being referenced resides inside the V8 heap or outside of it.

In the case of a 64-bit on-heap pointer, it points to an object located inside the GC-managed heap controlled by V8.

On the other hand, a 64-bit off-heap pointer points to native memory located outside the V8 heap.




With pointer compression enabled, the same ArrayBuffer object would be stored as shown next:

![sandbox](/assets/img/posts/20260527/sandbox-2_2.png)

Here, all on-heap pointers were converted to 32-bit compressed pointers. In this example, the heap base would be 0x250700000000 and so for example the compressed Map pointer (0x08281181) would be interpreted as the absolute pointer 0x250708281181.

The majority of V8 vulnerabilities can be exploited to corrupt memory (only) in the V8 heap (out-of-bounds accesses, type confusions, et al). With pointer compression enabled, an attacker with the ability to corrupt data in the V8 heap gains no additional capabilities by corrupting a compressed pointer. Instead, to reach outside the V8 heap, the attacker targets one of the remaining raw pointers in the heap (typically an ArrayBuffer or TypedArray backing store pointer). The fundamental idea behind this project is to protect (“sandboxify”) all remaining raw pointers in a way that prevents their abuse by an attacker. On a high level, this is achieved in the following way:

A large (for example 1TB) region of virtual address space - the sandbox - is reserved during initialization of V8. This region contains the pointer compression cage, and so all V8 heaps, as well as ArrayBuffer backing stores and similar objects. 
All objects inside the sandbox, but outside of V8 heaps, are addressed using fixed-size offsets (e.g. 40-bit offsets in the case of a 1TB sandbox) instead of raw pointers.
All remaining off-heap objects must be referenced through a pointer table, which contains the pointer to the object together with type information to prevent type confusion attacks. Entries in this table are then referenced from objects in the v8 heap through indices.

Compared to the past, the memory layout used to look like this:

```

[V8 Heap]
  JSObject
  Array
  Map
  ...

[Outside Heap]
  ArrayBuffer backing store
  Wasm memory
  native objects
  C++ structures

```

Only the V8 heap was GC-managed, while the native memory regions actually useful for exploitation existed outside the heap. As a result, attackers could use type confusion bugs to corrupt backing store pointers and ultimately achieve arbitrary native memory access.

However, the architecture has now changed into something more like this:

```

[Huge Sandbox Region]
 ├── V8 heap
 ├── pointer compression cage
 ├── ArrayBuffer backing stores
 ├── Wasm memory
 ├── various sandboxed native allocations
 └── other engine-managed memory

```

Most memory regions directly managed by the JS engine are now placed inside one huge sandboxed virtual address range.

Things such as:

ArrayBuffer backing stores
Wasm memory
external string data

still exist outside the GC heap, but they are now located inside the sandbox, meaning they are only accessible via sandbox-relative offsets.

Because of this, in modern V8, even if an attacker achieves:

heap object corruption
backing store corruption
typed array corruption

the accessible range is fundamentally restricted to memory inside the sandbox.

Unlike before, a JS bug no longer directly leads to arbitrary process memory R/W. Instead, attackers typically need to proceed in stages:

Obtain arbitrary R/W inside the sandbox
Gain an additional sandbox escape primitive
Achieve process-wide native memory access

With this, the ArrayBuffer object from above would become:

![sandbox](/assets/img/posts/20260527/sandbox-2_3.png)


ArrayBufferExtension is an auxiliary native-side (C++) structure internally associated with the ArrayBuffer object in V8. More precisely, it is not a JS heap object itself, but rather metadata used to manage external (native) state related to an ArrayBuffer.

The older design looked roughly like this:


```

JSArrayBuffer (heap object)
 ├── backing_store pointer  ---> raw native memory
 ├── byte_length
 ├── flags
 └── extension pointer ----> ArrayBufferExtension (native)
                                 ├── accounting info
                                 ├── GC interaction state
                                 ├── embedder fields
                                 ├── linked list nodes
                                 └── allocator metadata

```

The primary purposes of ArrayBufferExtension include:

```

backing store lifetime management
connecting GC with external memory accounting
embedder/native-side bookkeeping
ArrayBuffer tracking
detach state management
cleanup callback management

```

In particular, because the ArrayBuffer backing store resides outside the GC heap (in native memory), the V8 garbage collector could not directly manage it through ordinary mark-and-sweep mechanisms alone.

As a result, V8 used extension structures like this to track:

how much external memory is currently being used
when the memory should be freed
which isolate the memory belongs to

Previously, this structure existed as a native object outside the V8 heap, which also made it a potential exploitation target. However, with the introduction of the V8 sandbox, it is now located inside the sandbox region, adding an additional step for attackers.

Here, the raw, off-heap backing storage pointer (purple) has been replaced with a 40-bit offset from the base of the sandbox (the offset is 0x45c00, shifted to the left by 24 to guarantee that the top bits are zero). On the other hand, the raw pointer to the ArrayBufferExtension object (orange) has been replaced with a 32-bit index into a pointer table. In this example, the size (magenta) remains unchanged, but would be verified on access to be smaller than the maximum allowed size of an ArrayBuffer. Alternatively, it could also be stored shifted to the left as well.
Attackers are now assumed to be able to corrupt memory inside the sandbox arbitrarily and from multiple threads, and will now require an additional vulnerability to corrupt memory outside of it, and thus to execute arbitrary code. However, the attack surface of the sandbox will likely be significantly less complex than V8 itself due to the relatively low complexity of the embedder <-> V8 interface, and bugs in the sandbox appear to mostly be  “classic” memory corruption bugs as opposed to bugs in V8. For these reasons, the sandbox is assumed to be an easier-to-defend security boundary than the V8 VM.

Finally, it is worth noting that, although not an explicit goal, this design also prevents an attacker from directly (but not speculatively) reading, not just writing, data outside of the sandbox.

The remainder of this section discusses the central sandboxing mechanisms in some more detail, then concludes with a brief summary of the design.

last update : 2026/05/27
























