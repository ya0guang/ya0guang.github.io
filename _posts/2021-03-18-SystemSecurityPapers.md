---
layout: single
title: System Security 论文笔记 CC
date: 2021-01-31 22:10:07
categories: "Notes"
tags:
- Notes
- Research
- SGX
- CS
- Code Analysis
header:
  teaser: 
---

为了拯救我孱弱的记忆力，开坑记一些和最近的项目相关的论文的笔记。

# COIN Attack

- *ASPLOS 2020*
- [Paper](http://ww2.cs.fsu.edu/~khandake/paper/coin_asplos_2020.pdf)
- [Soruce Code](https://github.com/mustakimur/COIN-Attacks)

## Target

To find vulnerabilities of SGX applications in four models: 
- **C**oncurrent calls: multithread, race condition, improper lock...
- **O**rder: assumes the calling sequence
- **I**nput manipulation: bad OCALL return val & ECALL arguments
- **N**ested calls: calling OCALL that invokes ECALL, *not implemented*

## Design & Method

The design of COIN:

![COIN Overview](/images/CCPapers/COIN2.png)

![COIN Modules](/images/CCPapers/COIN3.png)

- Emulation: QEMU
- Symbolic execution: Triton (backed by z3)
- Policy-based vulnerability discovery

## Results

COIN uses 8 policies to find the potential vulnerabilities:

- Heap info leak 
- Stack info leak 
- Ineffectual condition
- Use after free
- Double free
- Stack overflow
- Heap overflow
- Null pointer dereference

## Review

### Strength

- Symbolic execution + emulation
- Policies can be configurable
- Real world problems

### Weakness

- *nested call* left unimplemented
- May not be powerful enough to deal with complicate situations
  - Policies are mainly at relatively low-level

# PartEmu

- *USENIX Security 2020*
- [Paper](https://www.usenix.org/conference/usenixsecurity20/presentation/harrison)
- **Source code unavailable**

Traditional TrustZone OSes and Applications is not easy to fuzz because they cannot be instrumented or modified easily in the original hardware environment. So to emulate them for fuzzing purpose.

## Targets

- Emulate TrustZone OSes(TZOS) and Trusted Applications (TAs) 
- Abstract and reimplement a subset of hardware/software interfaces
- Fuzz these components
- TZOSes: QSEE, Huawei, OPTEE, Kinibi, TEEGRIS(Samsung) & TAs

## Design & Method

![Architecture](/images/CCPapers/PARTEMU2.png)

- Re-host the TZOS frimware
- Choose the components to reuse/emulate carefully
  - Bootloader
  - Secure Monitor
  - TEE driver and TEE userspace
  - MMIO registers (easy to emulate)

### Tools

- TriforceAFL + QEMU
- Manually written Interfaces

## Results

Emulations works well. For upgraded TZOSes, only a few efforts are needed for compatibility.

### TAs

#### Challenges

- Identifying the fuzzed target
- Result stability (migrate to hardware, reproducibility)
- Randomness

| Class           | Vulnerability Types                                       | Crashes |
| --------------- | --------------------------------------------------------- | ------- |
| Availability    | Null-pointer  dereferences                                | 9       |
|                 | Insufficient shared memory crashes                        | 10      |
|                 | Other                                                     | 8       |
| Confidentiality | Read from attacker-controlled pointer  to shared memory   | 8       |
|                 | Read from attacker-controlled                             | 0       |
|                 | OOB buffer length to shared memory                        |
| Integrity       | Write to secure memory using  attacker-controlled pointer | 11      |
|                 | Write to secure memory using                              | 2       |
|                 | attacker-controlled OOB buffer length                     |         |

Just like the previous paper, the main causes of the crashes can be attributed to:
- Assumptions of Normal-World Call Sequence
- Unvalidated Pointers from Normal World
- Unvalidated Types

### TZOSes

- Normal-World Checks
- Assumptions of Normal-World Call Sequence