## Cycle-accurate simulator for a 5-stage pipelined MIPS processor in C++

The simulator supports a subset of the MIPS instruction set and should model the execution of each instruction with cycle accuracy.

The MIPS program is provided to the simulator as a text file “imem.txt” file which is used to initialize the Instruction Memory. Each line of the file corresponds to a Byte stored in the Instruction Memory in binary format, with the first line at address 0, the next line at address 1 and so on. Four contiguous lines correspond to a whole instruction. Note that the words stored in memory are in “Big-Endian” format, meaning that the most significant byte is stored first.   

The Data Memory is initialized using the “dmem.txt” file. The format of the stored words is the same as the Instruction Memory. As with the instruction memory, the data memory addresses also begin at 0 and increment by one in each line.  

The instructions that the simulator supports and their encodings are shown in Table 1. Note that all instructions, except for “halt”, exist in the MIPS ISA. The MIPS Green Sheet defines the semantics of each instruction. 

| Instruction | Format            | OpCode (hex)  | Funct. (hex) |
|-------------|:------------------|---------------|-------------:|
| Addu        | R-Type (ALU)      |     00        |    21 
| Subu        | R-Type (ALU)      |     00        |    23 
| Lw          | I-Type (Memory)   |     23        |    - 
| Sw          | I-Type (Memory)   |     2B        |    - 
| Beq         | I-Type (Control)  |     04        |    - 
| Halt        | Custom instruction|     FF        | 

## Special Note about Beq Instruction: 

We will assume that the beq (branch-if-qual) instruction operates like a bne (branch-if-not-equal) instruction. In other words, we will assume that the beq jumps to the branch address if 𝑅[𝑟𝑠] ≠ 𝑅[𝑟𝑡] and jumps to PC+4 otherwise, i.e., if 𝑅[𝑟𝑠] = 𝑅[𝑟𝑡].   
 
(Note that a real beq instruction would operate in the opposite fashion, that is, it will jump to the branch address if 𝑅[𝑟𝑠] = 𝑅[𝑟𝑡] and to PC+4 otherwise. The reason we had to make this modification for this lab is because to implement loops we actually need the bne instruction.) 

## 5 stages: 
 
1. Fetch (IF): fetches an instruction from instruction memory. Updates PC.  

2. Decode (ID/RF): reads from the register RF and generates control signals required in subsequent stages. In addition, branches are resolved in this stage by checking for the branch condition and computing the effective address.  

3. Execute (EX): performs an ALU operation.  

4. Memory (MEM): loads or stores a 32-bit word from data memory. 

5. Writeback (WB): writes back data to the RF. 

Each pipeline stages takes inputs from flip-flops. The input flip-flops for each pipeline stage are described in the tables below. 

IF Stage Input Flip-Flops 
| Flip-Flop Name | Bit-width | Functionality | 
|----------------|:----------|--------------:|
| PC | 32 | Current value of PC | 
|nop |1|If set, IF stage performs a nop |
 
ID/RF Stage Input Flip-Flops 
|Flip-Flop Name| Bit-width | Functionality| 
|--------------|:-----------|-------------:|
|Instr |32 |32-bit instruction read from IMEM |
|nop |1 |If set, ID/RF stage performs a nop |

  EX Stage Input Flip-Flops 
|Flip-Flop Name |Bit-width | Functionality| 
|---------------|:---------|-------------:|
|Read_data1, Read_data2| 32| 32-bit data values read from RF| 
|Imm |16| 16-bit| immediate for I-Type instructions. Don’t care for R-type instructions| 
|Rs, Rt| 5 |Addresses of source registers rs, rt. Note that these are defined for both R-type and I-type instructions| 
|Wrt_reg_addr| 5| Address of the instruction’s destination register. Don’t care if the instruction does not update RF| 
|alu_op| 1 |Set for addu, lw, sw; unset for subu| 
|is_I_type |1| Set if the instruction is an i-type instruction|  
|wrt_enable |1| Set if the instruction updates RF |
|rd_mem, wrt_mem| 1| rd_mem is set for lw instruction and wrt_mem for sw instructions|  
|nop |1 |If set, EX stage performs a nop| 
 
