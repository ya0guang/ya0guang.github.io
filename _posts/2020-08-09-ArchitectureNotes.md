---
layout: single
title: Computer Architecture Notes
date: 2020-08-09 21:10:07
categories: "CS Notes"
tags:
- English
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

## Trends

- IC: Moore's law is no longer applicable. $Die size +$, $transistor +$
- Efforts in making processor faster: ILP (pipeline, frequency) -> DLP/TLP/RLP (SIMD, multicore) -> application-specific
- Power consumption: $static power ++$, $voltage -$ -> $dynamic energy --$