# MU0 Processor

A fully structural SystemVerilog implementation of the MU0 processor that includes a complete datapath, control unit, memory subsystem with memory-mapped I/O, and a decimal-to-hexadecimal quiz game written in MU0 assembly.

---

## Architecture Overview

The MU0 is a minimal 16-bit processor with a 12-bit address space. It follows a two-phase fetch–execute cycle and is implemented entirely in structural Verilog using Xilinx Unisim primitive gates.

```
         ┌─────────────┐       ┌──────────────┐
  Din ──►│             │◄─────►│              │
  Dout◄──│  Datapath   │       │   Control    │
  Addr◄──│             │◄─────►│              │
         └─────────────┘       └──────────────┘
                │                      │
         ┌──────▼──────────────────────▼──────┐
         │              Memory                │
         │   RAM (0x000–0xEFF) + I/O (0xF00+) │
         └────────────────────────────────────┘
```

### Datapath Components

| Module | Description |
|---|---|
| `MU0_Reg16` | 16-bit register — used for Accumulator (ACC) and Instruction Register (IR) |
| `MU0_Reg12` | 12-bit register — used for Program Counter (PC) |
| `MU0_Mux16` | 16-bit 2-to-1 multiplexer — selects ALU X and Y inputs |
| `MU0_Mux12` | 12-bit 2-to-1 multiplexer — selects memory address (PC vs IR[11:0]) |
| `MU0_Alu`   | Combinatorial ALU supporting passthrough, add, increment, subtract |
| `MU0_Flags` | Generates N (negative) and Z (zero) flags from the accumulator |

### ALU Operations

