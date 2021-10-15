---
layout: single
title: Neuroscience Notes - Neurobiology of Motor Systems
date: 2020-10-19 09:10:07
categories: "SoK"
tags:
- Notes
- Course
- BCI
header:
  teaser: assets/images/Neuro/braInJar.jpg
---

Notes taken form Neuroscience course, Part 3: Neurobiology of Motor Systems.

Last update: Oct. 31, 2020

# Terms

| Eng             | Chs              | Comment                       |
| --------------- | ---------------- | ----------------------------- |
| somatotopy      | N/A              | soma-:body; -topy:org/mapping |
| rostral         |                  | towards head (face)           |
| caudal          |                  | towards tail (back head)      |
| dorsal          |                  | towards back  (top head)      |
| ventral         |                  | towards stomach               |
| ipsilateral     |                  | same side                     |
| contralateral   |                  | opposite side                 |
| cerebral        | 大脑的           |                               |
| proximal        |                  | closer (to head)              |
| distal          |                  | further (to head)             |
| coronal plane   |                  | 耳朵和头顶的平面              |
| sagittal plane  |                  | 鼻子嘴头顶的平面              |
| lobe            | 脑叶             |                               |
| sulcus          | 沟               |                               |
| corpus callosum | 胼胝体           | 连接半脑的                    |
| sulci           | 沟               | folds                         |
| gyri            | 脑回             |                               |
| cortex          | 皮层、灰质       |                               |
| afferent        |                  | (生理的) input                |
| cerebellum      | 小脑             |                               |
| proprioception  | 本体感受         |                               |
| kinesthesia     | 运动觉           |                               |
| spindle         | 纺锤             |                               |
| myelinated      | 有髓鞘的         |                               |
| flexor          | 屈肌             |                               |
| parietal        | 腔壁的；颅顶骨的 |                               |
| pons            | 脑桥             |                               |
| granular        | 颗粒的           | granular cell                 |
| basal ganglia   | 基底核           |                               |
| athetosis       | 手足徐动症       |                               |
| 

# Pre
 
![brainAnatomy](/assets/images/Neuro/anatomy-function-brain-areas-basics-large.jpg)

> pic from: https://www.dana.org/article/neuroanatomy-the-basics/

- cortex: 6 layers

## Brain areas definitions

- Brodmann's areas: divide brain into different areas, cytoarchitecture
- Functionality 

## Experiments

- monkey: usually used for experiments 
- single neuron recordings: raster plots

### Stimulation 

| Tech (superthreshold)      | Area           | Record                       |
| -------------------------- | -------------- | ---------------------------- |
| Microstimulation           | Neuron(s)      | behavior/state               |
| Single or paired pulse TMS | 1cm^2 cortex   | Amplitude of muscle twitches |
| Repetitive TMS             | 1cm^2 cortex   | behavior/state               |
| tDCS                       | 3x3cm^2 cortex | behavior/state               |

### Recording

| Tech                  | Record                        |
| --------------------- | ----------------------------- |
| Single unit recording | freq. of neuron firing        |
| EMG                   | Electrical activity in muscle |
| fMRI                  | Changes in BOLD signal        |

## Somatotopy

![brainAnatomy](/assets/images/Neuro/Somatotopy.jpg)

- Somatotopy: The brain represents the body in an organized way

## References

