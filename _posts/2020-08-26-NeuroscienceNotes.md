---
layout: single
title: Neuroscience Notes - Molecular Biology
date: 2020-08-26 21:10:07
categories: "Notes"
tags:
- Notes
- Course
header:
  teaser: 
---

Some notes taken from Neuroscience course, part 1: Molecular Biology.

last update: Aug. 27, 2020

# Translation of Terms

| Eng                    | Chs           | Comment                         |
| ---------------------- | ------------- | ------------------------------- |
| nucleus                | 细胞核        |                                 |
| chromosome             | 染色体        |                                 |
| polymerase             | 聚合酶        |                                 |
| cytoplasm              | 细胞质        |                                 |
| ribosome               | 核糖体        |                                 |
| polymer                | 聚合物        |                                 |
| deoxyribonucleotide    | 脱氧核糖核酸  | DNA                             |
| nucleotide             | 核酸          |                                 |
| nomenclature           | 命名法/术语   | 3' 5'末端                       |
| phosphate              | 磷酸盐        |                                 |
| hydroxyl               | 羟基          | -OH                             |
| plasmid                | 质粒          |                                 |
| ATP                    | 三磷酸腺苷    | adenosine triphosphate          |
| phosphodiester         | 磷酸二酯      | breaking ~ bond -> energy       |
| polypeptide            | 多肽          |                                 |
| codon                  | 密码子        |                                 |
| protease               | 蛋白酶        |                                 |
| catalytic              | 催化的        |                                 |
| denaturation           | 使变性        | （化学意义上                    |
| Hybridization          | 杂交？        | 两条相配的DNA(RNA)链            |
| phage                  | 噬菌体        |                                 |
| molar                  | 摩尔的        |                                 |
| oligonucleotide        | 低聚核苷酸    | short (DNAs)                    |
| triphosphate           | 三磷酸盐      |                                 |
| southern/northern blot | 南/北方墨点法 |                                 |
| thyrotropin            | 促甲状腺素    |                                 |
| glutamate              | 谷氨酸        |                                 | 
| retroviruse            | 逆转录病毒    |                                 |
| agar                   | 琼脂          | 培养基材料                      |
| E. Coli                | 大肠杆菌      |                                 |
| mammalian              | 哺乳类动物的  |                                 |
| eukaryotic             | 真核的        |                                 |
| endonuclease           | 内切酶        | DNA clone                       |
| ligase                 | 连接酶        | DNA clone                       |
| primer                 | 引物          | e.g., in PCR                    |
| acrylamide             | 丙烯酰胺      | used in chemical DNA sequencing |
|dideoxy| 双脱氧法||

# DNA and RNA

感觉一节课把我整个高中学过的东西都讲完了。

- DNA strand: a sugar/phosphate backbone; polarity of (5' to 3')
- DNA can be double stranded *ds* or single stranded *ss*, stability
- **"Wastson-Crick" double helical conformation**, A=(2H)=T, C=(3H)=G
- Increase temperature or change pH(>12.5): *ds* -> *ss*
- DNA can be linear (humnan) or circular (bacterial plasmid)
- *ds*: oriented in an antiparallel configuration
- *sense strand*: seq from cDNA (DNA copy of an mRNA)
- Polymerize: dATP => DNA; rATP => RNA
- Synthesis always occurs from 5' to 3': Leading and Lagging (discontinuous) strand. 
- RNA is susceptible to degradation by nucleases
- DNAs & RNAs can be size-separated using gel electrophoresis: negatively charged

