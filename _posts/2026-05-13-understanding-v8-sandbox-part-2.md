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
[High-Level Design Doc]https://docs.google.com/document/d/1FM4fQmIhEqPG8uGp5o9A-mnPB5BOeScZYpkHjo0KKA8/edit?tab=t.0 <br>

<br><br><br>

## Summeary

Objective: build a low-overhead, in-process sandbox for V8. <br>
Motivation: V8 bugs typically allow for the construction of unusually powerful and reliable exploits.   <br>
 Furthermore, these bugs are unlikely to be mitigated by memory safe languages or upcoming hardware-assisted security features such as MTE or CFI.  <br>
 As a result, V8 is especially attractive for real-world attackers.  <br>
Design: The proposal assumes that an attacker can arbitrarily corrupt memory on the V8 heap, where JavaScript objects are located.   <br>
This primitive can be constructed from typical V8 vulnerabilities.  <br>
To protect other memory within the same process from corruption, and by extension to prevent the execution of arbitrary code, the V8 heap is moved into a pre-reserved region of virtual address space: the sandbox.  <br>
Then, all memory accesses performed by V8 must either be restricted to the sandbox address space (e.g. by using offsets instead of raw pointers to reference objects) or be validated in some way (e.g. by using a pointer table indirection).  <br> 
In particular, full 64-bit pointers must be banned entirely from the V8 heap.  <br>

Before talking about MTE and CFI, let’s first understand what MTE and CFI actually are.
Both are security mitigation techniques designed to make it harder for memory vulnerabilities to lead to actual code execution.

### MTE 

First, MTE (Memory Tagging Extension) is a hardware feature available on ARM-based CPUs. It is a memory tagging extension mechanism. MTE attaches tags both to pointers and to memory regions, and whenever a program reads from or writes to memory, the CPU checks whether the two tags match.

For example, when an object is allocated on the heap, the memory region may receive tag A, and the pointer referencing that object will also receive tag A. Later, when the pointer is used to access memory, the CPU verifies whether the pointer tag and the memory tag are equal. If the tags do not match, the CPU treats it as an invalid memory access and raises an error.

The types of bugs MTE mainly tries to detect are usually UAF, BOF, OOB, and similar memory safety issues.

The tag itself is typically just a small integer value, and the allocator, runtime, or kernel usually selects it randomly or pseudo-randomly during memory allocation.

In other words, the tag value itself is unrelated to the object type.

ARM MTE tags are typically 4-bit values, meaning they range from 0 to 15.

For example, suppose the following code executes:

```C++
char* p = malloc(32);
```
The allocator would then:

Allocate the memory region
Set the tag of the memory granules to tag = 5
Record tag = 5 in the upper bits of the returned pointer

Later, when the memory is accessed, the CPU checks whether the tag values match, i.e., whether `5==5`.

Another important point is that although MTE stores tags per granule (16-byte units), if the allocated size is larger than 16 bytes, the allocator typically fills the entire allocation with the same tag value so that the tags remain consistent.

On the other hand, if the allocation size is smaller, for example 8 bytes, the granule size is still 16 bytes. As a result, two separate 8-byte objects A and B may share the same tag. This is referred to as a sub-granule overflow issue.

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

For memory tags, MTE stores tags both inside the pointer itself and inside memory-side metadata.

For pointer tags:

ARM64 does not use the full 64-bit address space, so the upper bits can be utilized like this:

`| tag | virtual address |`

Memory tags, on the other hand, are stored in hidden metadata associated with memory.

There are also situations where MTE is not sufficient.

Case 1 is simple: if an OOB access occurs but the out-of-bounds region happens to have the same tag value, the access will still succeed because the tags match.

Case 2 is the sub-granule overflow issue mentioned earlier.

Since MTE assigns tags in 16-byte granules, anything occurring within the same granule shares the same tag. As a result, overflows occurring inside a single granule cannot be detected.

### CFI

CFI stands for Control Flow Integrity. It is a technique designed to protect the integrity of a program’s control flow.

Programs execute according to control flow operations such as function calls, returns, branches, and virtual function calls. CFI is designed to prevent attackers from abusing memory corruption to redirect execution flow to unintended locations.

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

Next is Backward-edge CFI, which protects control flow returning backward.

Examples include:

function returns
return addresses stored on the stack

One common mechanism is the shadow stack.

When a function is called, the return address is stored both on the normal stack and on a separate protected stack called the shadow stack.

Then, when the function returns, the system compares the return address on the normal stack with the value stored in the shadow stack to ensure they match.

Although this explanation took a bit of a detour, the reason why MTE and CFI are considered insufficient for mitigating V8 vulnerabilities is the following:

MTE is strong at detecting invalid memory accesses, but as discussed earlier, it can still miss type confusion and data manipulation occurring within seemingly valid objects whose tags match.

CFI is strong at preventing control flow hijacking, but attackers do not always need to jump to invalid addresses. For example, attackers may perform data-only attacks by invoking legitimate functions within valid control flow while manipulating only data. Similarly, manipulation of JIT internal states is difficult to stop using CFI alone.

Additionally, because JIT engines generate code dynamically at runtime, defining what constitutes “legitimate control flow” is significantly harder than in ordinary static programs.

The sandbox design itself assumes that the V8 heap has already been compromised. Therefore, it reserves a large virtual memory region and places all JavaScript objects inside it so that memory outside the sandbox cannot be accessed.

Furthermore, if pointers inside the sandbox were stored as full 64-bit pointers, attackers could overwrite them with arbitrary addresses outside the sandbox. Instead, the sandbox represents addresses using offsets:

```C++
offset = 0x1234
real_address = sandbox_base + offset
```

By representing addresses as offsets, no matter what value an attacker writes, it becomes impossible to construct an address outside the sandbox range because the offset itself has a size limitation.

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

Finally, the reason raw pointers must not exist is, as discussed repeatedly above, because attackers could manipulate them to point to addresses outside the sandbox, ultimately enabling sandbox escape.




`The rest of the content will be added later.`



















