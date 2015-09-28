# Project 4: Processor Design
Based on original spec by Ben Sussman and Brian Zimmer, and modified spec of Albert Chae, Paul Pearce, Noah Johnson, Justin Hsia, Conor Hughes, Anirudh Todi, Ian Vonseggern, Sung Roa Yoon, and Alan Christopher.
Much thanks to Conor Hughes for an excellent assembler and autograder.

This project spec is ridiculously long, but don't fret! We've spelled out many things in excruciating detail, so if you just take things one-by-one, it won't be as bad as it looks.

We are also providing a set of abridged project notes to look at. These will NOT substitute for reading through the actual project specs, but can be used as a quick reference later on.

# Overview
- MAKE SURE TO CHECK YOUR CIRCUITS WITH THE GIVEN HARNESSES TO SEE IF THEY FIT! YOU WILL FAIL ALL OUR TESTS IF THEY DO NOT.  (This also means that you should not be moving around given inputs and outputs in the circuits).
- This is an INDIVIDUAL project.
- Tarball of sample tests for a completed CPU have been included in the proj4StartKit. Look at the README file for usage info. I recommend running the sample tests locally. These tests are NOT comprehensive, you will need to do further testing on your own.
- You are allowed to use any of Logisim's built-in blocks for all parts of this project.
- Save often. Use git for version control! Logisim can be buggy and the last thing you want is to lose some of your hard work. There are students every semester who has to start over large chunks of their projects this way.
- It is HIGHLY ADVISED that you use tunneling in your design. It will make writing and debugging your circuit much easier!

In this project you will be using Logisim to create a 16-bit two-cycle processor. It is similar to MIPS, except that both the datapath and the instructions are 16-bits wide, it has only 4 registers, and memory addresses represent 16-bit words instead of 8-bit bytes (word-addressed instead of byte-addressed).

Please read this document CAREFULLY as there are key differences between the processor we studied in class and the processor you will be designing for this project.

Before you begin, pull the start kit to an initialized git repo for proj4:

    $ git pull ~cs61c/proj/04 master

## Pipelining
Your processor will have a 2-stage pipeline:
1. Instruction Fetch: An instruction is fetched from the instruction memory.
2. Execute: The instruction is decoded, executed, and committed (written back). This is a combination of the remaining stages of a normal MIPS pipeline.

You should note that data hazards do NOT pose a problem for this design, since all accesses to all sources of data happens only in a single pipeline stage. However, there are still control hazards to deal with. Our ISA does not expose branch delay slots to software. This means that the instruction immediately after a branch or jump is not necessarily executed if the branch is taken. This makes your task a bit more complex. By the time you have figured out that a branch or jump is in the execute stage, you have already accessed the instruction memory and pulled out (possibly) the wrong instruction. You will therefore need to "kill" instructions that are being fetched if the instruction under execution is a jump or a taken branch. Instruction kills for this project MUST be accomplished by MUXing a nop into the instruction stream and sending the nop into the Execute stage instead of using the fetched instruction. Notice that 0x0006 is a nop instruction; please use this, as it will simplify grading and testing. You should only kill if a branch is taken (do not kill otherwise), but do kill on every type of jump.

Because all of the control and execution is handled in the Execute stage, your processor should be more or less indistinguishable from a single-cycle implementation, barring the one-cycle startup latency and the branch/jump delays. However, we will be enforcing the two-pipeline design. If you are unsure about pipelining, it is perfectly fine (maybe even recommended) to first implement a single-cycle processor. This will allow you to first verify that your instruction decoding, control signals, arithmetic operations, and memory accesses are all working properly. From a single-cycle processor you can then split off the Instruction Fetch stage with a few additions and a few logical tweaks. Some things to consider:

- Will the IF and EX stages have the same or different PC values?
- Do you need to store the PC between the pipelining stages?
- To MUX a nop into the instruction stream, do you place it before or after the instruction register?
- What address should be requested next while the EX stage executes a nop? Is this different than normal?

You might also notice a bootstrapping problem here: during the first cycle, the instruction register sitting between the pipeline stages won't contain an instruction loaded from memory. How do we deal with this? It happens that Logisim automatically sets registers to zero on reset; the instruction register will then contain a pseudo-nop (In general 0x0000 is not a nop, but 0 + 0 == 0, so it works out when all the registers contain zero). We will allow you to depend on this behavior of Logisim. Remember to go to Simulate --> Reset Simulation (Ctrl+R) to reset your processor.

## Deliverables
Approach this project like you would any coding assignment: construct it piece by piece and test each component early and often!