MEM Stage Input Flip-Flops 
|Flip-Flop Name |Bit-width | Functionality| 
|---------------|:--------|--------------:|
|ALUresult |32 |32-bit ALU result. Don’t care for beq instructions| 
|Store_data |32| 32-bit value to be stored in DMEM for sw instruction. Don’t care otherwise| 
|Rs, Rt |5| Addresses of source registers rs, rt. Note that these are defined for both R-type and I-type instructions| 
|Wrt_reg_addr |5| Address of the instruction’s destination register. Don’t care if the instruction does not update RF |
|wrt_enable |1| Set if the instruction updates RF| 
|rd_mem, wrt_mem| 1| rd_mem is set for lw instruction and wrt_mem for sw instructions|  
|nop |1 |If set, MEM stage performs a nop| 
 
WB Stage Input Flip-Flops 
|Flip-Flop Name |Bit-width | Functionality |
|---------------|:---------|---------------:|
|Wrt_data |32| 32-bit data to be written back to RF. Don’t care for sw and beq instructions |
|Rs, Rt |5| Addresses of source registers rs, rt. Note that these are defined for both R-type and I-type instructions |
|Wrt_reg_addr |5 |Address of the instruction’s destination register. Don’t care if the instruction does not update RF |
|wrt_enable |1| Set if the instruction updates RF |
|nop| 1| If set, MEM stage performs a nop |

## Dealing with Hazards 

1. RAW Hazards: RAW hazards are dealt with using either only forwarding (if possible) or, if not, using stalling + forwarding. 

2. Control Flow Hazards: We assume that branch conditions are resolved in the ID/RF stage of the pipeline. This processor deals with beq instructions as follows: 
  
a. Branches are always assumed to be NOT TAKEN. That is, when a beq is fetched in the IF stage, the PC is speculatively updated as PC+4. 
  
b. Branch conditions are resolved in the ID/RF stage. Testbenches will ensure that every beq instruction has no RAW dependency with its previous two instructions. In other words, NO RAW hazards for branches! 
  
c. Two operations are performed in the ID/RF stage: (i) Read_data1 and Read_data2 are compared to determine the branch outcome; (ii) the effective branch address is computed. 
  
d. If the branch is NOT TAKEN, execution proceeds normally. However, if the branch is TAKEN, the speculatively fetched instruction from PC+4 is quashed in its ID/RF stage using the nop bit and the next instruction is fetched from the effective branch address. Execution now proceeds normally.

## The nop bit 

The nop bit for any stage indicates whether it is performing a valid operation in the current clock cycle. The nop bit for the IF stage is initialized to 0 and for all other stages is initialized to 1. (This is because in the first clock cycle, only the IF stage performs a valid operation.) 

In the absence of hazards,, the value of the nop bit for a stage in the current clock cycle is equal to the nop bit of the prior stage in the previous clock cycle. 

However, the nop bit is also used to implement stall that result from a RAW hazard or to quash speculatively fetched instructions if the branch condition evaluates to TAKEN. 

## The HALT Instruction 
 
The halt instruction is a “custom” instruction we introduced so you know when to stop the simulation. When a HALT instruction is fetched in IF stage at cycle N, the nop bit of the IF stage in the next clock cycle (cycle N+1) is set to 1 and subsequently stays at 1. The nop bit of the ID/RF stage is set to 1 in cycle N+1 and subsequently stays at 1. The nop bit of the EX stage is set to 1 in cycle N+2 and subsequently stays at 1. The nop bit of the MEM stage is set to 1 in cycle N+3 and subsequently stays at 1. The nop bit of the WB stage is set to 1 in cycle N+4 and subsequently stays at 1.  
 
At the end of each clock cycle the simulator checks to see if the nop bit of each stage is 1. If so, the simulation halts. 

## Output 
 
Simulator will output the values of all flip-flops at the end of each clock cycle. Further, when the simulation terminates, the simulator also outputs the state of the RF and Dmem.
