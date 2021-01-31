---
layout: single
title: Modern Cryptography Notes (To be continued...)
date: 2021-01-31 22:10:07
categories: "Notes"
tags:
- Notes
- Course
- Crypto
header:
  teaser: https://upload.wikimedia.org/wikipedia/commons/4/4d/Lorenz-SZ42-2.jpg
---

Cryptography course notes taking from [Introduction to Modern Cryptography](http://www.cs.umd.edu/~jkatz/imc.html).

# Intro

## What make Modern Crypto **Modern**?

- The central role of *definitions*
- The importance of *formal* and *precise* assumptions
- The possibility of proof of security

## Scenarios

- Secure Communication Through Space ($A -> B$, E)
- Secure Communication Through Time ($A_{past} -> A_{future}$, E)

## Encryption Scheme Syntax

- $\mathcal{M}$: message space
- $\mathsf{Gen}$: procedure for generating keys
- $\mathsf{Enc}$: procedure for encrypting
- $\mathsf{Dec}$: procedure for decrypting
- Satisfy: $\mathsf{Dec}_k (\mathsf{Enc}_k (m)) = m$

### Steps

Encryption scheme: tuple $(\mathsf{Gen}, \mathsf{Enc}, \mathsf{Dec})$
1. $k \leftarrow \mathsf{Gen}(1^n)$ 
2. $c \leftarrow \mathsf{Enc}_k (m)$
3. $m \coloneqq \mathsf{Dec}_k(c)$

## Historic Cipher

- Shift Cipher
- Permutation Cipher
- Vigen`ere Cipher

### Attacks

- Frequency Analysis
- Index of Coincidence

## Kerckhoffs' Principle

> The cipher method must not be required to be secret, and it must be able to fall into the hands of the enemy without inconvenience.

## Formal Definition

> learn no additional information about the plaintext, regardless of any prior information an attacker has learned

## Threat Models

- Ciphertext-only attack
- Known-plaintext attack
- Chosen-plaintext attack
- Chosen-ciphertext attack

