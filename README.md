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

The second approach involves a behavior defined by the configuration memory. The most used layer is called Virtual Reconfigurable Circuit (VCR) where semantic is not based on trees, but on Carthesian Genetic Programming (CGP).

CGP does not allow to easily define a Crossover operator with a semantic meaning, and usually the only defined operator is the Mutation.

This project propose a new reconfigurable layer, based on tree semantic, called *Programmable Expression Tree*.

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

MSB bits represent the first node. Depth-First ordering enable us to select sub-trees by using adjacent bits. This property simplify the Crossover implementation.

**Important**: Every binary string permutation is a valid tree because the implementation add an additional virtual level, made of user-defined operators, to avoid issues with operators such that expect a parameter (ex. sum)

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

At the startup, a random population is generated and stored by the Random Number Generator.

In the next stage, every individual is loaded into the programmable expression tree for its evaluation. The module EXP_TREE_FIT is responsible to provide input values (ex. variables values) to the tree and to compare results with expected values. After comparing all values, a fitness score is calculated and stored in the specific memory.

After that every individual has a fitness score, there is a selection phase. Tournament selection is implemented in the reference implementation, although other strategies are possibles.

Genetic Operators are applied to selected Individuals to build the next generation.

Memories are on-chip to minimize latency and maximize bandwidth. Memories that require parallel access, such as MEM_F containing the fitness score of individuals, are multi-port to furthermore improve performance.

A large performance improvement over standard (ex. CPU) architectures, is given by the data bus width that match the tree size.

Given that every node is able to perform TREE_NODE_OP different operations, log2(TREE_NODE_OP) bit are required.
A tree of TREE_DEPTH levels has (2^TREE_DEPTH)-1 nodes, so is fully defined by 

BUS_BIT_WIDTH = log2(TREE_NODE_OP) * ((2^TREE_DEPTH)-1) bit.

If a full tree is defined by 2044 bit, then the population memory, the random number generator, the selection module, etc.. handle data transfer and processing at 2044 bit, with an edge over general-purpose 64 bit architecture.

**Important**: Large trees require large FPGA devices. Single trees over multiple FPGA devices is not supported at this time.

## Random Number Generator

Genetic Programming requires a large number of random numbers to execute.
A pseudo-random number generator (PRNG) based on linear feedback shift register (LFSR) is implemented.

![random number generator](/docs/images/rnd_gen.png)

Multiple LFSR are instanced at the same time, to generate in a single clock cycle a full tree. To improve the random numbers quality a single bit is used from each LFSR.

The seed of each LFSR is loaded externally, to guarantee repeatability of experiments. After that the system is verified, is possible to change the implementation to use the PLL phase noise as a randomness source.

When using deep trees, a large quantity of LFSR is required. To reduce the resource usage, a FIFO memory can be implemented, keeping mind of the production / consumption rate of random numbers.

## Programmable Expressions Trees

The node is the foundation of the binary tree 

![high level node structure](/docs/images/exp_node_hl.png)

Example of a node with data width of 8 bit and 8 possible operators

node_op and node_value are generally exclusive, both are there to support reprogrammability based on the content of a memory.

The number of bit to represent values (children nodes, node and result) must be coherent to simplify the interconnection.

The following architecture shows a node able to do addition, subtraction (implemented with an adder) and multiplication.

![node RTL implementation](/docs/images/exp_node.png)

Multiplexers select the operation to execute, ideally one for each output bit.

Bus size grow exponentially to the tree depth. In the following image, a three levels expression tree and the configuration memory. Terminal Virtual nodes are connected to the constant zero value.

![high level programmable expression tree structure](/docs/images/exp_tree_hl.png)

## Fitness Computation Unit

Fitness is calculated according to the following formula: 

fitness = SumK(abs(var_outputK - resultK))


![high level fitness unit](/docs/images/exp_tree_fit.png)

![pipelined programmable expression tree](/docs/images/exp_tree_hp.png)

EXP_MEM is a "Pipelined Content-Replaceable Memory"

![expression tree waveform](/docs/images/exp_tree_waveform.png)

