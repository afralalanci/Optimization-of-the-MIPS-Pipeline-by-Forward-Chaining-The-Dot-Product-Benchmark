# MIPS Pipeline Optimization with Forwarding

This project implements and evaluates a 5-stage MIPS pipeline in SystemVerilog using a vector dot-product benchmark.

The project compares a baseline pipeline that resolves data hazards by stalling against an optimized pipeline that uses forwarding, also known as data bypassing. Simulation results show that forwarding significantly improves execution performance.

## GitHub Description

SystemVerilog implementation of a 5-stage MIPS pipeline executing a vector dot-product benchmark. Compares a stall-based baseline with an optimized forwarding datapath, reducing execution time from 157 to 103 cycles while preserving the correct result of 138 (0x8A).

## Project Objectives

- Implement a functional 5-stage MIPS datapath.
- Execute a vector dot-product benchmark.
- Detect and resolve data hazards.
- Compare strict stalling with forwarding-based hazard resolution.
- Verify the final result using simulation waveforms.
- Measure the performance improvement achieved through data bypassing.

## MIPS Pipeline Stages

The processor uses the classic five-stage pipeline:

```text
1. IF  - Instruction Fetch
2. ID  - Instruction Decode
3. EX  - Execute
4. MEM - Memory Access
5. WB  - Write Back
```

## Dot-Product Benchmark

The benchmark calculates the dot product of two vectors:

```text
A = (0, 2, 0, 0, 8, 8, 6, 5, 3)

B = (3, 5, 3, 3, 1, 1, 9, 8, 6)
```

The mathematical operation is:

```text
result = A[0] * B[0] +
         A[1] * B[1] +
         ...
         A[8] * B[8]
```

The expected result is:

```text
138 decimal = 0x8A hexadecimal
```

## Benchmark Program

The MIPS instruction sequence used for the dot product is:

```asm
dot_product:
    addu  $r1, $r0, $r0       # result = 0

loop:
    beq   $r7, $r0, done      # branch if counter is zero
    lw    $r2, 0($r3)         # load an element from vector A
    lw    $r4, 0($r5)         # load an element from vector B
    mul   $r2, $r2, $r4       # multiply the loaded elements
    addu  $r1, $r1, $r2       # accumulate the product
    addiu $r3, $r3, 4         # advance vector A pointer
    addiu $r5, $r5, 4         # advance vector B pointer
    addiu $r7, $r7, -1        # decrement loop counter
    j     loop

done:
    jr    $r31
```

## Baseline Pipeline

The baseline implementation uses a strict hazard unit:

```text
hazard_unit_strict53
```

This unit monitors destination registers in the Execute, Memory, and Write Back stages.

If the Decode stage requires a register that is still being modified by an older instruction, the processor:

- Freezes the Program Counter.
- Prevents incorrect instruction progression.
- Inserts bubbles into the pipeline.
- Waits until the required value is written back.

This approach guarantees correct execution but introduces a large number of stall cycles.

## Forwarding Optimization

The optimized implementation uses:

```text
forwarding_unit53
hazard_unit_fwd53
```

Instead of waiting for a value to be written back to the register file, forwarding sends the result directly from later pipeline stages to the ALU inputs.

The forwarding paths are:

```text
EX/MEM pipeline register -> ALU input
MEM/WB pipeline register -> ALU input
```

Multiplexers select whether the ALU receives:

```text
1. The value read from the register file
2. A forwarded value from the EX/MEM stage
3. A forwarded value from the MEM/WB stage
```

The optimized hazard unit only stalls when a stall is truly necessary:

- Load-use hazards
- Branch control hazards

Most arithmetic data dependencies are resolved without stalling.

## Forwarding Concept

Without forwarding:

```text
Instruction 1: produces a value
Instruction 2: waits for Write Back
Instruction 2: uses the value
```

With forwarding:

```text
Instruction 1: produces a value
Instruction 2: receives the value directly from a pipeline register
```

Forwarding reduces the time that dependent instructions spend waiting for register-file updates.