## References
[DNA复制Wiki](https://zh.wikipedia.org/wiki/DNA%E5%A4%8D%E5%88%B6)

![CentralDogma](https://upload.wikimedia.org/wikipedia/commons/6/68/Central_Dogma_of_Molecular_Biochemistry_with_Enzymes.jpg)

![reverseTranscription](https://upload.wikimedia.org/wikipedia/commons/d/dd/Extended_Central_Dogma_with_Enzymes.jpg)

![Translation](https://upload.wikimedia.org/wikipedia/commons/thumb/4/44/Protein_synthesis.svg/600px-Protein_synthesis.svg.png)

## Central Dogma ~~中央教条~~中心法则

- DNA ---(Transcription)--> RNA ---(Translation)--> Protein

| General       | Special       | Unknown           |
| ------------- | ------------- | ----------------- |
| DNA → DNA     | RNA → DNA     | protein → DNA     |
| DNA → RNA     | RNA → RNA     | protein → RNA     |
| RNA → protein | DNA → protein | protein → protein |

# DNA Hybridization

- *ds* DNA can find their matching partners even after cooling from *ss* DNAs
- DNA *"melting"* *ds* -> *ss*
- After cooling down, *ss* DNAs hybridize

## Factors => Melting Temperature $T_m$

- Salt concentration affect melting temp. Higher salt -> higher temp
- GC pair more stable because: G≡C, A=T
- DNA length: longer -> higher temp

## Factors => Hybridization

- Time of half of the molecules to reassociate $t_{1/2}$
- Inversely proportional to the molarity of those molecules
- Also influenced by temp, salt concentration, DNA length
- Optimal temp: ~25°C below the $T_m$
- Short DNAs (oligonucleotides, primers) => larger targets
- Approximate formula: $t_{1/2} = 1 day/[DNA]$, \[DNA\] in *picomolar(PM)*
  
## Applications

- PCR (“polymerase chain reaction”) ~~公主链接~~
- Production of cDNA
- DNA sequencing
- Immobilize nucleic acids
- Hybridization-mediated: RNA interference & CRISPR]

# Cloning DNAs

 - Clone DNA copies from mRNAs <= reverse transcriptase enzyme
 - cDNA was originally to study how each gene product functioned

## Reverse Transcriptase

> Reverse transcriptase is a primer-dependent polymerase that will synthesize DNA from the RNA template.

- Convert DNA into a *ds* DNA
- Analyzing DNAs is easier than RNAs
- DNA oligonucleotides are used to “prime” first strand cDNA synthesis. 
- Using random primers/oligo-d(T) as a primer

## Cloning

The first technique that provided an abundant source of a specific DNA 

![Cloning](https://upload.wikimedia.org/wikipedia/commons/a/a3/Gene_cloning.svg)

1. Foreign DNA + Cloning vector are cut by the same Restriction Endonucleases
2. They are then Covalently joined by DNA ligase
3. Put the Cloned DNA vector into E.Coli
   
- Most frequently used cloning vector: a small circular double-stranded DNA called a “plasmid”
- Cloning involves inserting a specific DNA fragment into a vector and propagating the recombinant plasmid
- Each bacterium should only contain a single recombinant plasmid 
- DNAs can be cleaved with restriction endonucleases recognizing specific nucleotide sequences and resulting in "sticky ends"
- E.g. EcoR1: GAATTC => G + AATTC & CTTAA + G

### Plasmids

> Plasmids are (small) extrachromosomal circular double-stranded DNAs that replicate independently from the host chromosome (E. coli).

> Plasmids used for cloning contain an origin of replication (“replicon”), selectable marker >(typically antibiotic resistance) and specific sites into which the DNA fragment can be inserted (a “multiple cloning site” or “polylinker”).

- **Selectable markers** typically (but not exclusively) a gene conferring antibiotic resistance. Two examples are ampicillin (“aminopenicillin”) resistance (AmpR) and kanamycin resistance (KanR). 
- **Replicon**: *(Origin of replication and control elements)* This is the region of the plasmid that is required for replication. Many commonly used plasmids have a “pUC” type origin permitting replication in E. Coli.
- **Ampicillin** inhibits bacterial cell wall synthesis, greatly slowing bacterial growth. The AmpR gene encodes beta-lactamase, an enzyme that cleaves a ring in the structure of ampicillin, thereby inactivating it.
- **Kanamycin** interacts with multiple ribosomal proteins and inhibits protein synthesis. The KanR gene encodes an aminophosphotransferase that phosphorylates kanamycin, preventing it from binding to the ribosomal proteins.
- **Multiple cloning site** *(MCS or polylinker)* This region contains multiple, adjacent “unique” sites of restriction endonuclease cleavage– providing many ways to introduce our “inserts”.

### Subcloning

- Molecular cloning refers to the insertion of a DNA fragment into a plasmid
- **Subcloning**: to “move” DNAs from one plasmid to another, in order to obtain expression tailored to their needs.

# PCR
Polymerase Chain Reaction 聚合酶链式反应

![PCRWiki](https://upload.wikimedia.org/wikipedia/commons/thumb/9/96/Polymerase_chain_reaction.svg/1253px-Polymerase_chain_reaction.svg.png)

- Enabling the mass production of DNA without to clone it
- Exponential amplification of DNA, doubling DNA mass with every cycle

## Process

PCR requires: 2 opposing small DNA primers

- Step 1, 2, 3 represents 3 thermal cycles
  
1. Denaturation: separate 2 strands via heating
2. Annealing: reduce temp to permit the hybridization of primers
3. Extension: raise temp to the optimal temp for heat-stable polymerase to synthesis DNA
4. DNA fragments doubled with each cycle

- Hybridization of short DNA => to target/template molecule
- Primers define the boundaries of the amplified region (5' -> 3')
- But we can also add additional nucleotides to the primers (e.g. sticky ends)
- Heat-stable enzyme: Taq(generate sticky ends), Pfu&Q5(don't)
- High Fidelity enzyme? fewer mistakes
- Designing PCR primers: need both upstream/sense and downstream/antisense 
- Can also targeting a specific region (Open reading frame)

# DNA Sequencing

A target of cloning is sequencing.

- Cycle sequencing
- Capillary sequencing

## Chemical Sequencing

- **Chemical modification** Use different chemical modifications of the DNA that permit the subsequent specific cleavage of these molecules: at G, G/A, T/C, C
- **Different-sized DNAs** Resulted in the generation of a distribution of different-sized DNAs
- **Radioactively labeled** DNAs were “end-labeled” using T4 polynucleotide kinase and a radiolabeled form of ATP (“gamma-32P-ATP”). 

## Sanger/Dideoxy Sequencing

![SangerSeqWiki](https://upload.wikimedia.org/wikipedia/commons/thumb/b/b2/Sanger-sequencing.svg/1920px-Sanger-sequencing.svg.png)

- Dideoxynucleotide(ddNTP): X 3' -OH => termination of growing DNA strand
- In the original format, each tube contained all 4 dNTPs but just one ddNTP
- The chain-termination happened infrequently => a broad rage of sized-DNAs
- All the DNAs are “single-stranded” and all must share the same 5’ end.

### Fluorescently Tagged Primers

> The 4 different “colors”, permitted loading the sample onto a single lane.

### "Shotgun" Sequencing

For very long sequences: human, mouse, etc.

- DNA samples is first broken into small pieces
- The data is assembled bioinformatically <= overlapping information