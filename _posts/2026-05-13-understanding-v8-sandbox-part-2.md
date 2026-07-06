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

## Summary

- Objective: build a low-overhead, in-process sandbox for V8.
- Motivation: V8 bugs typically allow for the construction of unusually powerful and reliable exploits.
- Furthermore, these bugs are unlikely to be mitigated by memory safe languages or upcoming hardware-assisted security features such as MTE or CFI.
- As a result, V8 is especially attractive for real-world attackers.
- Design: The proposal assumes that an attacker can arbitrarily corrupt memory on the V8 heap, where JavaScript objects are located.
- This primitive can be constructed from typical V8 vulnerabilities.
- To protect other memory within the same process from corruption, and by extension to prevent the execution of arbitrary code, the V8 heap is moved into a pre-reserved region of virtual address space: the sandbox.
- Then, all memory accesses performed by V8 must either be restricted to the sandbox address space (e.g. by using offsets instead of raw pointers to reference objects) or be validated in some way (e.g. by using a pointer table indirection).

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

The allocator would then:

- Allocate the memory region.
- Set the tag of the memory granules to tag = 5.
- Record tag = 5 in the upper bits of the returned pointer.

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

- Case 1 is simple: if an OOB access occurs but the out-of-bounds region happens to have the same tag value, the access will still succeed because the tags match.
- Case 2 is the sub-granule overflow issue mentioned earlier.

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

- virtual calls
- function pointers
- indirect jumps

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

- function returns
- return addresses stored on the stack

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

- The attacker has a great amount of control over the memory corruption primitive and can often turn these bugs into highly reliable and fast exploits.
- Memory safe languages will not protect from these issues as they are fundamentally logic bugs.
- Due to CPU side-channels and the potency of V8 vulnerabilities, upcoming hardware security features such as memory tagging will likely be bypassable most of the time.
- Due to the nature of these vulnerabilities, and their uniqueness to JavaScript engines, it seems desirable to build a custom sandboxing mechanism for V8.

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

- A large (for example 1TB) region of virtual address space - the sandbox - is reserved during initialization of V8. This region contains the pointer compression cage, and so all V8 heaps, as well as ArrayBuffer backing stores and similar objects.
- All objects inside the sandbox, but outside of V8 heaps, are addressed using fixed-size offsets (e.g. 40-bit offsets in the case of a 1TB sandbox) instead of raw pointers.
- All remaining off-heap objects must be referenced through a pointer table, which contains the pointer to the object together with type information to prevent type confusion attacks. Entries in this table are then referenced from objects in the v8 heap through indices.

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

- ArrayBuffer backing stores
- Wasm memory
- external string data

still exist outside the GC heap, but they are now located inside the sandbox, meaning they are only accessible via sandbox-relative offsets.

Because of this, in modern V8, even if an attacker achieves:

- heap object corruption
- backing store corruption
- typed array corruption

the accessible range is fundamentally restricted to memory inside the sandbox.

Unlike before, a JS bug no longer directly leads to arbitrary process memory R/W. Instead, attackers typically need to proceed in stages:

1. Obtain arbitrary R/W inside the sandbox.
2. Gain an additional sandbox escape primitive.
3. Achieve process-wide native memory access.

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

- backing store lifetime management
- connecting GC with external memory accounting
- embedder/native-side bookkeeping
- ArrayBuffer tracking
- detach state management
- cleanup callback management

In particular, because the ArrayBuffer backing store resides outside the GC heap (in native memory), the V8 garbage collector could not directly manage it through ordinary mark-and-sweep mechanisms alone.

As a result, V8 used extension structures like this to track:

- how much external memory is currently being used
- when the memory should be freed
- which isolate the memory belongs to

Previously, this structure existed as a native object outside the V8 heap, which also made it a potential exploitation target. However, with the introduction of the V8 sandbox, it is now located inside the sandbox region, adding an additional step for attackers.

