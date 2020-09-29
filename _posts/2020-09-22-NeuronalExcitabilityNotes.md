---
layout: single
title: Neuroscience Notes - Neuronal excitability
date: 2020-09-22 09:10:07
categories: "Notes"
tags:
- Notes
- Course
header:
  teaser: 
---

Notes taken form Neuroscience course, Part 2: Neuronal excitability.

Last update: Sep. 28, 2020

![neuron](https://upload.wikimedia.org/wikipedia/commons/thumb/1/10/Blausen_0657_MultipolarNeuron.png/1920px-Blausen_0657_MultipolarNeuron.png)

# Translation of Terms

| Eng      | Chs  | Comment |
| -------- | ---- | ------- |
| dendrite | 树突 |         |
| axon | 轴突 | |
| synapse | 突触| |
| myelin sheath| 髓鞘| |

# Potentials

## Membrane Potential

![memPotential](https://upload.wikimedia.org/wikipedia/commons/thumb/f/fb/Basis_of_Membrane_Potential2.png/1280px-Basis_of_Membrane_Potential2.png)

- **ion pump**: brings $K^+$ in and pushes ${Na}^+$ out.

## Two forces on ions:

- Diffusion *high* -> *low* concentration $E_d = RTln([K^+]_o/[K^+]_i)$
- Coulomb: opposites attract; like charges repel $E_c = V_zF$
- For $K^+$, At equilibrium: $E_c = E_d$

But different ion channels cause different permeabilities for each ion. Therefore we need to times the permeability as weight to each ion when calculating the membrane potential.

- **Depolarization**: a change within a cell, during which the cell undergoes a shift in electric charge distribution, resulting in less negative charge inside the cell.
- **Hyperpolarization**: a change in a cell's membrane potential that makes it more negative.

## i-on channel

- ${Na}^+$ channel: voltage gated (neg potential -> closed gate)
- Depolarization: positive feedback of ${Na}^+$ channel

1. Membrane depolarization
2. increase in $g_{Na^+}$
3. more ${Na}^+$ rushes into cell
4. lead to 1 => positive feedback

However, $K^+$ behave differently => No feedback

## Electrophysiology

- Current: $I =\Delta Q/\Delta t$
- Ohm's law
- Abstract neuron as circuit: pumps(voltage src) - switch // resistor // capacitor

## Hodgkin & Huxley

### Energy Barrier Model

- Gate: voltage dependence, time dependent
- 双向流动的模型：$dy/dt = \alpha(1-y)-\beta y$ => $y_{\infty} = \alpha/(\alpha + \beta)$

``` 
      	β
open(y) ⇋ closed(1-y) => y∞ (steady state)
        α
```

### Hodgkin and Huxley's Model

ref: [wiki_HHModel](https://en.wikipedia.org/wiki/Hodgkin%E2%80%93Huxley_model)

- Experiment on squid giant axon with voltage clamp
- a brief inward current followed by a longer outward current
- TTX blocking $Na^+$ channels and TEA blocking $K^+$ channels => separate the current of each type of channels

![wiki_HHModelCircuit](https://upload.wikimedia.org/wikipedia/commons/thumb/9/98/Hodgkin-Huxley.svg/525px-Hodgkin-Huxley.svg.png)