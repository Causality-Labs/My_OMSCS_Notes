
#### Pipelining

- Given a certain latency, the idea is that the first unit of work is achieved in the same period of time, but successive units continue to be completed thereafter. (Oil pipeline vs. traveling with a bucket)

#### Pipelining in a Processor

Each logical phase of operation in the processor (fetch, decode, ALU, memory, write) operates as a pipeline in which successive instructions move through each phase immediately behind the current instruction, instead of waiting for the entire instruction to complete.

![Pipelining in a Processor](https://i.imgur.com/0m5vXEf.png)


#### Pipeline CPI (Cycles Per Instruction)

- If there an instruction has to wait at a pipeline stage, all the instructions ahead of it proceed through the pipeline, all the instructions behind it are also stalled. 
- This is called a delay in the pipeline. The pipeline ahead of the delay will not have instructions to execute as the pipeline empties and the instructions behind the delay will be stalled. As the number of delays increase through the pipeline, the CPI will increase.


####  Processor Pipeline Stalls

In the example below, we see one example of a pipeline stall in a processor in a program like this:

```mipsasm

LW  R1, ...

ADD R2, R1, 1

ADD R3, R2, 1

```

  

![Processor Pipeline Stalls](https://i.imgur.com/E21sE3o.png)

  

The second instruction depends on the result of the first instruction, and must wait for it to complete the ALU and MEM phases before it can proceed. Thus, the CPI is actually much greater than 1.

#### Processor Pipeline Stalls and Flushes

```mipsasm

LW  R1, ...

ADD R2, R1, 1

ADD R3, R2, 1

JUMP ...

SUB ... ...

ADD ... ...

SHIFT

```
  

In this case, we have a jump, but we don't know where yet (maybe the address is being manipulated by previous instructions). Instructions following the jump instruction are loaded into the pipeline. When we get to the ALU phase, the `JUMP` will be processed, but this means the next instruction is not the `SUB ... ...` or `ADD ... ...`. These instructions are then flushed from the pipeline and the correct instruction (`SHIFT`) is fetched.


![Processor Pipeline Stalls and Flushes](https://i.imgur.com/5S9f2kB.png)


#### Control Dependencies

For the following program, the `ADD`/`SUB` instructions have a control dependence on `BEQ`. Similarly, the instructions after `label:` also have a control dependence on `BEQ`.
  

```mipsasm

    ADD R1, R1, R2

    BEQ R1, R3, label

    ADD R2, R3, R4

    SUB R5, R6, R8

label:

    MUL R5, R6, R8

```

  

We estimate that:
- 20% of instructions are branch/jump
- slightly more than 50% of all branch/jump instructions are taken

On a 5-stage pipeline, CPI = 1 + 0.1*2 = 1.2 (10% of the time (50% * 20%) an instruction spends two extra cycles)

With a deeper pipeline (more stages), the number of wasted instructions increases.

**Overall CPI = CPI of program + % of instructions mispredicted * penalty for misprediction**

#### Data Dependencies

```mipsasm

    ADD R1, R2, R3

    SUB R7, R1, R8

    MUL R1, R5, R6

```

This program has 3 dependencies:
1. Lines 1 and 2 have a RAW (Read-After-Write) dependence on R1 (also called Flow dependence, or TRUE dependence). The `ADD` instruction must be completed before the `SUB` instruction.
2. Lines 1 and 3 have a WAW (Write-After-Write) dependence on R1 with the `MUL` instruction in which the `ADD` must complete first, else R1 is overwritten with an incorrect value. This is also called an Output dependence.
3. Lines 2 and 3 have a WAR (Write-After-Read) dependence on R1 in that the `SUB` instruction must use the value of R1 before the `MUL` instruction overwrites it. This is also called an Anti-dependence because it reverses the order of the flow dependence.
WAW and WAR dependencies are also called "False" or "Name" dependencies. RAR (Read-After-Read) dependencies do not matter since the value could not have changed in-between and is thus safe to read.



#### Dependencies and Hazards

- Dependence - property of the program alone.
- Hazard - when a dependence results in incorrect execution.
For example, in a 5-stage pipeline, a dependency that is 3 instructions apart may not cause a hazard, since the result will be written before the dependent instruction reads it.


#### Handling of Hazards 

First, detect hazard situations. Then, address it by:
1. Flush dependent instructions
2. Stall dependent instruction
3. Fix values read by dependent instructions

Must use flushes for control dependencies, because the instructions that come after the hazard are the wrong instructions.

For data dependence, we can stall the next instruction, or fix the instruction by forwarding the value to the correct stage of the pipeline (e.g. "keep" the value inside the ALU stage for the next instruction to use). Forwarding does not always work, because the value we need is produced at a later point in time. In this cases we must stall.


#### How many Stages

Every pipeline should be achieving the required CPI, it is different for every pipeline. When more stages are added to a pipeline: 1. There are more hazards introduced into the pipeline. 2. The penalty for hazards increases. 3. There is less work for each stage, so the cycle time can be smaller. The Iron Law of Performance says: The number of stages must be balanced between the CPI and the cycle time. If only performance is considered, 30 ­ 40 stages is ideal. But if power consumption is also considered, the ideal pipeline is 10 ­ 15 stages.

![[Screenshot 2026-01-18 131600.png]]