Here, the raw, off-heap backing storage pointer (purple) has been replaced with a 40-bit offset from the base of the sandbox (the offset is 0x45c00, shifted to the left by 24 to guarantee that the top bits are zero). On the other hand, the raw pointer to the ArrayBufferExtension object (orange) has been replaced with a 32-bit index into a pointer table. In this example, the size (magenta) remains unchanged, but would be verified on access to be smaller than the maximum allowed size of an ArrayBuffer. Alternatively, it could also be stored shifted to the left as well.

Attackers are now assumed to be able to corrupt memory inside the sandbox arbitrarily and from multiple threads, and will now require an additional vulnerability to corrupt memory outside of it, and thus to execute arbitrary code. However, the attack surface of the sandbox will likely be significantly less complex than V8 itself due to the relatively low complexity of the embedder <-> V8 interface, and bugs in the sandbox appear to mostly be “classic” memory corruption bugs as opposed to bugs in V8. For these reasons, the sandbox is assumed to be an easier-to-defend security boundary than the V8 VM.

Finally, it is worth noting that, although not an explicit goal, this design also prevents an attacker from directly (but not speculatively) reading, not just writing, data outside of the sandbox.

The remainder of this section discusses the central sandboxing mechanisms in some more detail, then concludes with a brief summary of the design.

## Sandbox Address Space

The sandbox currently assumes a shared pointer compression cage which is shared between all V8 Isolates and thus Heaps in the process. This 4GB region is then placed at the start of a much larger (e.g. 1TB) virtual address space reservation - the sandbox - which is surrounded by large guard regions on both sides to prevent indexed accesses to reach outside of the sandbox. The remaining space in the sandbox is used to allocate other objects directly referenced by V8, such as ArrayBuffer backing stores and the 10GB Wasm memory cages.