Tidyness and readability will be a large factor in grading your circuit if there are any issues, so please make it as neat as possible! If we can't comprehend your circuit, you will probably receive no partial credit.
logisim divider
1) Register File [show]
2) Arithmetic Logic Unit (ALU) [show]
3) Processor [show]
3a) Data Memory [show]
3b) Output Devices [show]
4) Test Code [show]
*) Extra for Experts: NOT EXTRA CREDIT [show]
Updates | Overview | Deliverables | ISA | Logisim | Testing | Submission

## Instruction Set Architecture (ISA)
You will be implementing a simple 16-bit processor with four registers ($r0-$r3). It will have separate data and instruction memory. **Because this is a 16-bit architecture, our words are 16 bits wide, unlike the 32-bit MIPS ISA we have been studying in class. For the remainder of this document, a WORD refers to 16 bits.** Each of the four registers is big enough to hold ONE word.

**IMPORTANT**: Because of the limitations of Logisim (and to make things simpler), our memories will be word-addressed (16 bits), unlike MIPS, which is byte-addressed (8 bits).

The instruction encoding is given below. Your processor will pull out a 16-bit value from instruction memory and determine the meaning of that instruction by looking at the opcode (the top four bits, which are bits 15-12). If the instruction is an R-type (i.e. opcode == 0), then you must also look at the funct field.

Notice how we do not use all 64 R-type instructions. Your project only has to work on these specified instructions. This way the project is shorter and easier.

    15-12       11  10      9   8       7   6       5   4   3   2   1   0
    0           rs          rt          rd                  funct                       See R-type Instructions
    1                            target address                                                 jal into $r3
    2                            target address                                                 j: jump to target add.
    3           rs               unused                                                        jr:   PC = $rs
    4           rs          rt                  offset (signed)                             beq: branch if equal
    5           rs          rt                  immediate (signed)                          addi: $rt = $rs + imm
    6           rs          rt                  immediate (unsigned)                        andi: $rt = $rs & imm
    7           rs          rt                  immediate (unsigned)                        ori:  $rt = $rs | imm
    8           rs          rt                  offset (signed)                             bne: branch if not equal
    9           rs          rt                  immediate (signed)                          sw:   MEM[$rs+imm] = $rt
    10          rs          rt                  immediate (unsigned)                        lui:  $rt = imm << 8
    11          rs          rt                  immediate (signed)  lw:                     $rt = MEM[$rs + imm]
    12          rs          rt                  immediate (unsigned)                        disp: DISP[imm] = $rs

    R-Type Instructions
        funct   Instruction
        0   add:  $rd = $rs + $rt
        1   sub:  $rd = $rs - $rt
        2   sllv: $rd = $rs << $rt
        3   srlv: $rd = $rs >> $rt (zero extend)
        4   srav: $rd = $rs >> $rt (sign extend)
        5   slt:  $rd = ($rs < $rt) ? 1 : 0
        6   or:   $rd = $rs | $rt
        7   and:  $rd = $rs & $rt
        8   addp8: $rd = {$rs[15:8] + $rt[15:8] , $rs[7:0] + $rt[7:0] }
        9   subp8: $rd = {$rs[15:8] - $rt[15:8] , $rs[7:0] - $rt[7:0] }

Some specifics on selected instructions:
### Shifting
We will not test shift amounts greater than 15; behavior in this case is undefined.

### Jumping
- The argument to the jump and jal instructions is a pseudoabsolute address, similar to MIPS. The target address is an unsigned number representing the lower 12 bits of the next instruction to be executed. The upper four bits are taken from the current PC. We do NOT concatenate any zeroes to the bottom of our address like we would in MIPS. This is because our processor is word-addressed, so every possible address holds a valid 16-bit instruction:

            nextPC = ("currPC" & 0xF000) | target address

- Note that "currPC" is the PC of the jump instruction itself. We will be assuming small program sizes (< 210 instructions) and will not be strictly enforcing currPC+1 as it would be in MIPS.
- Note that you should kill the next instruction after a jump or jal even if that is the instruction you are going to be jumping to.
- On a jal the address of the next instruction should be written into $r3. This is what we mean by "link into $r3".

### Branching
- The argument to the beq and bne instructions is a signed offset relative to the next instruction to be executed if we don't take the branch, which is similar to MIPS. Note that the address of this next instruction is PC+1 rather than PC+4 because our processor is word-addressed. Here, currPC means the address of the branch instruction. We can write beq as the following:

            if $rs == $rt
                    nextPC = currPC+1 + offset
            else
                    increment PC like normal        

- Think! There's a reason we write "increment PC like normal" here instead of just "currPC+1".
- The bne instruction differs only by the test in the if statement: replace the == with !=.
- Note that you should not kill the next instruction if the branch is not taken. If the branch is taken you should always kill.