## Simulation Results

The design was simulated using Icarus Verilog through EDA Playground. VCD waveform dumping was used to inspect the processor cycle by cycle.

### Baseline Execution

```text
Final result: 138 decimal
Final result: 0x8A hexadecimal
Completion cycle: 157
```

### Optimized Execution with Forwarding

```text
Final result: 138 decimal
Final result: 0x8A hexadecimal
Completion cycle: 103
```

## Performance Comparison

```text
Baseline pipeline:       157 cycles
Forwarding pipeline:     103 cycles
Cycle reduction:          54 cycles
```

The percentage reduction in execution cycles is:

```text
(157 - 103) / 157 * 100 = 34.39%
```

The optimized pipeline completes the benchmark 54 cycles earlier than the baseline implementation.

The report also describes this improvement as a 54-cycle performance improvement.

## Verification

Both processor versions produce the same correct dot-product result:

```text
Expected result: 138 decimal
Hardware result: 138 decimal
```

The waveform analysis confirms that:

- The baseline processor inserts stalls for data hazards.
- The forwarding processor bypasses available results.
- Load-use hazards are still handled correctly.
- The final value is written to register `$r1`.
- Forwarding preserves functional correctness while reducing execution time.

## Example Simulation Output

Baseline pipeline:

```text
Cycle: 155 | PC: 00000028 | r1: 120 | r3: 32 | r5: 96 | r7: 1
Cycle: 156 | PC: 0000002c | r1: 120 | r3: 32 | r5: 96 | r7: 1
Cycle: 157 | PC: 00000030 | r1: 138 | r3: 32 | r5: 96 | r7: 1
Cycle: 158 | PC: 00000034 | r1: 138 | r3: 36 | r5: 96 | r7: 1
```

Optimized pipeline with forwarding:

```text
Cycle: 101 | PC: 00000028 | r1: 120 | r3: 32 | r5: 96 | r7: 1
Cycle: 102 | PC: 0000002c | r1: 120 | r3: 32 | r5: 96 | r7: 1
Cycle: 103 | PC: 00000030 | r1: 138 | r3: 32 | r5: 96 | r7: 1
Cycle: 104 | PC: 00000034 | r1: 138 | r3: 36 | r5: 96 | r7: 1
```

## Waveform Generation

The testbench generates a VCD waveform file:

```systemverilog
$dumpfile("dump.vcd");
$dumpvars(0, tb_mac53);
```

The waveform can be inspected using GTKWave:

```bash
gtkwave dump.vcd
```

## Simulation

Compile the SystemVerilog source files using Icarus Verilog:

```bash
iverilog -g2012 -o mips_simulation.vvp *.sv
```

Run the simulation:

```bash
vvp mips_simulation.vvp
```

If the source files are compiled separately, include the datapath, hazard units, forwarding unit, memory modules, and testbench:

```bash
iverilog -g2012 -o mips_simulation.vvp \
    mips_datapath.sv \
    hazard_unit_strict53.sv \
    hazard_unit_fwd53.sv \
    forwarding_unit53.sv \
    tb_mips.sv
```

## Main Modules

The project includes the following logical components:

```text
- 5-stage MIPS datapath
- Program Counter
- Instruction Memory
- Data Memory
- Register File
- ALU
- Control Unit
- Strict Hazard Unit
- Forwarding Unit
- Forwarding-Aware Hazard Unit
- Testbench
```

## Technologies

- SystemVerilog
- MIPS instruction set architecture
- Five-stage CPU pipeline
- Data hazard detection
- Operand forwarding
- Instruction and data memories
- Icarus Verilog
- EDA Playground
- GTKWave

## Key Takeaway

The project demonstrates that forwarding is an effective technique for reducing data-hazard penalties in pipelined processors.

By forwarding values directly from pipeline registers to the ALU, the processor reduces unnecessary stalls while maintaining the correct program result.

```text
157 cycles -> 103 cycles
54 cycles saved
Final result: 138 = 0x8A
```


