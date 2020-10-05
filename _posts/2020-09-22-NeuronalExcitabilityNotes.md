---
layout: single
title: Neuroscience Notes - Neuronal excitability
date: 2020-09-22 09:10:07
categories: "Notes"
tags:
- Notes
- Course
- BCI
header:
  teaser: https://upload.wikimedia.org/wikipedia/commons/thumb/1/10/Blausen_0657_MultipolarNeuron.png/1920px-Blausen_0657_MultipolarNeuron.png
---

Notes taken form Neuroscience course, Part 2: Neuronal excitability.

Last update: Sep. 28, 2020

![neuron](https://upload.wikimedia.org/wikipedia/commons/thumb/1/10/Blausen_0657_MultipolarNeuron.png/1920px-Blausen_0657_MultipolarNeuron.png)

# Translation of Terms

| Eng           | Chs        | Comment |
| ------------- | ---------- | ------- |
| dendrite      | 树突       |         |
| axon          | 轴突       |         |
| synapse       | 突触       |         |
| myelin sheath | 髓鞘       |         |
| vagus nerve   | 迷走神经   |         |
| ionotropic    |            |         |
| exocytosis    | 胞外分泌   |         |
| vesicle       | 泡，囊     |         |
| ionotropic    | 离子移变的 |         |
| acetylcholine | 乙酰胆碱 | ACh|


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

# Synaptic Transmission

- Electrical synapses
- **Chemical synapses**

## Otto Loewi's Experiments

![wiki_loewi](https://upload.wikimedia.org/wikipedia/commons/c/c6/Vagusstoff2.png)

- Two hearts
- stimulate one's vagus nerve
- transfer the solution to heart 2
- both of these two heart rates slows

## Bernard Katz's Experiments

m[EPP](https://en.wikipedia.org/wiki/End-plate_potential): miniature end plate potential

- Action potential needs to reach the threshold (EPP)
- mEPPs are spontaneous
- EPPs are evoked!
- => quantal hypothesis: probability; quantum; evoked EPP by several quanta

## Quantal Analysis

- number of packets released $m = V_{EPP}/V_{mEPP}$
- release of quanta: follow a Poisson distribution: $P(x) = m^xe^{-m}/x!$ 
- => counting the times evoked EPPs deliver no transmitter => P(0) => m
- => vesicles were the transmitter quanta

## Chemical Synaptic Transmission 

![chemTrans](/assets/images/Neuro/chemTrans.jpg)

1. $Na^+$ in, $K^+$ out => Synapse
2. voltage-gated channels open => $Ca^{++}$ in
3. vesicles to fuse with membrane => release transmitter that diffuse across the synaptic cleft
4. transmitter molecules reach receptors (postsynaptic) => open channels and letting in ions
5. synaptic vesicles in presynaptic terminals can be recycled

Two Categories:

- Ionotropic: fast, local channel-mediated changes in membrane potential (EPSPs, IPSPs)
- Metabotropic: slow, G-protein- and second-messenger- mediated changes 


## Benifits of chemical transmission

- Graded vs. all-or-none: analog vs. digital
- Amplification
- Sign inversion
- Varied temporal duration
- Multiple receptors with distinct properties
- Malleability/plasticity

## Synaptic Reversal Potentials

- EPC: end plate current, the macroscopic current resulting from the summed opening of many ion channels
- reversal potential: the membrane potential at which a given neurotransmitter causes no net current flow of ions through that neurotransmitter receptor's ion channel.

- $Ca^{2+}$ entry through the specific voltage-dependent calcium channels in the presynaptic terminals causes transmitter release
- a rise in presynaptic Ca2+ concentration triggers transmitter release from presynaptic terminals

## Gap Junctions

![wiki_gapJ](https://upload.wikimedia.org/wikipedia/commons/thumb/b/b7/Gap_cell_junction-en.svg/1920px-Gap_cell_junction-en.svg.png)

- Electrical Synapse