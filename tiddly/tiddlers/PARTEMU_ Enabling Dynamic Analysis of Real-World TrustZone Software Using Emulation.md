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

![Architecture](https://ya0guang.com/assets/images/CCPapers/PARTEMU2.png)

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

## Review

### Strength

- Solid work
- Efforts taken to run TZOS and TA in emulation environment
- Acceptable performance

### Weakness

- Low coverage
- Crashes -X-> vulnerabilities