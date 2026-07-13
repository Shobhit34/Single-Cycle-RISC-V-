# Single-Cycle RISC-V Processor (RV32I Subset)

A single-cycle implementation of a subset of the RV32I instruction set in Verilog.

## Overview

This project implements a single-cycle RISC-V CPU datapath and control unit.
Only the Program Counter is a clocked register in the datapath (aside from
the Register File and Data Memory, which use synchronous writes / async
reads); everything else is combinational logic connecting each module.

## Supported Instructions

| Type   | Instructions        | Notes                                   |
|--------|----------------------|------------------------------------------|
| R-type | `add`, `sub`, `and`, `or` | Selected via `funct3` + `funct7[5]` |
| I-type | `addi`               | ALU forced to add                        |
| Load   | `lw`                 | Address = rs1 + imm                      |
| Store  | `sw`                 | Address = rs1 + imm                      |
| Branch | `beq`                | ALU subtracts, branches if `Zero`        |

> `slt` is also implemented in the ALU/ALU Decoder (`ALUControl = 4'b0111`)
> but is not currently wired up in the Main Decoder (no R-type `funct3=010`
> path maps to it via opcode-only decode logic — see **Limitations**).

## Architecture

```
                做+--------+
        +------>|  PC    |------+
        |       +--------+      |
        |                       v
        |                +-------------+
        |                | InsMem      |---> Instr[31:0]
        |                +-------------+
        |                       |
        |        +--------------+---------------+
        |        v              v                v
        |  +-----------+  +-----------+   +--------------+
        |  | regfile   |  | extend    |   | ControlUnit  |
        |  | (RD1,RD2) |  | (ImmExt)  |   | (all ctrl    |
        |  +-----------+  +-----------+   |  signals)    |
        |        |              |         +--------------+
        |        |     +--------+
        |        v     v
        |     +-----------+        +-----------+
        |     |  mux2     |------->|   ALU     |---> ALUResult, Zero
        |     | (ALUSrc)  |        +-----------+
        |     +-----------+              |
        |                                v
        |                         +-------------+
        |                         |   datam     |---> RD
        |                         +-------------+
        |                                |
        |                       +-----------------+
        |                       |     mux2         |
        |                       | (ResultSrc) -----+---> WD3 -> regfile
        |                                |
        |                         +-----------+
        +-------------------------| PC logic  |
        (add, mux2 for PCSrc)     +-----------+
```

## File Structure

```
├── src/
│   ├── top.v            # Top-level module — instantiates and wires everything
│   ├── add.v             # Generic 32-bit adder (used for PC+4 and branch target)
│   ├── mux2.v              # Generic 2-to-1 32-bit mux
│   ├── InsMem.v              # Instruction memory (word-addressed, 64 words)
│   ├── ALU.v                   # Arithmetic Logic Unit
│   ├── regfile.v                 # 32x32-bit register file, x0 hardwired to 0
│   ├── extend.v                    # Immediate generator (I/S/B/J formats)
│   ├── datam.v                       # Data memory (64 words)
│   ├── ControlUnit.v                   # Wraps MainDecoder + ALUDecoder + PCSrc logic
│   ├── MainDecoder.v                     # Opcode -> control signals
│   └── ALUDecoder.v                        # ALUOp + funct3/funct7 -> ALUControl
├── tb/
│   └── top_tb.v          # Testbench for the full processor
├── programs/
│   └── test.hex           # Machine code loaded into InsMem via $readmemh
└── README.md
```

## Module Interfaces (as implemented)

| Module | Key Ports |
|---|---|
| `add` | `a, b -> c` (32-bit adder) |
| `mux2` | `a, b, s -> c` (`c = s ? a : b`) |
| `InsMem` | `A -> RD` (reads `I_Mem[A[7:2]]`) |
| `ALU` | `A, B, ALUControl[3:0] -> ALUResult, Zero` |
| `regfile` | `CLK, WE3, A1, A2, A3, WD3 -> RD1, RD2` |
| `extend` | `Instr[31:7], ImmSrc[1:0] -> ImmExt` |
| `datam` | `A, clk, WE, WD -> RD` |
| `ControlUnit` | `op, funct3, funct7b5, Zero -> RegWrite, ImmSrc, ALUSrc, MemWrite, ResultSrc, PCSrc, ALUControl` |

### Control Signal Encoding

**ALUControl (from `ALUDecoder`)**

| Code | Operation |
|---|---|
| `0000` | AND |
| `0001` | OR |
| `0010` | ADD |
| `0110` | SUB |
| `0111` | SLT |
| `1100` | NOR |
| `1111` | Undefined |

**ImmSrc (from `MainDecoder`, used by `extend`)**

| Code | Format |
|---|---|
| `00` | I-type |
| `01` | S-type |
| `10` | B-type |
| `11` | J-type (extend logic present, not driven by MainDecoder yet) |

## How to Simulate

Using Icarus Verilog:

```bash
iverilog -o sim.out \
    src/add.v src/mux2.v src/InsMem.v src/ALU.v src/regfile.v \
    src/extend.v src/datam.v src/ControlUnit.v src/MainDecoder.v \
    src/ALUDecoder.v src/top.v tb/top_tb.v

vvp sim.out
```

View waveforms (if the testbench dumps a VCD):

```bash
gtkwave dump.vcd
```

## Running a Program

1. Write RISC-V assembly and assemble/link with the RISC-V GNU toolchain, or
   hand-encode instructions.
2. Convert the machine code to a hex file (one 32-bit instruction per line,
   in hex, no `0x` prefix) — this matches `$readmemh` format expected by
   `InsMem`.
3. Place the file at `programs/test.hex` and load it in `InsMem`:
   ```verilog
   initial $readmemh("programs/test.hex", I_Mem);
   ```
4. Run the simulation as shown above.

> Note: both `InsMem` and `datam` are sized `[63:0]`, i.e. 64 words (256
> bytes) each, addressed via `A[7:2]`. Programs and data must fit within
> this range.

## Testing

`tb/top_tb.v` drives `clk`/`rst`, loads a test program into instruction
memory, and checks register file / data memory values against expected
results after each instruction executes. Pass/fail results are printed to
the console via `$display`.

## Known Limitations

- No support for `jal`, `jalr`, `lui`, `auipc`, or other branch types
  (`bne`, `blt`, `bge`, etc.) — only `beq` is decoded.
- No M-extension (`mul`, `div`).
- No CSR instructions, interrupts, or exception handling.
- `slt` exists in the ALU/ALU Decoder but has no Main Decoder path wiring
  it up for a standalone use case beyond R-type `funct3 = 010`.
- No hazard detection needed (single-cycle, so this is expected/by design).
- Long combinational critical path — not optimized for clock frequency.
- `InsMem` / `datam` are small (64 words) — adjust array size if larger
  programs are needed.

## References

- RISC-V ISA Specification: https://riscv.org/technical/specifications/
- Harris & Harris, *Digital Design and Computer Architecture: RISC-V Edition*
  (datapath and control signal naming closely follows this text)