[brain](https://smallcollation.blogspot.com/2013/05/brian.html#gsc.tab=0)

# Proprioception and Spinal Reflexes

- Proprioception: sense of limb position, limb movement, tension or force, effort, balance
- Kinesthesia: sense of limb position, limb movement
- motor control: feedback/feedforward

## Proprioception

### Muscle spindles

Respond to stretch

- stretched: high freq.
- contracted: low freq.
- Gamma motor neurons maintain spindle sensitivity
- Joint replacement surgery：posi & mov sense remained intact

### Golgi tendon organs

Respond to force & heaviness

- Vibration Experiment: over tendon => illusory movement/position change

### Skin stretch

- Movement sensation at most joints
- Afferents have similar sensitivity as spindle afferents, and populations have similar directional sensitivity.

### Considerations

- Exercise can affect proprioception
  - Awkwardness/clumsiness after intense exercise is not just fatigued muscles
  - Proprioceptive errors after exercise are NOT generated within the muscles
- Aging can affect proprioception
  - Sarcopenia: loss of muscle mass associated with aging
  - Spindles in aged muscle have fewer intrafusal fibers (Swash & Fox, 1972)
- Disease/injury can affect proprioception

## Spinal Cord

- Myelinated white fiber tracts (axons) surround a gray, butterfly-shaped region (cell bodies).
- Spinal tracts carry information to (afferent pathways) and from (efferent pathways) the brain, or between different spinal levels (propriospinal pathways).

### Reflex

- Spinal stretch reflex
- Flexor withdrawal
- Crossed extension: excite contralateral extensors and inhibit contralateral flexors
- **Hoffmann response (H-reflex)**
- Reflexes can be modulated, by descending inputs as well as peripheral inputs

#### Central pattern generators

 Neuronal networks that are capable of generating and sustaining rhythmic motor activity **in the absence of sensory feedback or descending input**  
 - Human CPGs are less clear

# Motor & Premotor cortices

## Primary motor cortex (M1)
!!

- M1 has gross somatotopy & forms most of the corticospinal tract
- M1 codes for movement force and direction
- M1 and skilled motor performance
- M1 **output**: Large pyramidal cell neurons (found in layer 5 of cortex) comprise the lateral corticospinal tract.
- M1 **input**: other motor areas; Somatosensory and posterior parietal cortical areas; subcortical structures 
- **BOTH** muscle activity and movement direction are represented in M1. (Kakei et al. 1999
)
- M1 may be a site of transformation from coding of direction into muscle-like coordinates
- Skilled movement <= Plasticity

| PMv                  | PMd               |
| -------------------- | ----------------- |
| “What”               | “Where”, “How”    |
| Grasping             | Reaching          |
| Object perception    | Spatial attention |
| Action understanding | Working memory    |

## PMv

- PMv translates sensory information about peripersonal space into appropriate action => Grasping
  - Caudal PMv involved in peripersonal space perception
  - Rostral PMv represents hand and mouth movements
- Visuomotor transformation for **grasping** involves parietofrontal network that includes PMv

## PMd

- PMd associated with reach preparation
- Most neurons were modulated by direction and distance.  
- Visuomotor transformation for **reaching** involves parietofrontal network that includes PMd
- 70% of PMd cells: motor preparation; 30%: spatial attention

## SMA

- SMA neurons care about sequences with conditions
- Movement sequences
- Internally generated and mental rehearsal of movements.
- Linking conditions to actions
- Intention to move
  
# Non-invasive brain stimulation

## Transcranial magnetic stimulation (TMS)

- To measure cortical activity
- To determine cause and effect
- To understand disease
- Treatment (?)	

### The corticospinal pathway

[pathway](https://media.springernature.com/original/springer-static/image/chp%3A10.1007%2F978-3-030-28852-5_3/MediaObjects/460284_1_En_3_Fig7_HTML.png)

### Mechanism

1. TMS coil (with current pulse) 
2. Magnetic field **B** 
3. Electric filed **E** in cortex
4. A current in the cortical tissue
5. Depolarization: a set of corticospinal cells  (direct/indirect)
6. Pulse in EMG

- Coil shape and focality matters
- Coil can not have deep effect and be very focal

### EMG response: Motor evoked potential (MEP)

- Response to stimulation of contralateral M1
- Easiest to measure in hand muscles
- **Motor hot spot**: The location in M1 that yields the largest MEP for a given muscle
- Neuronavigation: brainsight: brain mapping

#### Properties

- MEP delay <= distance of the muscle from motor cortex
- Amplitude <= M1 excitability, TMS intensity, awareness, muscle contraction
- Silent period

## Measurement

- Excitability
- Changes in the cortical map
  - pretraining - training - posttraining
  - shift/expansion in the area that represents a muscle
  - use-dependent plasticity
- Connectivity

### Recruitment curve (input/output curve)

- Stimulus intensity - MEP(mV)
- Gradual increase in TMS intensity => Gradual increase in MEP amplitude
- Plasticity = change in slope

### Paired pulse TMS

- Assess changes in networks of interneurons within M1
- Sub-threshold conditioning pulse (CP) followed by supra-threshold test pulse (TP)

### Measuring connectivity

- Paired pulse TMS 
  - Test pulse over M1
  - Preceded by conditioning pulse over area connected to M1

## Cause and effect

brain area X => behavior Y?
- Neuroimaging(fMRI) -> Correlation but not causation
- lesion study -> Yes, but not so practical
- brain stimulation -> rTMS/tDSC

### Repetitive TMS (rTMS)

- Train of pulses (subthreshold)
- Can increase or decrease excitability depending on pattern/frequency: LTP/LTD-like mechanism
- Can affect behavior: Effect lasts past the end of stimulation
- Often used outside of M1

#### Theta Burst TMS

- Modified rTMS paradigm to produce clearer aftereffects in human motor cortex 
- Tried 3 stimulation paradigms and measured MEPs in hand
- cTBS: most robust effect on behavior

### Transcranial direct current stimulation (tDCS)

![tDCS](https://corticalchauvinism.files.wordpress.com/2015/11/tdcs.jpg)

- Brain stimulation technique where a weak electric current is passed through the head via two electrodes.  
- Study effect of activity in area of interest by positioning electrodes:
  - Under the anode, neurons become more excitable.
  - Under the cathode, neurons become less excitable.

#### Pros and Cons

- Even lower-risk than TMS
- Portable and cheap
- But not as focal as TMS
- Intervention only, doesn’t measure anything.

# Cerebellum & Basal Ganglia

- Cerebellum excites thalamus; basal ganglia inhibit thalamus.
`just like a pair of controller`

## Cerebellum: little brain

- 10% of total brain volume, half of neurons 
- influences behavior
- trail-and-error adaptation (motor learning)

### Anatomy

- Divided into anterior, posterior, and flocculonodular lobes.
- All inputs and outputs go through the cerebellar peduncles
- All outputs are via deep cerebellar nuclei(excitatory)
- Cerebellum controls ipsilateral movements

![functional_regions](/assets/images/Neuro/Cerebellum_Functional_regions.png)

### Cortex

![cortex](https://bellspalsycranialnerves.files.wordpress.com/2013/01/cerebellar-connections.jpg)

- Purkinje cell
  - simple spike: spontaneous; driven by mossy fiber, convey ongoing info about different parameters of movement.
  - complex spike: low rate, driven by climbing fibers, related to errors in movement -> motor learning
  - Pairing stimulation of climbing fibers (CF) and parallel fibers (PF) causes LTD that reduces the parallel fiber EPSP.

- Plasticity: Movement results in error.
1. Inferior olive compares predicted and actual result of movement.
2. Climbing fiber fires, conveying error.
3. LTD of parallel fiber-Purkinje cell synapses.
4. Purkinje cell less active.
5. Deep cerebellar nucleus disinhibited.
6. Cerebellar excitation of cerebral cortex increases.
7. This tells cerebellar targets that an error was made, and changes are made to the motor command to reduce the error.

### Lesion

- Motor learning in humans with cerebellar damage -> have difficulty learning /adapting movements with practice
- cerebellar patients are slow to adapt movement to novel contexts and often do not store the effect of short-term training. 
- Cerebellar deficits <= stroke, tumor, or atrophy
- **Ataxia**: any incoordination between movements of body parts
- **Nystagmus**: an involuntary and rhythmic eye movement.
- **Dysdiadochokinesia** and **Dysmetria**

### Cognitive functions

- Anatomy: Lateral cerebellum receives input from parietal, temporal and prefrontal cortex
- Linguistic prediction: activation linked to unexpected word order and sentence ending in visually-presented sentences

## Basal ganglia

- Function: brake hypothesis

### Direct & Indirect pathways

![bgPathways](/assets/images/Neuro/bgPathway.jpg)
[Pic Source](https://www.neurovascularmedicine.com/basalgangliastrokes.php)

- Neuro transmitters
  - GABA: inhibitory 
  - Glutamate: excitatory
  - Dopamine: can be both

- **Direct**: inhibit the basal ganglia output nuclei => disinhibition of the thalamus  
cerebral cortex -> striatum -> GPi/SNpr -> thalamus -> cerebral cortex
- **Indirect**: excite the basal ganglia output nuclei => inhibition of the thalamus  
cerebral cortex -> striatum -> GPe -> STN -> GPi/SNpr -> thalamus -> cerebral cortex
- **Dopamine**: modulatory
  - Neurons in the striatum that receive dopamine inputs from SNpc can have different types of dopamine receptors 
  - excites D1 receptors (direct pathway)
  - inhibites D2 receptors (indirect pathway)
- **Hyberdirect**: May be important for action cancelling/stop signals for motor & cognitive programs

### Lesion

- Hyperkinetic: **Huntington’s** Disease
  - early HD: D2 receptor fail
- Hypokinetic: **Parkinson’s** Disease
  - PD Mechanism: loss of dopaminergic cells


