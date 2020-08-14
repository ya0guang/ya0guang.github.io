---
layout: single
title: Micro-architectural Side Channel (Attacks)
date: 2020-08-09 21:10:07
categories: "Research Notes"
tags:
- Notes
- Research
- SGX
header:
  teaser: /assets/images/micro-archAtk/oen9fbv.jpg
---

A summary of micro architecture side-channel attacks in recent years. 

# Overview

| Paper   | Micro-architecture | Attack Model | Ability (attack)                          |
| ------- | ------------------ | ------------ | ----------------------------------------- |
| Fallout | Store Buffer       | SGX          | *mem* within the same address space, ASLR |

# Fallout

MDS-type attack, focusing on store buffer.

## Store Buffer

When CPU needs to write data to the memory, the CPU will enqueue this write operation and dispatch it to the store buffer rather than waiting the operation to complete. There are some slots in the store buffer to serve multiple store request in Intel CPU.

## Write Transient Forwarding (WTF)

The CPU first check the store buffer while loading from memory to see if it will load from a memory unit which is going to be modified a unfinished store request in SB. If the CPU finds a match (on **some** lower bits of the address), the value to be stored will be forwarded to the corresponding load.

The forwarding may have no architectural effect if it's a fault load (e.g. load from a illegal non-canonical address). However, the load may still affect cache, which can be then recovered from Flush+Reload attack.

The general steps for the attack:

1. Victim store
2. Attacker load triggering WTF. Notice that such load may fail, but have micro-architectural change.
3. Flush+Reload to recover the victim store(attacker load but failed) value from cache.

## Data Bounce

Since the store-to-load lacks write permission check and only load with valid virtual address would be forwarded, the attacker can know if an address in virtual address space (VAS) is valid (e.g. occupied by kernel).

Therefore, if the attacker can correctly recover a previously and transiently stored and reloaded value from Flush+Reload, then the address (the store and load target address) is mapped in VAS.

```asm
mov (0) -> $dummy
mov $x -> (p)
mov (p) -> $value 
mov ($mem + $value * 4096) -> $dummy
```

1. Invoke an exception, which means once the exception is caught, no architectural change will be committed.
2. Store a know value *x* to some address
3. Try to trigger WTF
4. Encode *x* and later to see if we can recover it correctly


## Fetch+Bounce

A new primitive, Fetch+Bounce can then be constructed from aforementioned two building blocks.

```pseudo
for retry = 0...2 
    mov $x (p)
    mov (p) $value
    mov ($mem + $value * 4096) $dummy 
    if flush_reload($mem + $x * 4096) then break
```

If Data Bounce succeeds on the first try, the address is in the TLB.If it succeeds onthesecond try,the address is valid but not in the TLB.


## Competence

- Breaking KASLR
- Leaking kernel memory/control flow
- Recovering AES-NI schedule


## Limitations

- Inability on KAISER-enabled machine


# Reference 

1. [MDS Attack](https://mdsattacks.com/)
2. [Fallout Paper](https://mdsattacks.com/files/fallout.pdf)