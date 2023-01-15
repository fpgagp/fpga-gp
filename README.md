# fpga-gp

High-performance Genetic Programming on FPGA 

## Introduction

This project provides a full, high-performance, Genetic Programming framework for FPGA devices.

Symbolic Regression implementation is provided as benchmark and reference.## 

## Algorithm

![gp algorithm](/docs/images/gp_algorithm.png)

#### Implementation

In literature, Genetic Programming implementations follow two main different approaches:

- Bitstream manipulation

- Reconfigurable layers    

The first approach is now abandoned because of undocumented bitstreams and the possibility of physical damage to the FPGA device in case of errors.

The most used layer is called Virtual Reconfigurable Circuit (VCR) where semantic is not based on trees, but on Carthesian Genetic Programming (CGP).

CGP does not allow to easily define a Crossover operator with a semantic meaning, and usually the only defined operator is the Mutation.

This project propose a new reconfigurable layer, based on tree semantic, called *Programmable Expression Tree*.

## High-Level Architecture

The algorithm is implemented inside the GP_CORE block, is written in VHDL 2008 and expose an AXI interface to interconnect with the soft core that manage the communication with the Human Machine Interface (HMI).


![high level architecture](/docs/images/hl_arch.png)

A system can contain one or more GP_CORE blocks and a soft core that efficiently implement standard such as Ethernet, RS232, USB, etc..

Multiple instances running independently enable a parallel exploration based on islands.



![gp_core high level architecture](/docs/images/gp_core_hl_arch.png)


| Module       | Description                            |
| ------------ | -------------------------------------- |
| RND_GEN      | Random number generator                |
| EXP_TREE     | Programmable expression tree           |
| EXP_TREE_FIT | Tree manager and fitness evaluation    |
| SEL          | Selection                              |
| GP_OP        | Copy, Mutation and Crossover operators |
| CONTROL FSM  | Control Finite State Machine           |



| Memory | Description                |
| ------ | -------------------------- |
| MEM_P  | Population                 |
| MEM_F  | Fitness of individuals     |
| MEM_N  | New Generation             |
| MEM_L  | Lookup to evaluate fitness |



## Expression Trees

Functions, individuals of the population, are semantically represented as binary tree.

The expression (2-1)+(X*Y) for example is represented by the following tree:

At physical level, every tree is a a string of bits that encodes a depth-first traversing binary tree.



![trasversing order](/docs/images/exp_tree_order.png)

Depth First trasversing order



![tree string](/docs/images/exp_tree_string.png)

Linearly represented tree



Assuming the following node encoding:

| Function | Encoding |
| -------- | -------- |
| +        | 0        |
| *        | 1        |
| X        | 2        |
| Y        | 3        |
| -        | 4        |
| R2       | 5        |
| R1       | 6        |
| R0       | 7        |

The previous function is binary encoded as

![binary tree string](/docs/images/exp_tree_string_bin.png)

MSB bits represent the first node

**Important**: Every binary string permutation is a valid tree because the implementation add an additional virtual level, made of user-defined operators, to avoid issues with operators such that expect a parameter (ex. sum)
