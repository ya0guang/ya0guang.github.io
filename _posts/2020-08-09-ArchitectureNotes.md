---
layout: single
title: Computer Architecture Notes
date: 2020-08-09 21:10:07
categories: "CS Notes"
tags:
- Notes
- Research
- Architecture
- Books
header:
  teaser: 
---

# Intro

This note is taken from the book: *Computer Architecture: A Quantitative Approach, 6th Edition*. 

# Chapter 1: Fundamentals of Quantitative Design and Analysis

## Concepts

### ISA

1. Class: GPR, Stack, Accumulator
2. Memory addressing: byte/others, aligned or not
3. Addressing modes: imm, base+displacement, base+scaled index + displacement
4. Types & size of operands: supported FPs and ints
5. Operations: allowed operations of data transfer, arithmetic, logical, control
6. Control flow ins: conditional/unconditional branch, procedure call/return
7. Encoding: compressed or not, etc.

## Ideas

- Embedded processors (especially ARM) compose the biggest part of CPU market.
- Even a very small portion of downtime can lead to huge financial loss for service provider.
- Design goals: application area, SW compatibility, OS requirement, standards
- Processors and compilers can be tuned for benchmarks, thus real-world apps will be the best gauge for performance.
- Enhancing common cases would achieve better performance.

## Trends

- IC: Moore's law is no longer applicable. $Die size +$, $transistor +$
- Efforts in making processor faster: ILP (pipeline, frequency) -> DLP/TLP/RLP (SIMD, multicore) -> application-specific
- Power consumption: $static power ++$, $voltage -$ -> $dynamic energy --$

## Formulas

### Cost of IC (*C for Cost*)

$$ C = \frac{C_{Die} + C_{packaging\&test}}{Final\ test\ yield}$$
$$ C_{die}  = \frac{C_{Wafer}}{Die\ per\ wafer * Die\ yield}$$
$$ Dies\ per\ wafer = \frac{\pi * (Wafer\ diameter/2)^2 }{Die\ area} - \frac{\pi * Wafer\ diameter}{\sqrt{2 * Die\ area}}$$

### Dependability  

**Reliability**: measure of the continuous service accomplishment from a reference instant. MTTF, FIT, MTTR.  
**Availability**: measure of the service accomplishment w.r.t. the alteration between the two states of accomplishment and interruption.  
$$Module availability = \frac{MTTF}{MTTF + MTTR} $$

### Performance

$$n = \frac{Exec\ time_Y}{Exec\ time_X} = \frac{Perf_X}{Perf_Y}$$

#### *Amdahl's law*
$$Speedup = \frac{Perf.\ for \ entire\ task\ with\ enhancement}{Perf.\ for \ entire\ task\ without\ enhancement}$$  
Thus, $Exec\ time_{new} = Exec\ time_{old} * ((1-Fraction_{enhanced}) + \frac{Fraction_{enhanced}}{Speedup_{enhanced}})$

#### CPU time
$$CPU\ time = \frac{Instructions}{Program} * \frac{Clock\ cycles}{Instructions} * \frac{Seconds}{Clock cycle} = \frac{Seconds}{Program}$$

- Clock cycle time: HW tech. & org.
- CPI: Org. & ISA
- \#Instruction: ISA & compiler