The address space reservation backing the sandbox is discussed in more detail in this 
[document](https://docs.google.com/document/d/1PM4Zqmlt8ac5O8UNQfY7fOsem-6MhbsB-vjFI-9XK6w/edit?usp=sharing).

The structure looks like this:

```text
[ Guard Region ]
        ↓
[ V8 Sandbox: for example, 1TB ]
    ├── [ 4GB Pointer Compression Cage ]
    │       └── Area where JS objects are mainly stored
    │
    ├── [ ArrayBuffer Backing Stores ]
    │       └── Actual data storage for ArrayBuffers
    │
    ├── [ Wasm Memory Cages ]
    │       └── WebAssembly memory area
    │
    └── Other memory directly referenced by V8
        ↓
[ Guard Region ]
```

Structurally, this means that the 4GB pointer compression cage is not the entire sandbox. Instead, the 4GB cage is placed inside a much larger sandbox. For example, the entire sandbox may be 1TB in size, and the 4GB cage is placed at the beginning of it.

```text
1TB Sandbox
┌──────────────────────────────────────────────┐
│ 4GB pointer compression cage                 │
├──────────────────────────────────────────────┤
│ ArrayBuffer backing store                    │
├──────────────────────────────────────────────┤
│ Wasm memory cage                             │
├──────────────────────────────────────────────┤
│ Other sandboxed V8 memory                    │
└──────────────────────────────────────────────┘
```

The reason for doing this is to limit memory accesses to within the sandbox, even if a V8 vulnerability corrupts an object’s length or index and causes an out-of-bounds read/write.

## Sandboxed Pointers

All objects located inside the sandbox can be referenced from objects in the V8 heaps through a 40-bit (in the case of a 1TB sandbox) offset from the base of the sandbox. These are called “SandboxedPointers”. Using an offset instead of a raw pointer prevents an attacker from addressing memory outside of the sandbox.

SandboxedPointers are especially performant as the base address of the sandbox is usually already available in a register. The offsets can then be stored shifted to the left, in which case loading a sandboxed pointer only requires shifting the loaded value to the right and adding it to the base register. This is possible with two additional instructions on x64 (shift + add) and a single additional instruction on arm64 (the add instruction can also perform the shifting).

An attacker able to corrupt data inside a V8 heap can then corrupt the offsets (and sizes) of ArrayBuffer objects, and so it must be assumed that an attacker can corrupt any data inside the sandbox. However, a SandboxedPointer guarantees that it will always point into the sandbox.

Sandboxed pointers are discussed in more detail in this [document](https://docs.google.com/document/d/1HSap8-J3HcrZvT7-5NsbYWcjfc0BVoops5TDHZNsnko/edit?usp=sharing).

What are the similarities and differences between the sandbox and the pointer-compression cage?

First, the similarity is that both access memory by using an offset within a memory region.

```text
Pointer Compression Cage:
  real_ptr = cage_base + 32-bit offset

V8 Sandbox:
  real_ptr = sandbox_base + 40-bit offset
```

The difference is that they serve different purposes. The pointer-compression cage is intended to reduce 64-bit pointers to 32-bit values in order to save memory. The sandbox, on the other hand, is intended to prevent an attacker from escaping outside the sandbox even if they manage to manipulate pointer values.

Another difference is how values are stored and retrieved. I also included Smi because it came to mind.

```text
Smi:
  When storing: value << 1
  When loading: stored >> 1
  Purpose: to use the lower bit as a tag

SandboxedPointer:
  When storing: offset << 24
  When loading: stored >> 24
  Purpose: to keep only the 40-bit sandbox offset from a 64-bit value

Pointer Compression:
  When storing: store the 32-bit offset within the 4GB cage
  When loading: cage_base + offset
  Purpose: to reduce 64-bit pointers to 32-bit values
```

## Pointer Tables

All objects located outside the sandbox (“external entities”) are referenced through pointer tables, which are themselves also located outside of the sandbox. This is conceptually similar to the file descriptor table used by an operating system’s kernel or a Wasm table.

In general, the sandbox differentiates between three different types of external entities:

- Executable code, which is managed by V8’s garbage collector but allocated in a dedicated code space, located outside of the sandbox. These are referenced via a CodePointerTable (CPT), discussed in more detail in this [document](https://docs.google.com/document/d/1CPs5PutbnmI-c5g7e_Td9CNGh5BvpLleKCqUnqmD82k/edit?usp=sharing).
- V8 objects managed by V8’s garbage collector but located outside of the sandbox for security reasons. See the Trusted Objects section below and this document for more details. These are referenced via a TrustedPointerTable (TPT).
- Non-V8 objects that are not directly managed by V8’s garbage collector. These include all embedder-managed objects such as DOM nodes as well as the ArrayBufferExtension object from the example above and many other types of objects. These are referenced via an ExternalPointerTable (EPT), which is discussed in more detail in this [document](https://docs.google.com/document/d/1V3sxltuFjjhp_6grGHgfqZNK57qfzGzme0QTk0IXDHk/edit?usp=sharing).

This section now briefly discusses how those objects are protected, in particular how memory safety is achieved. For a more detailed discussion of these mechanisms, refer to the design documents linked above.

I think understanding this properly requires a deeper understanding of each pointer table, so I will not go into too much detail for now. It would probably be better to write about this in more detail later, when I study these concepts more thoroughly.

The main point seems to be that objects inside the sandbox do not directly store addresses outside the sandbox. Instead, they store references through tables.

For example, suppose an object directly stored the actual address of an object outside the sandbox like this:

```cpp
object->external_ptr = 0x7ffff1234567;
```

If this were the case, an attacker could manipulate this value and make it point to an arbitrary address outside the sandbox. That would effectively break the purpose of the sandbox.

Because of this, objects inside the sandbox do not directly store external addresses. Instead, they store a table index or handle.

```cpp
object->external_ptr_handle = 0x1234;
```

The object uses the handle like this, while the actual address is stored in a pointer table located outside the sandbox.

```cpp
ExternalPointerTable[0x1234] = 0x7ffff1234567;
```

So the actual access happens roughly like this:

```cpp
handle = object->external_ptr_handle;
real_ptr = ExternalPointerTable[handle];
```

This can be thought of as being similar to a file descriptor.

```cpp
fd = 3;
```

A program only knows the number `3`, which acts like a handle. The actual file object is managed by the file descriptor table inside the kernel. In other words, the program does not directly hold a kernel pointer.

The pointer tables in the V8 sandbox work in a similar way.

```text
Objects inside the sandbox do not contain actual external pointers.
Instead, they contain table indexes or handles.
The actual pointers are stored in pointer tables outside the sandbox.
```

I will not go into too much detail here, but roughly speaking, each table has the following role:

```text
CodePointerTable, CPT
→ Manages executable code addresses
→ Used for things like JIT code and builtin code

TrustedPointerTable, TPT
→ Manages trusted V8 objects that are handled by V8 but need to live outside the sandbox
→ Used for Trusted Objects

ExternalPointerTable, EPT
→ Manages external objects that are not directly managed by V8’s garbage collector
→ Used for things like DOM nodes, embedder objects, ArrayBufferExtension, and so on
```

The goal is to make it difficult for an attacker to directly overwrite external raw pointers, even if they can corrupt some memory inside the sandbox. Even if an attacker can overwrite an external pointer field inside a sandbox object, that value is not an actual address. It is only a handle.

```cpp
object->external_ptr_handle = attacker_value;
```

At this point, V8 usually uses mechanisms such as handle validation, tag checks, and table entry type checks to prevent an invalid handle from being interpreted as an arbitrary external address.

One limitation here is that if an attacker can overwrite the handle, they may still be able to use an existing function or change the handle value to refer to another function already present in the table.

Of course, this is still much better than storing a raw pointer directly. If it were a raw pointer, the attacker could point it wherever they wanted.

## Temporal Memory Safety

Entries in a pointer table are managed by the garbage collector. If an entry is no longer referenced from any object in the V8 heap, the entry is cleared and the pointed-to object is released if it is managed by V8 (unless there are other references to it from elsewhere). Any subsequent access to the entry will then either see an invalid pointer if the entry is still free, or see a valid pointer to a live object if it was reused. In the latter scenario, the access is safe by design if the new object is of the same type as the previous one, otherwise, the access would fail due to the type safety mechanism described below.

Here, it is useful to think of an entry as the slot that a handle refers to. More specifically, an entry means one slot in the pointer table.

The structure looks like this:

```text
V8 Heap Object
┌────────────────────────────┐
│ external_ptr_handle = 3    │
└────────────────────────────┘
              │
              ▼
Pointer Table
┌─────────┬──────────────────────────┐
│ Entry 0 │ pointer A                │
│ Entry 1 │ pointer B                │
│ Entry 2 │ free                     │
│ Entry 3 │ actual external pointer  │  ← entry
│ Entry 4 │ pointer C                │
└─────────┴──────────────────────────┘
```

An entry usually contains information such as:

```text
entry = {
  actual pointer value,
  type information or tag,
  mark/free state,
  other metadata
}
```

This part can be summarized as follows:

```text
1. A V8 object references a pointer table entry.
2. During garbage collection, the GC checks whether any V8 heap object still references this entry.
3. If no V8 heap object references it anymore, the entry is cleared or marked as free.
4. If the external object pointed to by the entry is also managed by V8, that object may be released as well.
5. After that, if the old handle is used again:
   - if the entry is still free, it will result in an invalid pointer;
   - if the entry has been reused, it may point to another live object.
6. However, if the new object has a different type, the access will be blocked by the type checking mechanism.
```

## Spatial Memory Safety

Some objects such as ExternalStrings reference a buffer of data of a given length. This sandbox must ensure that any access to those external buffers stays in bounds of the allocated memory. This is generally possible in two ways: (1) by storing length information in the table or the external object itself and bounds-checking against that or (2) by moving those buffers into the sandbox.

This section explains that the V8 sandbox must not only protect pointer values, but also validate lengths when accessing external buffers.

For example, an `ExternalString` may not store the string data itself inside the V8 heap. Instead, it may point to an external memory buffer outside the sandbox.

```text
ExternalString object
┌──────────────────────────────┐
│ external_buffer_handle       │ ──┐
│ length = 100                 │   │
└──────────────────────────────┘   │
                                   ▼
External Buffer
┌──────────────────────────────────────┐
│ 100 bytes of string data              │
└──────────────────────────────────────┘
```

The problem here is that if an attacker can corrupt the `length` value or pointer-related metadata through some bug, they may be able to read or write beyond the actual buffer.

Therefore, the sandbox cannot stop at simply avoiding direct storage of external pointers and using handles or tables instead. It must also guarantee that whenever an external buffer is accessed, the access range stays within the actual allocated size.

The first approach is to store the length information in a safe place and check it every time the buffer is accessed.

```cpp
entry = {
  pointer: actual external buffer address,
  length: 100,
  tag: ExternalString
}
```

When using it, it would be checked like this:

```cpp
if (index < entry.length) {
    read(entry.pointer + index);
}
```

In other words, the length is stored together with the pointer table entry or the external object, and a bounds check is performed before accessing the buffer.

The second approach is to move the buffer itself into the sandbox.

```text
Before:
[V8 Sandbox] ──handle──> [External Buffer outside sandbox]

After:
[V8 Sandbox]
┌──────────────────────────────┐
│ ExternalString object         │
│ string buffer data            │
└──────────────────────────────┘
```

With this design, even if an attacker manages to create an incorrect buffer access range, the accessible memory is at least limited to the inside of the sandbox. This reduces the risk of reading from or writing to arbitrary memory outside the sandbox.

## Type Safety

To ensure type safety of external objects, the table entries consist of pairs of pointers and type tags (with the type tags potentially stored in unused pointer bits for efficiency). Before an external object is accessed, the type tag of the entry must be checked against the caller-supplied expected type, either with an explicit check or by ensuring that wrong types will result in an inaccessible address.

In simple terms, if an entry in the external pointer table only stores an address, an attacker may be able to abuse it, as mentioned earlier. Therefore, the table also stores what kind of external object the address points to, making it slightly safer to use.

Example:

```text
External Pointer Table

Entry 0:
  pointer = 0x7fff_aaaa_bbbb
  type_tag = ExternalString

Entry 1:
  pointer = 0x7fff_cccc_dddd
  type_tag = ArrayBufferBackingStore

Entry 2:
  pointer = 0x7fff_eeee_ffff
  type_tag = WasmTrustedInstanceData
```

## Thread Safety

The sandbox has to prevent concurrent access to external objects that aren’t thread safe. Ideally, this will be achieved by having a dedicated pointer table per Isolate and thus per mutator thread and ensuring that a non-thread-safe external object is only referenced from at most one table.

This section explains that external objects may become dangerous if they are accessed concurrently by multiple threads, so the pointer table structure is used to restrict the scope of access.

In V8, some objects can point to resources outside the sandbox. Examples include the external buffer of an `ExternalString`, an `ArrayBuffer` backing store, and WebAssembly-related external data. However, these external objects are not always safe to access concurrently.

For example, if an external object internally modifies its state without using locks, the following situation may occur:

```text
Thread A: modifies external_object->state
Thread B: modifies external_object->state at the same time
```

In this case, a race condition can occur.

Therefore, the ideal design looks like this:

```text
Isolate A / Mutator Thread A
┌─────────────────────────────┐
│ External Pointer Table A    │
│ Entry 0 -> External Object X│
└─────────────────────────────┘

Isolate B / Mutator Thread B
┌─────────────────────────────┐
│ External Pointer Table B    │
│ Entry 0 -> External Object Y│
└─────────────────────────────┘
```

On the other hand, the following structure is dangerous:

```text
              ┌── Pointer Table A / Thread A
External Object X
              └── Pointer Table B / Thread B
```

The contents can be summarized as follows:

```text
1. Objects inside the V8 sandbox do not directly store pointers to external objects.
   Instead, they reference them through pointer table entries.

2. Some external objects are not thread-safe.

3. If such objects are referenced simultaneously from the pointer tables of multiple
   Isolates or multiple threads, a race condition may occur.

4. Therefore, each Isolate should have its own separate pointer table.

5. Non-thread-safe external objects should be referenced from at most one pointer table.
```

If the same value is needed by two Isolates, then two separate copies of that value can be created.

```text
Isolate A / Pointer Table A
  Entry 1 -> External Object X_copy_for_A

Isolate B / Pointer Table B
  Entry 5 -> External Object X_copy_for_B
```

Alternatively, if the object was originally not thread-safe but must be shared across multiple Isolates, it can be made safe by adding mechanisms such as locks, atomic operations, reference counting, and proper lifetime management.

```text
Isolate A / Pointer Table A ─┐
                             ├── Thread-safe External Object X
Isolate B / Pointer Table B ─┘
```

In this case, the same object can be referenced from multiple pointer tables.

The last approach is to let only one thread own the object, while the other threads do not access it directly. For example, Thread A may own the external object, and Thread B may send messages to Thread A to request operations on that object.

```text
Thread A
  owns External Object X

Thread B
  does not access X directly
  only sends messages/requests
```

## Trusted Objects

There are a number of internal V8 objects that do not contain pointers, but which could still allow an attacker to break out of the sandbox. One example are code-like objects, such as interpreter bytecode, JIT-compiled machine code, and any related metadata. These are not generally robust against corruption and will thus allow an attacker to escape from the sandbox. Other examples could include heap allocator metadata or V8 objects that contain indices into off-heap data structures.

In general, for the sandbox to become robust, these objects either need to move out of the V8 heap into a “trusted” heap space, become robust against corruption, be treated as untrusted (and have some way of checking their integrity on access), or be marked as read-only.

A generic mechanism for moving such objects out of the sandbox and referencing them via a pointer table is discussed in this [document](https://docs.google.com/document/d/1IrvzL4uX_Zv0k2Iakdp_q_z33bj-qlYF5IesGpXW0fM/edit?usp=sharing).

The basic goal of the V8 sandbox is to prevent an attacker from directly pointing to memory outside the sandbox, even if they can corrupt objects inside the V8 heap. For this reason, external pointers are not stored directly. Instead, they are managed through mechanisms such as the External Pointer Table.

However, trusted objects are a different problem. Some objects may not contain external pointers directly, but if those objects are corrupted, they can still affect the execution flow or the internal state of V8.

Examples include:

```text
Interpreter bytecode
JIT-compiled machine code
Code metadata
Heap allocator metadata
Off-heap structure index
```

These objects are not just simple data. They affect how V8 executes code or how its internal structures work.

For example, if interpreter bytecode is corrupted, V8 may execute instructions that were not originally intended. If JIT code or its related metadata is corrupted, it may lead to the execution of incorrect machine code. If heap allocator metadata is corrupted, the layout of objects or memory management itself can become inconsistent.

In other words, if an attacker can corrupt these kinds of objects, the result could be:

```text
Without directly manipulating pointers
→ Manipulation of V8’s internal execution state
→ Manipulation of code execution flow
→ Possible sandbox escape
```

This is why the document describes four possible ways to protect these objects:

```text
1. Move them out of the V8 heap into a trusted heap
2. Make them robust against corruption
3. Treat them as untrusted objects and check their integrity on access
4. Mark them as read-only so they cannot be modified
```

The best approach is to move them into a trusted heap. The general V8 heap should be assumed to be corruptible by an attacker through a vulnerability. Therefore, keeping important objects inside that heap is dangerous.

So the idea is to change the structure as follows:

```text
Existing structure

Inside the V8 Heap
┌──────────────────────────────┐
│ JSObject                     │
│ BytecodeArray                │  ← Dangerous if corrupted
│ Code metadata                │  ← Dangerous if corrupted
│ Allocator metadata           │  ← Dangerous if corrupted
└──────────────────────────────┘
```

```text
Improved structure

V8 Sandbox / Normal Heap
┌──────────────────────────────┐
│ JSObject                     │
│ trusted_object_handle        │
└──────────────────────────────┘
              │
              ▼
Trusted Pointer Table
┌─────────┬────────────────────┐
│ Entry 0 │ TrustedObject A    │
│ Entry 1 │ Bytecode metadata  │
│ Entry 2 │ Code-like object   │
└─────────┴────────────────────┘
              │
              ▼
Trusted Heap / Outside the Sandbox or Protected Region
┌──────────────────────────────┐
│ Actual important internal objects │
└──────────────────────────────┘
```

Next time, I think I should study the fundamentals more thoroughly until I can understand the basic concepts accurately. That way, I can build a stronger foundation and understand the later material more quickly.