| M[1:0] | Operation | Description |
|---|---|---|
| `00` | `Q = Y` | Passthrough Y |
| `01` | `Q = X + Y` | Addition |
| `10` | `Q = X + 1` | Increment (PC advance) |
| `11` | `Q = X − Y` | Subtraction (two's complement) |

### Control Unit

`MU0_Control` implements the fetch/execute FSM using a single flip-flop (`execute` state bit) and structural gate logic. It decodes the top 4 bits of the IR (the opcode field `F[3:0]`) and drives all datapath enable and select signals.

---

## Instruction Set

MU0 instructions are 16 bits wide: `[15:12]` hold the opcode, `[11:0]` hold the operand address S.

| Opcode | Mnemonic | Operation |
|---|---|---|
| `0000` | `LDA S` | ACC ← mem[S] |
| `0001` | `STA S` | mem[S] ← ACC |
| `0010` | `ADD S` | ACC ← ACC + mem[S] |
| `0011` | `SUB S` | ACC ← ACC − mem[S] |
| `0100` | `JMP S` | PC ← S |
| `0101` | `JGE S` | if ACC ≥ 0, PC ← S |
| `0110` | `JNE S` | if ACC ≠ 0, PC ← S |
| `0111` | `STP`   | Halt |

---

## Memory Map

The memory module (`MU0_Memory`) implements a 16×4096 synchronous RAM with address-decoded memory-mapped I/O registers in the top of the address space.

| Address | Peripheral |
|---|---|
| `0x000`–`0xEFF` | General-purpose RAM |
| `0xFF0` | Simple buttons (4 push-buttons) |
| `0xFF1` | Buttons A–H |
| `0xFF2` | 16-key keypad |
| `0xFF3` | Buzzer busy flag (read-only) |
| `0xFF4` | S7mini LEDs |
| `0xFF5`–`0xFFA` | Seven-segment digits 0–5 (BCD) |
| `0xFFD` | Buzzer control register |
| `0xFFF` | Traffic light LEDs |

The buzzer register encodes `[15]` program mode, `[11:8]` duration (in 0.1 s steps), `[7:4]` octave, and `[3:0]` note (C4–B5).

---

## Project Structure

```
mu0-processor/
├── MU0_Processor/
│   ├── MU0.sv                    # Top-level processor (structural)
│   ├── MU0_Datapath.sv           # Datapath instantiation
│   ├── MU0_Control.sv            # Control FSM (structural gates)
│   ├── MU0_Alu.sv                # ALU (combinatorial)
│   ├── MU0_Flags.sv              # Flag generator
│   ├── MU0_Reg12.sv              # 12-bit enabled register
│   ├── MU0_Reg16.sv              # 16-bit enabled register
│   ├── MU0_Mux12.sv              # 12-bit 2-to-1 mux
│   ├── MU0_Mux16.sv              # 16-bit 2-to-1 mux
│   ├── MU0_Memory.sv             # RAM + memory-mapped I/O
│   ├── MU0_Board.sv              # FPGA board top-level
│   ├── MU0_Testbench.sv          # Simulation testbench
│   ├── MU0_test.s                # Verification assembly program
│   └── MU0_test.mem              # Assembled hex memory image
└── MU0_Code/
    ├── game.s                    # Quiz game source (MU0 assembly)
    └── game.s.kmd                # Assembled game binary
```

---

## Simulation

The testbench (`MU0_Testbench.sv`) instantiates the MU0 processor alongside `MU0_Memory`, loads `MU0_test.mem`, applies a reset pulse, and runs until the `Halted` signal goes high (after approximately 34 clock cycles for the verification program).

The verification program (`MU0_test.s`) exercises:
- `STA` — store accumulator to memory
- `LDA` — load accumulator from memory
- `ADD` — addition with overflow
- `SUB` — subtraction (produces −1 = `0xFFFF`)
- `JMP` — unconditional branch
- `JNE` — conditional branch (both taken and not-taken paths)

Results are stored in memory and can be inspected in the waveform viewer after simulation. Pass values are `0x1A55`; failure values are `0xFA01`–`0xFA03`.

To simulate in Questa/ModelSim:
```
vlog MU0_Processor/*.sv
vsim MU0_Testbench
```

For Xilinx synthesis, the memory init path switches from the Questa path (`./src/Ex3/MU0_test.mem`) to the synthesis path (`MU0_test.mem`) via the `SYNTHESIS` define guard.

---

## Quiz Game — *10-16 Quiz*

`MU0_Code/game.s` is a complete interactive game written in MU0 assembly and deployed on the FPGA board.

**Objective**: Convert 15 decimal numbers to hexadecimal. The player has 4 lives and enters the two hex digits using the 16-key keypad (keys 0–F).

**Game flow**:
1. Startup — displays "HELLO", plays a welcome melody, then shows "CHANGE FROM BASE10 TO BASE16"
2. Each round — a 3-digit decimal number is shown on DIGIT_5–DIGIT_3; the player enters the two hex digits on DIGIT_1–DIGIT_0
3. Correct answer — displays "RIGHT", plays a rising C-E-G chord, advances to the next question
4. Wrong answer — displays "WRONG", plays a descending G-E-C chord, loses one life
5. Lives are shown on the traffic light LEDs (green LEDs 0–3)
6. After all questions or losing all lives — displays the score as `XX//15`, then shows "WIN" or "LOSE" with a corresponding jingle
7. Press any simple button to restart

**Question bank** (15 questions, decimal → hex):

| # | Decimal | Hex |
|---|---|---|
| 0 | 100 | 64 |
| 1 | 245 | F5 |
| 2 | 128 | 80 |
| 3 | 199 | C7 |
| 4 | 063 | 3F |
| 5 | 255 | FF |
| 6 | 170 | AA |
| 7 | 085 | 55 |
| 8 | 016 | 10 |
| 9 | 222 | DE |
| 10 | 144 | 90 |
| 11 | 189 | BD |
| 12 | 032 | 20 |
| 13 | 207 | CF |
| 14 | 136 | 88 |

---

## Target Hardware

- **FPGA board**: Xilinx S7mini (Spartan-7)
- **Clock**: 25 MHz system clock
- **Peripherals used**: 16-key keypad, 6 seven-segment digits, traffic light LEDs, piezo buzzer, simple push-buttons

---