### Immediates
Note that the immediate field is only 8 bits wide, so we must perform some kind of extension on it before passing it to the ALU. If an immediate is supposeed to be unsigned, be sure to zero-extend it. If an immediate is signed, be sure to sign-extend it.

### Functions not in MIPS Greensheet
You may have noticed that there are a few functions not defined in the Greensheet, namely addp8 and subp8. We've all seen these types of SIMD instructions in different architectures (if you haven't, I'd be curious to see just how much fun you had with proj3), and we thought it might be fun for you guys to try implementing them! 

## Logisim Notes
It is strongly recommended that you download and run Logisim on your local machine while developing your processor to avoid overwhelming the instructional machines. Though Logisim is relatively stable compared to prior semesters, it is still recommended that you save often and also make backup copies of your .circ files early and often. The official version of Logisim we will be using for evaluation is v2.7.1.

If you are having trouble with Logisim, **RESTART IT and RELOAD** your circuit! Don't waste your time chasing a bug that is not your fault. However, if restarting doesn't solve the problem, it is more likely that the bug is a flaw in your project. Please post to Piazza about any crazy bugs that you find and we will investigate.

### Things to Look Out For
- Do NOT gate the clock! This is very bad design practice when making real circuits, so we will discourage you from doing this by heavily penalizing your project if you gate your clock.
- BE CAREFUL with copying and pasting from different Logisim windows. Logisim has been known to have trouble with this in the past.
- When you import another file (Project --> Load Library --> Logisim Library...), it will appear as a folder in the left-hand viewing pane.
- Changing attributes before placing a component changes the default settings for that component. So if you are about to place many 16-bit pins, this might be desireable. If you only want to change that particular component, place it first before changing the attributes.
- When you change the inputs & outputs of a sub-circuit that you have already placed in main, Logisim will automatically add/remove the ports when you return to main and this sometimes shifts the block itself. If there were wires attached, Logisim will do its automatic moving of these as well, which can be extremely dumb in some cases. Before you change the inputs and outputs of a block, it can sometimes be easier to first disconnect all wires from it.
- Error signals (red wires) are obviously bad, but they tend to appear in complicated wiring jobs such as the one you will be implementing here. It's good to be aware of the common causes while debugging:
![circuit](/img/error_wire)


### Logisim's Combinational Analysis Feature
Logisim offers some functionality for automating circuit implementation given a truth table, or vice versa. Though not disallowed (enforcing such a requirement is impractical), use of this feature is discouraged. Remember that you will not be allowed to have a laptop running Logisim on the final.


## Testing
Once you've implemented your processor, you can test its correctness by writing programs to run on it! First, try this simple program as a sanity check: halt.s. This program loads the same immediate into two different registers using lui/ori and then branches back one instruction (offset = -1) if these registers are equal.

         Assembly:              Binary:
         ========               ======
         lui $r0, 0x33          A033
         ori $r0, $r0, 0x44     7044
         lui $r1, 0x33          A133
         ori $r1, $r1, 0x44     7544
    self: beq $r0, $r1, self     41FF

For practice, verify that the assembly on the left matches the translated binary on the right. This program effectively "halts" the processor by putting it into an infinite loop, so you can observe the outputs as well as memory and register state. Of course, you could do this "halt" with only the beq line, but it is very important that you test your lui/ori or the programs we will use during grading will not work.

To test your processor, open the cpu-harness.circ. Find the Instruction Memory RAM and right click --> Load Image... Select the assembled program (.hex file - see details on the Assembler below) to load it and then start clock ticks.

As described in the Deliverables, you are REQUIRED to write and submit two sample programs to test your processor (octal.s and mod.s), but you should also write others to test all your instructions.

Remember: *Debugging Sucks*. Testing Rocks.

### Assembler
We've provided a basic assembler to make writing your programs easier so you can use assembly instead of machine code. You should try writing a few by hand before using this, mainly because it's good practice and makes you feel cooler. This assembler.py supports all of the instructions for your processor.

The assembler is included in the start kit (one you pull from the repo with earlier instruction) or can be downloaded from the link above. The standard assembler is a work in progress, so please report bugs to Piazza!

The assembler takes files of the following form (this is halt.s, which is included in the start kit):

         #Comments are great!
         lui $r0, 0x33          #A033
         ori $r0, $r0, 0x44     #7044
         lui $r1, 0x33          #A133
         ori $r1, $r1, 0x44     #7544
    self: beq $r0, $r1, self     #41FF

Anywhere a register is required, it must be either $r0, $r1, $r2, or $r3. Commas are optional but the '$' is not. '#' starts a comment. The assembler can be invoked with the following command:

    $ python assembler.py input.s [-o output.hex]

The output file is input.hex if not explicitly set - that is, the same name as the input file but with a .hex extension. Use the -o option to change the output file name arbitrarily.