---
layout: single
title: Constant Time
date: 2023-03-01 10:10:07
categories: "Tech"
tags:
- Research
- CS
- Crypto
- PL
header:
  teaser: 
---

# Intro

Constant-time作为密码学实现上一个重要的property，在抵御side-channel上面有很重要的作用。尤其是在这个Post-spectre的年代，microarchitectural level的side-channel attack已经成为了一个很大的威胁。近年来这个领域的researcher们飙了很多关于constant time的文章，很多都和microarchitectural side-channel相关。其中A哥（Almeida）和B哥(Barthe)尤甚。我还发现这个领域的大佬们似乎有很多欧洲的，可能也和欧洲佬数学好，喜欢搞各种verification有关。

大体上讲，Constant-time指的是运行的时间与secret无关。运行时间的不同主要源于：

- Control-flow dependency：比如secret在`if`中出现
- Latency fo instruction：一些指令的运行时间并不是固定的，其可能取决于oprands。e.g. `idiv`）
- Data/Code access dependency：根据secret来access data时，显然access memory的时间比register/cache长
- Microarchitectural：即使constant-time在通常的model下面被满足了，speculative execution可能会导致secret泄漏

## Summary

| Paper               | Abstract Language | IR  | Binary | Notes                                                                                                                       | Code                                               | Comments           |
| ------------------- | ----------------- | --- | ------ | --------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- | ------------------ |
| [PLDI'20](#PLDI'20) | ✅                 | ✅   | -      | [here](https://ya0guang.com/tiddly/output/notebook.html#Constant-Time%20Foundations%20for%20the%20New%20Spectre%20Era)      | [link](https://github.com/PLSysSec/pitchfork-angr) | Symbolic Execution |
| [PLDI'19](#PLDI'19) | ✅  DSL            | ✅   | -      | [here](https://ya0guang.com/tiddly/output/notebook.html#FaCT%3A%20A%20Flexible%2C%20Constant-Time%20Programming%20Language) | [link](https://github.com/PLSysSec/FaCT)           | z3 + ct-verif      |

## A Dilemma in Verification

对于Constant time的verification而言这是一个格外困难的问题。source code和assembly code之间是存在gap的。这就导致了在一些情况下，即便是source code的constant-time property验证通过时，其对应的assembly code也可能不满足constant-time property。Compiler在优化时一般会guarantee semantics consistency，但是无法保证constant-time。
在源码上验证的context更为丰富，信息较多，但是在binary上验证的结果会更加可靠。

这就对verification造成了困难。目前主流的做法是在IR上面做验证。其既能很大程度上保留语义的信息，在compilation pipeline里面也比较贴近low-level assembly。

## Abstract Language Model

这个level的工作主要是从理论上解决一些问题。作者首先定义一个abstract language model，然后在此model上面定义可能的side-channel leak, e.g. small-step semantics[PLDI'20](#PLDI'20)。 这个model需要能够帮助real-world assembly/IR来reason constant time。

## At IR Level

大多数工作都是在IR level上做verification，especially LLVM IR。这个level能使用的分析工具就有很多，同时比较贴近low-level assembly code。这里可能也存在一些IR constant-time到assembly constant-time的gap。

## Executable Binary

这个level的工作一般而言不是很好formalize。似乎大多数都是使用接近于test的方法去做的（[DATE'17](#DATE'17), [SP'20](#SP'20)）。

## Reference

1. <a name="PLDI'20" href="https://dl.acm.org/doi/pdf/10.1145/3385412.3385970">PLDI'20</a> Sunjay Cauligi, Craig Disselkoen, Klaus v. Gleissenthall, Dean Tullsen, Deian Stefan, Tamara Rezk, and Gilles Barthe. 2020. Constant-time foundations for the new spectre era. In Proceedings of the 41st ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI 2020). Association for Computing Machinery, New York, NY, USA, 913–926. https://doi.org/10.1145/3385412.3385970
2. <a name="POPL'19" href="https://dl.acm.org/doi/pdf/10.1145/3371075">PLDI'20</a> Gilles Barthe, Sandrine Blazy, Benjamin Grégoire, Rémi Hutin, Vincent Laporte, David Pichardie, and Alix Trieu. 2019. Formal verification of a constant-time preserving C compiler. Proc. ACM Program. Lang. 4, POPL, Article 7 (January 2020), 30 pages. https://doi.org/10.1145/3371075
3. <a name="Security'16" href="https://www.usenix.org/conference/usenixsecurity16/technical-sessions/presentation/almeida">Security'16</a> Almeida, Jose Bacelar, et al. "Verifying {Constant-Time} Implementations." 25th USENIX Security Symposium (USENIX Security 16). 2016.
4. <a name="DATE'17" href="https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=7927267">DATE'17</a> Reparaz, Oscar, Josep Balasch, and Ingrid Verbauwhede. "Dude, is my code constant time?." Design, Automation & Test in Europe Conference & Exhibition (DATE), 2017. IEEE, 2017.
5. <a name="EuroSP'18" href="https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=8406587"> EuroSP'18</a> Simon, Laurent, David Chisnall, and Ross Anderson. "What you get is what you C: Controlling side effects in mainstream C compilers." 2018 IEEE European Symposium on Security and Privacy (EuroS&P). IEEE, 2018.
6. <a name="CCS'22" href="https://dl.acm.org/doi/pdf/10.1145/3548606.3560689"> CCS'22</a>  Ammanaghatta Shivakumar, Basavesh, et al. "Enforcing Fine-grained Constant-time Policies." Proceedings of the 2022 ACM SIGSAC Conference on Computer and Communications Security. 2022.
7. <a name="INDOCRYPT'20" href="http://repositorium.uminho.pt/bitstream/1822/71547/1/Certified%20Compilation%20for%20Cryptography.pdf"> INDOCRYPT'20</a>  Almeida, José Bacelar, et al. "Certified compilation for cryptography: Extended x86 instructions and constant-time verification." Progress in Cryptology–INDOCRYPT 2020: 21st International Conference on Cryptology in India, Bangalore, India, December 13–16, 2020, Proceedings 21. Springer International Publishing, 2020.
8. <a name="Security'19" href="usenix.org/system/files/sec19-gleissenthall.pdf?ref=hvper.com&utm_source=hvper.com&utm_medium=website"> Security'19</a>  Gleissenthall, Klaus V., et al. "IODINE: Verifying constant-time execution of hardware." Usenix Security. Vol. 19. No. 10.5555. 2019.
9. <a name="PLDI'19" href="https://dl.acm.org/doi/pdf/10.1145/3314221.3314605"> PLDI'19</a> Cauligi, Sunjay, et al. "Fact: a DSL for timing-sensitive computation." Proceedings of the 40th ACM SIGPLAN Conference on Programming Language Design and Implementation. 2019.
10. <a name="SP'20" href="https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9152766"> SP'20 </a> Daniel, Lesly-Ann, Sébastien Bardin, and Tamara Rezk. "Binsec/rel: Efficient relational symbolic execution for constant-time at binary-level." 2020 IEEE Symposium on Security and Privacy (SP). IEEE, 2020.

## Less Related Papers

1. [“They’re not that hard to mitigate”: What Cryptographic Library Developers Think About Timing Attacks](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9833713&tag=1)
2. [Jasmin: High-Assurance and High-Speed Cryptography](https://dl.acm.org/doi/pdf/10.1145/3133956.3134078)
3. [SoK: Computer-Aided Cryptography](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9519449)
4. [Eliminating timing side-channel leaks using program repair](https://dl.acm.org/doi/pdf/10.1145/3213846.3213851)
6. [Memory-Safe Elimination of Side Channels](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9370305)
7. [Constantine: Automatic Side- Channel Resistance Using Efficient Control and Data Flow Linearization](https://dl.acm.org/doi/pdf/10.1145/3460120.3484583)

## Other Resources

1. <a name="Agner" href="https://agner.org/optimize/"> Agner Software optimization resources</a> There is a table of instruction latency and throughput: [pdf](https://agner.org/optimize/instruction_tables.pdf).
2. [Is CLMUL constant time?](https://stackoverflow.com/questions/53401547/is-clmul-constant-time)
3. [Intel: Guidelines for Mitigating Timing Side Channels Against Cryptographic Implementations](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/secure-coding/mitigate-timing-side-channel-crypto-implementation.html)
4. [Intel® 64 and IA-32 Architectures Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
