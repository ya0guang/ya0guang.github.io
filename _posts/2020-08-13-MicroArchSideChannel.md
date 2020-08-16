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

| Paper     | Micro-architecture | Attacker Priv.   | Victim Priv. | Ability (attack)                          |
| --------- | ------------------ | ---------------- | ------------ | ----------------------------------------- |
| Spectre   | Branch Predictor   | User space prog. | cross-prog.  | Control flow hijack                       |
| Meltdown  | Out of Order       |                  |              |                                           |
| Fallout   | Store Buffer       | User space prog. | Kernel       | *mem* within the same address space, ASLR |
| LVI       | SB, L1D            | Kernel           | SGX Enclave  | AES-NI, Quoting Enclave                   |
| Crosstalk | Stageing Buffer    |                  |              |                                           |


## Transient Domain

Since the correctness of execution should be ensured, CPU will not commit false results and the instructions invoking fault may not retire. Thus, to change micro architecture states transiently, instruction causing exception/fault/microcode assist should be put into the pipeline before the attack instructions. Once the CPU detects exception/fault/microcode assist, the pipeline might then be squashed and the attacker may then recover the changes on micro architecture states in corresponding exception handler.


# Spectre

To be continued...

# Meltdown

To be continued...

# Fallout

MDS-type attack, focusing on store buffer. Fallout can leak memory writes/VAS (virtual address space) info by forwarding load from store buffer to cache and cache side-channel exploitation.

## Store Buffer

When CPU needs to write data to the memory, the CPU will enqueue this write operation and dispatch it to the store buffer rather than waiting the operation to complete. There are some slots in the store buffer to serve multiple store request in Intel CPU.

## Write Transient Forwarding (WTF)

The CPU first check the store buffer while loading from memory to see if it will load from a memory unit which is going to be modified a unfinished store request in SB. If the CPU finds a match (on **some** lower bits of the address), the value to be stored will be forwarded to the corresponding load.

The forwarding may have no architectural effect if it's a invalid load (e.g. load from a illegal non-canonical address). However, the load may still affect cache, which can be then recovered from Flush+Reload attack.

The general steps for the attack:

1. Victim store
2. Attacker load triggering WTF. Notice that such load may fail, but have micro-architectural change.
3. Flush+Reload to recover the victim store(attacker load but failed) value from cache.

## Data Bounce

Since the store-to-load lacks write permission check and only load with valid virtual address would be forwarded, the attacker can know if an address in VAS is valid (e.g. occupied by kernel).

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


# Load Value Injection (LVI)

LVI can load values incorrectly from various micro architectural buffers in the transient domain. Moreover, the CPU can even execute following instructions transiently on these faulting load values.

*Note: all pics in this section are from https://lviattack.eu/ or the LVI paper*


## Attack Steps

![Attack Steps](/assets/images/micro-archAtk/lvi-workings.gif)  


> 1. Poison a hidden processor buffer with attacker values.
> 2. Induce a faulting or assisted load in the victim program.
> 3. The attacker's value is transiently injected into code gadgets following the faulting load in the victim program.
> 4. Side channels may leave secret-dependent traces, before the processor detects the mistake and rolls back all operations.

### Attack Gadget Phases

- P1: Microarchitectural Poisoning
- P2: Provoking Faulting or Assisted Loads
- P3: Gadget-Based Secret Transmission

## Details

- Since the attacker want the victim to load a value controlled by him, the attacker then needs to search for gadgets.
- This attack require microcode assistance or fault, thus in SGX environment, page fault is a choice for the attacker.
- Buffer should not be flushed/overwritten when switching from attacker to victim domain, and the transient exec is totally within the victim domain.
- There should also be disclosure gadgets to encode secret data in a side-channel(e.g. cache).
- By injecting values to branch target(jmp, ret), the attacker may construct a ROP-like chain and execute any code transiently.
- Transient execution window is limited by the depth of reorder buffer.

## Taxonomy

### LVI-L1D

![LVI-L1D](/assets/images/micro-archAtk/lvi-l1d.png)

> (1) the enclave’s stack PTE is remapped to a user page outside the enclave;  
> (2) a P1 gadget inside the enclave loads attacker-controlled data into L1D;  
> (3) a P2 gadget pops trusteddata (return address) from the enclave stack, leading to faulting loads whichare transiently served with poisoned data from L1D;  
> (4) the enclave’s transient execution continues at an attacker-chosen P3 gadget encoding arbitrary secrets in the microarchitectural CPU state  

> "reverse Foreshadow"

> LVI-L1D can best be regarded as a transient page-remapping primitive allowing to arbitrarily replace the outcome of any legitimate enclave load value (e.g., a return address on the stack) with any data currently residing in the L1D cache and sharing the same virtual page offset.


### LVI-SB, LVI-LFB, and LVI-LP

The micro architectural buffers exploited in MDS-type attacks can also be abused in LVI. 

| LVI type | Corresponding MDS attack | Buffer used      | Requirement                |
| -------- | ------------------------ | ---------------- | -------------------------- |
| LVI-SB   | Fallout                  | store buffer     | page offset match (12 LSB) |
| LVI-LFB  | RIDL                     | line-fill buffer |                            |
| LVI-LP   | Zombieload               | load ports       |                            |

### LVI-Null

This is an interesting one: the patch of meltdown introduced new exploitable mechanism.  

On meltdown-resistent machines, `0x00` will still be passed to dependent instructions even though the CPU detects faulting load. The `0x00` value can be used to play with conditional branch instruction.

![LVI-Null](/assets/images/micro-archAtk/lvi-null.png)

> (1) a P2 gadget inside the enclave dereferences a function pointer-to-pointer, leading to a faulting load which forwards the dummy value null;  
> (2) the following indirect call transiently dereferences the attacker-controlled null page outside the enclave, causing execution to continue at an attacker-chosen P3 gadget address  

## Competence

Since such an attack requires fine-grained control over faults/PTE entries and knowledge about source code/current IP, the authors apply SGX threat model. But they also claim LVI attack may also be extended to attack kernel from user space or even launch cross-process attack.

- Attack: Quoting enclave, AES-NI

## Mitigation

- `lfence`: high overhead

# Crosstalk

Traditionally, people believe that side-channel leakage is only possible when the attacker and the victim is on the same core. This paper introduces a new way of leaking info across cores. 

# Reference 

1. [MDS Attack](https://mdsattacks.com/)
2. [Fallout Paper](https://mdsattacks.com/files/fallout.pdf)
3. [LVI](https://lviattack.eu/)