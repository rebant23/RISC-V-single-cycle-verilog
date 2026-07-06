# HOLY CORE — Single-Cycle Edition: Supplemental Deep-Dive Notes

> **Purpose:** These notes are a companion to the `single_cycle_edition.md` tutorial. Everything here is **additional** — think of it as a "teacher sitting next to you" explaining the *why*, the *how*, and the *gotchas* behind every step. If the tutorial tells you *what* to do, these notes tell you *why it matters* and *what's really happening under the hood*.

---

## Table of Contents

1. [The Big Picture: What is a Single-Cycle CPU?](#1-the-big-picture)
2. [Why RISC-V? Why Not x86 or ARM?](#2-why-risc-v)
3. [The Clock: The Heartbeat of the CPU](#3-the-clock)
4. [Program Counter (PC): The Brain's Pointer](#4-program-counter)
5. [Instruction Memory: The Recipe Book](#5-instruction-memory)
6. [Instruction Encoding: How 32 Bits Tell a Story](#6-instruction-encoding)
7. [The Control Unit: The Brain Behind the Brain](#7-the-control-unit)
8. [The ALU: Where Math Happens](#8-the-alu)
9. [The Register File: Fast Scratch Paper](#9-the-register-file)
10. [Sign Extension: Small Numbers, Big Registers](#10-sign-extension)
11. [Data Memory: The Filing Cabinet](#11-data-memory)
12. [Muxes: The Traffic Directors](#12-muxes)
13. [R-Type Deep Dive: Register-to-Register Operations](#13-r-type)
14. [I-Type Deep Dive: Immediates and Loads](#14-i-type)
15. [S-Type Deep Dive: Stores to Memory](#15-s-type)
16. [B-Type Deep Dive: Branching and Conditions](#16-b-type)
17. [J-Type Deep Dive: Unconditional Jumps](#17-j-type)
18. [U-Type Deep Dive: Upper Immediates](#18-u-type)
19. [Subtraction and 2's Complement: The Elegant Trick](#19-subtraction)
20. [Partial Memory Operations: Bytes and Halfwords](#20-partial-memory)
21. [Verification: Why Cocotb and How It Works](#21-verification)
22. [Common Pitfalls and Lessons Learned](#22-pitfalls)
23. [What Comes After: Pipelining, Hazards, and Beyond](#23-what-comes-after)

---

## 1. The Big Picture: What is a Single-Cycle CPU? {#1-the-big-picture}

### What "Single-Cycle" Actually Means

Imagine you're a factory worker on an assembly line. In a **single-cycle** CPU, every instruction — no matter how complex — must be completed in **one tick of the clock**. That means:
- Fetch the instruction from memory
- Decode what it means
- Read any data from registers
- Do the computation
- Write the result back

All of that happens in **one clock cycle**. It's like saying "every product that comes through this factory, no matter how complicated, must be assembled in exactly 5 seconds."

### Why This is Both Beautiful and Problematic

**The beauty:** It's conceptually simple. You can reason about exactly what happens at each clock tick.

**The problem:** Your clock speed is limited by the **slowest possible instruction**. If loading from memory takes 200 picoseconds but adding two numbers takes 5 picoseconds, your entire clock cycle must be at least 200 picoseconds. You're wasting 195 picoseconds on every ADD instruction.

This is why real CPUs use **pipelining** (covered in future tutorials) — but you have to understand single-cycle first to appreciate why pipelining exists.

### The Analogies

| Component | Real-World Analogy | What It Does |
|-----------|-------------------|--------------|
| PC | Bookmark in a book | Points to current instruction |
| Instruction Memory | The book itself | Stores all instructions |
| Control Unit | Your brain reading instructions | Decodes what to do |
| ALU | Calculator | Performs math/logic |
| Register File | Sticky notes on your desk | Fast temporary storage |
| Data Memory | Filing cabinet across the room | Slow but large storage |
| Muxes | Traffic cops | Route data to the right place |

---

## 2. Why RISC-V? Why Not x86 or ARM? {#2-why-risc-v}

### The Philosophy Difference

**CISC (x86/ARM):** "I'll give you powerful, complex instructions that each do a lot. One instruction can load from memory, do math, and store back."

**RISC (RISC-V):** "I'll give you simple, uniform instructions that each do exactly one thing. You combine them like LEGO bricks."

### Why RISC-V for Learning

1. **Open source:** Anyone can implement it without paying licensing fees
2. **Clean design:** No historical baggage (unlike x86 which has 40+ years of backward compatibility cruft)
3. **Fixed instruction width:** Every instruction is exactly 32 bits. This is HUGE for simplicity — you always know where the next instruction starts
4. **Regular encoding:** The opcode is always in bits [6:0]. The destination register is always in bits [11:7]. This consistency makes decoding trivial
5. **Modular extensions:** The base integer ISA (RV32I) is minimal but complete. You can add multiply/divide (M extension), floating point (F/D), etc.

### The 6 Instruction Formats

RISC-V uses 6 instruction formats, each a different way to arrange 32 bits:

```
R-Type:  [funct7][rs2][rs1][funct3][rd][opcode]    — Register operations
I-Type:  [imm[11:0]][rs1][funct3][rd][opcode]      — Immediate/load ops
S-Type:  [imm[11:5]][rs2][rs1][funct3][imm[4:0]][opcode]  — Stores
B-Type:  [imm[12|10:5]][rs2][rs1][funct3][imm[4:1|11]][opcode]  — Branches
U-Type:  [imm[31:12]][rd][opcode]                   — Upper immediate
J-Type:  [imm[20|10:1|11|19:12]][rd][opcode]        — Jumps
```

**Key insight:** The opcode (bits [6:0]) tells you the FORMAT, and from that you know how to parse every other bit field. It's like a shipping label that says "this box contains electronics" — now you know to look for voltage specifications rather than weight limits.

---

## 3. The Clock: The Heartbeat of the CPU {#3-the-clock}

### What the Clock Signal Really Does

The clock is a square wave that oscillates between 0 and 1:

```
    ┌───┐   ┌───┐   ┌───┐   ┌───┐
    │   │   │   │   │   │   │   │
────┘   └───┘   └───┘   └───┘   └───
    ↑       ↑       ↑       ↑
  rising  rising  rising  rising
  edge    edge    edge    edge
```

In this design, we use **posedge clk** — meaning things happen on the **rising edge** (when the signal goes from 0 to 1).

### Why Rising Edge Matters

All the state elements (PC, registers, memory) update their values on the rising edge. Between edges, the combinational logic (ALU, control, muxes) settles to new values. This is like:

1. **Clock rises:** "Snap! Take a photo of everything as it is now."
2. **Between clocks:** "Everyone compute your new values. But don't write them down yet."
3. **Clock rises again:** "Snap! Now update everything with the computed values."

This is the fundamental principle of **synchronous digital design**.

### Clock Speed = Performance Ceiling

In a single-cycle CPU, the clock period must be long enough for the **slowest possible instruction** to complete. This means:

```
T_clock ≥ T_fetch + T_decode + T_execute + T_memory + T_writeback
```

If one path through the CPU takes 2ns, your clock can't be faster than 500 MHz. Every instruction, even the simple ones, "wastes" the extra time.

### Why We Use Active-Low Reset (`rst_n`)

Notice `rst_n` in the code — the `n` means "not" (active-low). So when `rst_n = 0`, the system is in reset. When `rst_n = 1`, it's running normally.

Why active-low? Historical convention from TTL logic days where it was easier/more reliable to pull a line low to trigger something. It stuck.

---

## 4. Program Counter (PC): The Brain's Pointer {#4-program-counter}

### The PC is Just a Register

The PC is a 32-bit register that holds the memory address of the current instruction. That's it. It's one of the simplest components, yet absolutely critical.

### Why PC+4?

Since every instruction is exactly 32 bits (4 bytes), and memory is byte-addressed, the next instruction is always at `PC + 4`. It's like reading a book where every page has exactly one paragraph — to get to the next paragraph, you always turn exactly one page.

```
Address: 0x00  0x04  0x08  0x0C  0x10
         │     │     │     │     │
         ▼     ▼     ▼     ▼     ▼
        [I1]  [I2]  [I3]  [I4]  [I5]
         ↑
         PC points here
```

### The PC Next Logic

The PC needs to know WHERE to go next. It has two choices:
1. **PC + 4** (default: just go to the next instruction)
2. **PC + immediate** (branch/jump: skip ahead or loop back)

A mux selects between these based on `pc_source`.

### Why Reset to 0?

When the CPU starts (or resets), the PC is set to 0. This means instruction memory must have the first instruction at address 0. This is a design choice — you could start anywhere, but 0 is conventional.

### Gotcha: The PC Updates AFTER Everything Else

The PC only changes on `posedge clk`. This means during one clock cycle, the PC holds a constant value while all the combinational logic works with that fixed address. Only at the NEXT rising edge does the PC update to `pc_next`. This is what makes everything deterministic.

---

## 5. Instruction Memory: The Recipe Book {#5-instruction-memory}

### ROM vs RAM

The instruction memory is implemented as a **ROM** (Read-Only Memory) — we never write to it during execution. It's initialized from a `.hex` file at startup using `$readmemh`.

### Why Separate Instruction and Data Memory?

This is the **Harvard Architecture** (separate instruction and data memories) vs **Von Neumann** (shared memory). Harvard is simpler for single-cycle because:
- We need to read an instruction AND potentially read/write data in the SAME cycle
- With shared memory, we'd need two ports (dual-port memory), which is more complex
- With separate memories, we can do both operations simultaneously

### The Address Mapping

Memory is word-addressed internally but the CPU provides byte addresses. So:
- CPU sends address `0x00` → memory reads word at index `0` (bits [31:2] of address)
- CPU sends address `0x04` → memory reads word at index `1`
- CPU sends address `0x08` → memory reads word at index `2`

The **bottom 2 bits are discarded** because instructions are always word-aligned (4-byte aligned). This is why `address[31:2]` is used as the memory index.

### The `.hex` File Format

The hex file is just a list of 32-bit hex values, one per line:
```
00802903
01202623
01002983
```

Each line is one instruction. The `$readmemh` SystemVerilog task reads this into the memory array at initialization time.

### Why `write_enable = 0` for Instruction Memory?

We connect `write_enable = 0` to ensure we never accidentally write to instruction memory. It's belt-and-suspenders safety.

---

## 6. Instruction Encoding: How 32 Bits Tell a Story {#6-instruction-encoding}

### The Anatomy of Every Instruction

Every 32-bit instruction has the **opcode** in bits [6:0] (the lowest 7 bits). This is always the first thing the control unit looks at.

### The Opcode Tells You the Format

```
0110011  → R-Type (register operations: add, sub, and, or, xor, slt...)
0000011  → I-Type Load (lw, lb, lh, lbu, lhu)
0010011  → I-Type ALU (addi, xori, ori, andi, slti, sltiu, slli, srli, srai)
0100011  → S-Type (store: sw, sb, sh)
1100011  → B-Type (branch: beq, bne, blt, bge, bltu, bgeu)
0110111  → U-Type LUI
0010111  → U-Type AUIPC
1101111  → J-Type JAL
1100111  → J-Type JALR
```

### The Funct3 and Funct7: Sub-_OPCODEs

Once you know the format, you need `funct3` (bits [14:12]) and sometimes `funct7` (bits [31:25]) to distinguish between operations of the same type.

**Why can't the opcode do it all?** Because there are only 128 possible opcodes (7 bits), but RISC-V has hundreds of instructions. Funct3 and Funct7 give you the extra encoding space.

**Key insight for R-Types:** `add` and `sub` have the SAME opcode AND SAME funct3 (both are 000). The ONLY difference is funct7: `add` has `0000000` and `sub` has `0100000`. This is why the control unit needs to check funct7 for R-types.

### The Destination Register: Always [11:7]

Every instruction that writes a result puts the destination register number in bits [11:7]. This allows the register file to start writing as soon as the computation is done, without waiting for the control unit.

### The Source Registers: [19:15] and [24:20]

For instructions that read registers, `rs1` is always at [19:15] and `rs2` is always at [24:20]. This parallelism (both register reads happen simultaneously) is crucial for single-cycle performance.

---

## 7. The Control Unit: The Brain Behind the Brain {#7-the-control-unit}

### What the Control Unit Really Does

The control unit is a giant **lookup table** (implemented as combinational logic). It takes the opcode (and sometimes funct3/funct7) as input and produces a set of control signals as output.

Think of it like a decoder ring: you see a coded message (the instruction), and the ring tells you what each part means and what to do.

### The Two-Level Decoder Structure

The tutorial uses a **two-level decoding** scheme:

1. **Main Decoder:** Looks at the opcode and sets major signals (reg_write, mem_write, alu_source, imm_source, etc.)
2. **ALU Decoder:** Looks at the ALU opcode (a 2-bit signal from the main decoder) AND funct3 to determine the ALU operation.

**Why two levels?** Because many instruction types need the SAME ALU operations. For example, both R-type `add` and I-type `addi` need the ALU to do addition. The main decoder classifies the instruction type, and the ALU decoder picks the specific operation.

### The Control Signals Explained

| Signal | Width | What It Does | Why It Exists |
|--------|-------|--------------|---------------|
| `reg_write` | 1 | Enable writing to register file | Don't clobber registers on store/branch instructions |
| `mem_write` | 1 | Enable writing to data memory | Only `sw`, `sb`, `sh` should write to memory |
| `alu_source` | 1 | Choose ALU's second input (rs2 vs immediate) | R-Types use rs2; I/S/B-Types use immediate |
| `write_back_source` | 2 | Choose what gets written to register | ALU result, memory data, PC+4, or PC+imm |
| `imm_source` | 3 | Choose how to extract the immediate | Different formats have immediates in different bit positions |
| `alu_control` | 4 | Tell ALU what operation to do | Add, sub, and, or, xor, slt, shift... |
| `branch` | 1 | Signal that this is a branch instruction | Combined with ALU flags to determine if branch is taken |
| `jump` | 1 | Signal that this is a jump instruction | Always changes PC (unconditional) |
| `pc_source` | 1 | Choose next PC (PC+4 vs target) | Branches and jumps redirect execution |
| `second_add_source` | 2 | Choose what the second adder adds to PC | Different jump/branch types compute target differently |

### Why `alu_op` Exists (The Indirection Layer)

Instead of the main decoder directly setting `alu_control`, it sets a 2-bit `alu_op`:
- `00` = Load/Store (ALU always adds for address calculation)
- `01` = Branch (ALU does subtraction for comparison)
- `10` = R-Type/I-Type ALU (look at funct3 for specific operation)

This indirection **simplifies the main decoder**. The main decoder doesn't need to know about every ALU operation — it just says "this is a math instruction" and lets the ALU decoder figure out the details.

### The `pc_source` Logic

```
pc_source = (assert_branch & is_branch_instruction) | jump
```

This elegant formula handles:
- `beq` when registers are equal → `assert_branch = 1`, `branch = 1` → `pc_source = 1`
- `jal` → `jump = 1` → `pc_source = 1` (always)
- Any non-branch, non-jump instruction → `pc_source = 0` (default to PC+4)

---

## 8. The ALU: Where Math Happens {#8-the-alu}

### The ALU is a Swiss Army Knife

The ALU (Arithmetic Logic Unit) is a combinational circuit that performs one of several operations based on the `alu_control` signal. It takes two 32-bit inputs and produces a 32-bit result plus flags.

### The Operations

| ALU Control | Operation | What It Computes | Used By |
|-------------|-----------|------------------|---------|
| `0000` | ADD | src1 + src2 | lw, sw, add, addi, auipc |
| `0001` | SUB | src1 - src2 | beq, sub |
| `0010` | AND | src1 & src2 | and, andi |
| `0011` | OR | src1 \| src2 | or, ori |
| `0100` | SLL | src1 << src2[4:0] | sll, slli |
| `0101` | SLT | (src1 < src2) ? 1 : 0 | slt, slti, blt |
| `0110` | SRL | src1 >> src2[4:0] | srl, srli |
| `0111` | SLTU | (src1 < src2) ? 1 : 0 (unsigned) | sltu, sltiu, bltu |
| `1000` | XOR | src1 ^ src2 | xor, xori |
| `1001` | SRA | src1 >>> src2[4:0] | sra, srai |

### The Flags: `zero` and `last_bit`

- **`zero`:** 1 if `alu_result == 0`. Used by `beq` — if subtracting two numbers gives 0, they're equal!
- **`last_bit`:** 1 if `alu_result[0] == 1`. Used by `blt` — the SLT operation puts the comparison result in bit 0.

**Why separate flags instead of having the ALU decide branching?** Because the ALU shouldn't know about branching. It just does math. The control unit interprets the flags. This separation of concerns makes the design cleaner and more modular.

### The SLT Trick

"Set Less Than" is one of the most elegant tricks in RISC-V:

```verilog
3'b101 : alu_result = {31'b0, $signed(src1) < $signed(src2)};
```

This computes `src1 < src2` as a signed comparison and puts the result (0 or 1) in bit 0, with all other bits being 0. Then:
- For `slt`/`slti`: the result IS the comparison (0 or 1)
- For `blt`: the control unit reads `last_bit` (which is bit 0) to decide branching

**Why `$signed`?** Without it, `src1 < src2` would be an unsigned comparison in SystemVerilog. We need signed comparison for `slt` and `slti` because they're defined as signed operations in the RISC-V spec.

### Subtraction via 2's Complement

The ALU doesn't have a dedicated subtractor. Instead:

```verilog
3'b001 : alu_result = src1 + (~src2 + 1'b1);
```

`~src2 + 1` is the 2's complement of `src2`, which is its negative. So `src1 + (-src2)` = `src1 - src2`. This is mathematically elegant and saves hardware — we reuse the adder.

---

## 9. The Register File: Fast Scratch Paper {#9-the-register-file}

### Why 32 Registers?

RISC-V has 32 general-purpose registers (x0-x31). This is a sweet spot:
- **Too few** (like early 8-bit CPUs with 2-4 registers): you'd constantly need to save/restore values to slow memory
- **Too many** (like some VLIW architectures with 128+ registers): each register needs wires, and the chip area grows quadratically

32 registers means 5 bits to address each one (2⁵ = 32), which fits nicely in the instruction encoding.

### Register x0 is Special

Register x0 is **hardwired to zero**. Writing to x0 is ignored. Reading x0 always returns 0.

This is incredibly useful:
- `add x0, x1, x2` → does nothing (discards result)
- `lw x0, 0(x1)` → loads data but throws it away (useful for prefetching)
- `addi x0, x0, 0` → this is the NOP (No Operation) instruction!

### The Read-Write Ports

The register file has:
- **2 read ports** (address1/read_data1, address2/read_data2): can read two registers simultaneously
- **1 write port** (address3/write_data/write_enable): can write one register

Why 2 read ports? Because most instructions read TWO source registers (rs1 and rs2). With 2 ports, both reads happen in parallel.

Why only 1 write port? Because every instruction writes at most ONE destination register. Having more would waste resources.

### Read-During-Write Behavior

What happens if you try to read and write the same register in the same cycle? The register file is designed to output the **old value** (the value before the write). This is called "read-first" behavior and is important for correctness.

### The Write-Back Path

The register file receives data from a mux called `write_back_source`. This mux selects between:
- `00`: ALU result (for R-types and I-type ALU)
- `01`: Memory read data (for loads)
- `10`: PC + 4 (for JAL/JALR return addresses)
- `11`: PC + second_add (for U-types like AUIPC)

---

## 10. Sign Extension: Small Numbers, Big Registers {#10-sign-extension}

### The Problem

Instructions encode immediates (constant values) in limited bit fields. For example, I-type instructions only have 12 bits for the immediate. But the ALU needs 32-bit inputs. How do you turn 12 bits into 32 bits?

### The Solution: Sign Extension

If the number is positive (bit 11 = 0), pad with zeros:
```
000000001010 → 00000000000000000000000000001010
```

If the number is negative (bit 11 = 1), pad with ones:
```
111111111010 → 11111111111111111111111111111010
```

This preserves the sign and magnitude of the original number.

### Why the Immediate Bit Positions Are Weird

Each instruction format scatters the immediate bits across different positions in the instruction word. The sign extender must gather them and reassemble them.

**Why not just put the immediate in a contiguous block?** Because RISC-V was designed so that:
- The opcode is always at [6:0]
- The destination register is always at [11:7]
- The source registers are always at [19:15] and [24:20]

These fields need to be in FIXED positions so the hardware can decode them in parallel. The immediate bits get "squeezed" into whatever space is left.

### B-Type Immediate: The Most Bizarre

The B-type immediate is:
```
{imm[12], imm[10:5], rs2, rs1, funct3, imm[4:1], imm[11], opcode}
```

The bits are scattered because:
1. Bit 0 is always 0 (instructions are 2-byte aligned, so bit 0 is never needed)
2. Bit 11 ends up near the opcode due to field alignment constraints
3. The RISC-V designers prioritized hardware simplicity over human readability

### The Trailing Zero Trick

Notice that both B-type and J-type immediates have a trailing 0 appended (bit 0 is always 0). This effectively doubles the range of the immediate:
- Without trailing 0: ±2048 byte range (12-bit signed)
- With trailing 0: ±4096 byte range (13-bit signed offset)

Since instructions are always 2-byte aligned (or 4-byte in our case), the LSB is always 0, so we can "steal" it for range.

---

## 11. Data Memory: The Filing Cabinet {#11-data-memory}

### RAM vs ROM

Unlike instruction memory (ROM), data memory is **RAM** — we both read and write to it. It's initialized from a hex file but can be modified during execution.

### The Memory Interface

```
address[31:0]     → Where to read/write
write_data[31:0]  → Data to write
write_enable      → 1 = write, 0 = don't write
byte_enable[3:0]  → Which bytes to write (for sb/sh)
read_data[31:0]   → Data read from memory
```

### Word Alignment

The memory is organized as an array of 32-bit words. The CPU provides a byte address, but the memory uses `address[31:2]` as the index (discarding the bottom 2 bits). This means:
- Address 0x00 → word 0
- Address 0x04 → word 1
- Address 0x08 → word 2
- Address 0x01 → word 0 (same as 0x00! Bottom bits ignored)

This is why misaligned accesses (like loading a word from address 0x01) are problematic — they'd hit the wrong word.

### The 1-Cycle Memory Access Assumption

In this single-cycle design, we assume memory access takes exactly 1 clock cycle. In reality, off-chip memory (DRAM) takes hundreds of cycles. This is one of the biggest simplifications of the single-cycle design.

---

## 12. Muxes: The Traffic Directors {#12-muxes}

### What is a Mux?

A multiplexer (mux) is a combinational circuit that selects one of several inputs and forwards it to a single output. Think of it as a railroad switch:

```
Input A ─────┐
             │
Input B ─────┼───── Output
             │
Select ──────┘
```

If Select = 0, Output = A. If Select = 1, Output = B.

### The Critical Muxes in the CPU

1. **PC Source Mux:** Selects between PC+4 (sequential) and PC+immediate (branch/jump)
2. **ALU Source Mux:** Selects between rs2 (register) and immediate (constant)
3. **Write-Back Source Mux:** Selects what gets written to the register file (ALU result, memory data, PC+4, or PC+imm)
4. **Second Add Source Mux:** Selects what gets added to PC for jump targets (PC, 0, or rs1+immediate)

### Why Muxes Are Everywhere

Every time you have a choice in the datapath, you need a mux. The control unit's job is essentially to set all the mux select signals correctly for each instruction.

### Mux Width Matters

A 1-bit mux selects between two 32-bit values. A 2-bit mux selects between four 32-bit values. A 3-bit mux selects between eight 32-bit values. The select signal width determines how many options you have.

---

## 13. R-Type Deep Dive: Register-to-Register Operations {#13-r-type}

### The R-Type Format

```
[funct7 (7 bits)][rs2 (5)][rs1 (5)][funct3 (3)][rd (5)][opcode (7)]
```

### Why R-Types Don't Use Immediates

R-Types operate on two registers and write to a third. They don't need constants. This is why `alu_source = 0` (select rs2, not immediate).

### The Funct7 Distinction

R-Types are the only format that uses all three fields (opcode, funct3, AND funct7) to determine the operation. This is because:
- `add` and `sub` have the same opcode and funct3 — only funct7 differs
- `srl` and `sra` have the same opcode and funct3 — only funct7 differs

**Why this design?** It keeps the opcode/funct3 encoding space manageable. If every operation needed a unique opcode, we'd run out of encodings.

### The Pattern: Control → ALU → Write Back

For R-types, the data flow is:
1. Read rs1 and rs2 from register file
2. ALU computes result
3. Result is written back to rd (because `write_back_source = 00` selects ALU result)

This is the simplest data path in the CPU.

---

## 14. I-Type Deep Dive: Immediates and Loads {#14-i-type}

### Two Kinds of I-Types

The tutorial distinguishes between:
- **I-Type Load:** `lw`, `lb`, `lh`, `lbu`, `lhu` (opcode `0000011`)
- **I-Type ALU:** `addi`, `xori`, `ori`, `andi`, `slti`, `sltiu`, `slli`, `srli`, `srai` (opcode `0010011`)

They have the SAME format but different opcodes and different behavior:
- Load I-Types: ALU adds rs1 + immediate to compute address, then memory is read
- ALU I-Types: ALU performs operation on rs1 and immediate, result goes to rd

### Why the Same Format?

Because both use one register (rs1) and one immediate. The hardware is identical — only the control signals differ. This is the beauty of RISC-V's regular encoding.

### The 12-bit Immediate Limitation

I-Type immediates are 12 bits signed, giving a range of -2048 to +2047. For `addi`, this means you can only add constants in this range. To load larger constants, you need `lui` + `addi` (covered in U-Types).

### Shift Instruction Special Case

For `slli`, `srli`, `srai`, the "immediate" field is actually a 5-bit shift amount (bits [24:20] of the instruction, which overlaps with what would be rs2 in R-types). The upper 7 bits act like funct7 to distinguish `srl` from `sra`.

---

## 15. S-Type Deep Dive: Stores to Memory {#15-s-type}

### The S-Type Format

```
[imm[11:5] (7 bits)][rs2 (5)][rs1 (5)][funct3 (3)][imm[4:0] (5)][opcode (7)]
```

The immediate is SPLIT across two locations — the upper 7 bits at [31:25] and the lower 5 bits at [11:7]. The sign extender must reassemble them.

### Why Split the Immediate?

To keep rs1 and rs2 in the SAME positions as R-types. This means the register file reads don't need to change based on instruction type — rs1 is always at [19:15] and rs2 is always at [24:20]. This parallelism is critical for single-cycle performance.

### The Data Flow

For `sw`:
1. Read rs1 (base address) and rs2 (data to store)
2. ALU computes address: rs1 + sign-extended immediate
3. Data from rs2 is written to data memory at the computed address

**Key insight:** The ALU does the same addition as `lw` — the only difference is that `sw` writes TO memory while `lw` reads FROM memory.

---

## 16. B-Type Deep Dive: Branching and Conditions {#16-b-type}

### What Branching Really Means

A branch is an **conditional** change of the program counter. "IF this condition is true, jump to address X; otherwise, continue normally."

### The Data Flow for `beq`

1. Read rs1 and rs2 from register file
2. ALU subtracts: rs1 - rs2 (using SUB operation)
3. If result is 0 (meaning rs1 == rs2), `alu_zero = 1`
4. Control unit sees `branch = 1` AND `alu_zero = 1` → `pc_source = 1`
5. PC gets loaded with PC + immediate (the branch target)

### Why Subtraction for Equality?

If `rs1 - rs2 = 0`, then `rs1 = rs2`. The subtraction gives us an equality check "for free" — we're already doing the math, and the `zero` flag tells us the answer.

### The B-Type Immediate Range

The B-type immediate is 13 bits (12 bits + a trailing 0), giving a range of ±4096 bytes (±2048 instructions). This is relative to the current PC, so you can branch forward or backward.

### Forward vs Backward Branches

- **Forward branch:** Positive immediate → skip ahead (skip loop iterations, skip error handling code)
- **Backward branch:** Negative immediate → go back (loop back to start)

The two's complement representation handles both cases transparently.

### The Branch Logic in Control

```verilog
case (func3)
    F3_BEQ : assert_branch = alu_zero & branch;
    F3_BLT : assert_branch = alu_last_bit & branch;
    default : assert_branch = 1'b0;
endcase
```

Each branch type uses a different ALU flag:
- `beq`: uses `zero` (result of subtraction)
- `blt`: uses `last_bit` (result of SLT comparison)

The control unit combines the flag with the `branch` signal to decide whether to redirect the PC.

---

## 17. J-Type Deep Dive: Unconditional Jumps {#17-j-type}

### JAL: Jump And Link

`jal` does two things:
1. **Jump:** Set PC to PC + immediate (always, no condition)
2. **Link:** Store PC + 4 into rd (the return address)

### Why Store PC+4?

When you call a function, you need to know where to return. By storing PC+4 (the address of the next instruction) into a register (usually x1, called `ra` for return address), the function can later jump back using `jalr`.

### The J-Type Immediate

The J-type immediate is 21 bits (20 bits + trailing 0), giving a range of ±1 MB. This is the largest range of any instruction type, because jumps often need to reach far-away code.

### Why J-Type Has Its Own Format

JAL is the only J-type instruction in RV32I. It needs its own format because:
- It writes to a register (rd) → needs the rd field
- It doesn't read any registers → no rs1 or rs2 fields
- It has a very large immediate → needs 20+ bits for the offset

---

## 18. U-Type Deep Dive: Upper Immediates {#18-u-type}

### The Problem: Loading 32-bit Constants

I-type immediates are only 12 bits. But sometimes you need to load a full 32-bit constant into a register. How?

### The Solution: Two-Step Loading

1. `lui x5, 0x12345` → loads the upper 20 bits: x5 = 0x12345000
2. `addi x5, x5, 0x678` → adds the lower 12 bits: x5 = 0x12345678

This two-instruction sequence can load ANY 32-bit constant.

### LUI vs AUIPC

- **LUI** (Load Upper Immediate): Loads the upper 20 bits directly into rd
- **AUIPC** (Add Upper Immediate to PC): Loads the upper 20 bits and adds PC, storing the result in rd

AUIPC is useful for PC-relative addressing — loading the address of data that's far away but at a known offset from the current code.

### The U-Type Immediate

The U-type immediate is the simplest: bits [31:12] of the instruction are directly the upper 20 bits of the immediate, with the lower 12 bits being zero. No bit gymnastics needed!

### Why U-Type is Different

U-types don't use the ALU for computation (in the traditional sense). Instead:
- LUI: the immediate goes directly to the write-back path
- AUIPC: the PC + immediate goes to the write-back path

This is why they need a new `write_back_source` value (2'b11) and a `second_add_source` signal.

---

## 19. Subtraction and 2's Complement: The Elegant Trick {#19-subtraction}

### 2's Complement Review

To negate a binary number in 2's complement:
1. Flip all bits (NOT operation)
2. Add 1

So `-5` in 8-bit is:
```
5 = 00000101
~5 = 11111010
~5+1 = 11111011 = -5
```

### Why the ALU Uses Addition for Subtraction

The ALU has one adder circuit. To subtract, it computes:
```
result = src1 + (~src2 + 1)
```

This is `src1 + (-src2)` = `src1 - src2`. We save hardware by reusing the adder instead of building a separate subtractor.

### The Order Matters: src1 - src2, NOT src2 - src1

In RISC-V, `sub rd, rs1, rs2` computes `rs1 - rs2`. The ALU must be wired so that:
- `src1` = value from rs1
- `src2` = value from rs2 (or immediate)

Getting this backwards would give `rs2 - rs1`, which is wrong.

### Why We Don't Care About Signs at the ALU Level

The ALU just does binary arithmetic. It doesn't know or care whether the numbers are signed or unsigned. The interpretation (signed vs unsigned) happens at the instruction level:
- `sub` → signed subtraction
- `bltu` → unsigned comparison
- `slti` → signed comparison
- `sltiu` → unsigned comparison

The same hardware supports both — only the interpretation changes.

---

## 20. Partial Memory Operations: Bytes and Halfwords {#20-partial-memory}

### Why Not Just Use Words?

Sometimes you need to work with smaller data:
- A character (ASCII) is 1 byte (8 bits)
- A short integer is 2 bytes (16 bits)
- A pixel in some color formats is 1-2 bytes

Using a full 32-bit word for a single byte wastes 75% of memory bandwidth.

### The Byte Enable Mask

When writing a byte, the memory needs to know WHICH byte(s) to modify:

```
byte_enable = 4'b0001 → write only byte 0 (bits [7:0])
byte_enable = 4'b0010 → write only byte 1 (bits [15:8])
byte_enable = 4'b0100 → write only byte 2 (bits [23:16])
byte_enable = 4'b1000 → write only byte 3 (bits [31:24])
byte_enable = 4'b1111 → write all bytes (full word)
```

### The Load Store Decoder

This new module sits between the ALU and memory. It:
1. Takes the ALU result (address) and the instruction's funct3
2. Determines which bytes are being accessed
3. Generates the appropriate byte_enable mask
4. Shifts the data to the correct byte lanes

### Why Data Must Be Shifted

When storing a byte to address 0x02 (byte 2 of word 0), the byte must be placed in bits [23:16], not bits [7:0]. The load_store_decoder handles this shifting.

### Misaligned Accesses

What happens if you try to store a halfword at an odd address? The RISC-V spec says this is technically misaligned. The tutorial's approach: **do nothing** (set byte_enable to 0). In a real CPU, this might trigger an exception.

### Sign Extension for Loads

When loading a byte, should `0xDE` become `0x000000DE` (unsigned) or `0xFFFFFFDE` (signed)? That depends on the instruction:
- `lb` (Load Byte): sign-extends → `0xFFFFFFDE`
- `lbu` (Load Byte Unsigned): zero-extends → `0x000000DE`

The `reader` module handles this using `f3[2]` as the sign-extension control bit.

### The Valid Flag

The `reader` module produces a `valid` flag that's 0 when the byte_enable mask is all zeros (misaligned access). This prevents writing garbage data to the register file. The valid flag is ANDed with `reg_write` to create the actual write enable.

---

## 21. Verification: Why Cocotb and How It Works {#21-verification}

### What is Cocotb?

Cocotb is a Python-based verification framework for HDL designs. Instead of writing Verilog testbenches (which are verbose and hard to debug), you write Python tests that interact with your design through a simulation interface.

### How It Works

1. **Compile:** Your Verilog/SystemVerilog is compiled with Icarus Verilog or Verilator
2. **Simulate:** The compiled design is loaded into a simulator
3. **Control:** Cocotb sends signals to the design (like setting input values) and reads signals (like checking output values)
4. **Assert:** Python assertions verify that the design behaves correctly

### Why Randomized Testing?

The tutorial uses `random.randint()` extensively. This is called **constrained random verification** and it's powerful because:
- It exercises corner cases you might not think of
- It tests thousands of input combinations automatically
- It catches bugs that hand-crafted tests miss

### The Test Hierarchy

The tutorial uses a bottom-up approach:
1. **Unit tests:** Test each module independently (ALU, sign extender, register file, memory)
2. **Integration tests:** Test modules connected together (control + datapath)
3. **System tests:** Run a complete program on the CPU and verify register values

This catches bugs at the lowest possible level, making debugging easier.

### Why `RisingEdge(dut.clk)`?

In cocotb, `await RisingEdge(dut.clk)` waits for the next clock rising edge. This is how you synchronize with the CPU's operation — you set up inputs, wait for a clock edge, then check outputs.

### The `Timer(1, units="ns")` Trick

A small delay allows combinational logic to settle before checking outputs. Without it, you might check a signal before it has propagated through all the gates.

---

## 22. Common Pitfalls and Lessons Learned {#22-pitfalls}

### Pitfall 1: Forgetting to Update Signal Widths

When you expand `alu_control` from 3 to 4 bits, you must update:
- The ALU module declaration
- The control unit output declaration
- The CPU datapath wire declaration
- ALL testbench assertions that check `alu_control`

**Lesson:** Changing one signal's width ripples through the entire design.

### Pitfall 2: Mixing `always_comb` Blocks

The tutorial warns about putting `second_add_select` and `pc_select` in the same `always_comb` block. This can cause simulation issues because:
- `always_comb` blocks are sensitive to their inputs
- If two logical decisions are in one block, the simulator might not propagate changes correctly
- **Rule of thumb:** Keep independent mux selections in separate `always_comb` blocks

### Pitfall 3: Signed vs Unsigned Comparisons

Using `src1 < src2` without `$signed()` in SystemVerilog performs an UNSIGNED comparison. This is a very common bug when implementing `slt` (which needs signed comparison).

### Pitfall 4: Forgetting the Trailing Zero

When constructing B-type and J-type immediates, forgetting to append the trailing 0 bit means your branch/jump targets are wrong by a factor of 2.

### Pitfall 5: Not Testing After Every Change

The tutorial emphasizes re-running ALL tests after every modification. This catches regression bugs early, when they're easy to fix.

### Pitfall 6: Misaligned Memory Accesses

If you forget to discard the bottom 2 bits of the address when accessing data memory, you'll read/write to the wrong word. Always use `address[31:2]` as the memory index.

### Pitfall 7: Hardcoded Values vs Constants

The tutorial ends by introducing a package with named constants. Hardcoded binary values like `7'b0110011` are error-prone and hard to read. Using `OPCODE_R_TYPE` is much clearer.

### Pitfall 8: Not Handling Invalid Instructions

The tutorial doesn't implement illegal instruction detection. In a real CPU, fetching an invalid opcode should trigger an exception. For now, invalid instructions simply produce undefined behavior.

---

## 23. What Comes After: Pipelining, Hazards, and Beyond {#23-what-comes-after}

### The Fundamental Problem with Single-Cycle

Every instruction takes the same amount of time — the time of the SLOWEST instruction. This wastes enormous amounts of time on simple instructions.

### Pipelining: The Factory Assembly Line

Instead of one worker doing everything, pipeline stages specialize:
1. **IF (Instruction Fetch):** Get instruction from memory
2. **ID (Instruction Decode):** Decode instruction, read registers
3. **EX (Execute):** ALU computation
4. **MEM (Memory Access):** Read/write data memory
5. **WB (Write Back):** Write result to register file

Now each stage takes roughly the same time, and the clock can be much faster.

### The Three Hazards

1. **Data Hazards:** An instruction depends on the result of a previous instruction that hasn't finished yet
2. **Control Hazards:** A branch/jump changes the PC, but the next instruction was already fetched
3. **Structural Hazards:** Two instructions need the same hardware resource at the same time

### Forwarding: The First Fix

Instead of waiting for a result to be written back to the register file, forward it directly from the EX or MEM stage to where it's needed. This eliminates most data hazards without stalling.

### Branch Prediction: Guessing the Future

Instead of waiting to know if a branch is taken, the CPU GUESSES. If it guesses right, no time is lost. If wrong, it flushes the wrong instructions and starts over. Modern CPUs achieve 95%+ prediction accuracy.

### The FPGA Edition

The tutorial mentions implementing the core on an FPGA. This involves:
- Building an SoC (System on Chip) around the core
- Adding bus interfaces (AXI) to connect peripherals
- Implementing cache to handle slow external memory
- Running real programs with I/O (LEDs, UART, etc.)

---

## Quick Reference: Instruction Types Summary

| Type | Opcode | Immediate | Uses ALU? | Writes Reg? | Writes Mem? | Example |
|------|--------|-----------|-----------|-------------|-------------|---------|
| R | 0110011 | None | Yes | Yes | No | `add x1, x2, x3` |
| I (Load) | 0000011 | 12-bit signed | Yes (address) | Yes | No | `lw x1, 0(x2)` |
| I (ALU) | 0010011 | 12-bit signed | Yes | Yes | No | `addi x1, x2, 5` |
| S | 0100011 | 12-bit signed | Yes (address) | No | Yes | `sw x1, 0(x2)` |
| B | 1100011 | 13-bit signed | Yes (compare) | No | No | `beq x1, x2, label` |
| J | 1101111 | 21-bit signed | No | Yes | No | `jal x1, label` |
| U (LUI) | 0110111 | 20-bit upper | No | Yes | No | `lui x1, 0x12345` |
| U (AUIPC) | 0010111 | 20-bit upper | No | Yes | No | `auipc x1, 0x12345` |

---

## Final Thoughts

Building a single-cycle RISC-V CPU is one of the most educational exercises in computer engineering. You touch every layer of abstraction:
- **Architecture:** Instruction formats, register conventions, memory model
- **Logic Design:** Combinational vs sequential logic, muxes, decoders
- **HDL Coding:** SystemVerilog syntax, module hierarchy, parameterization
- **Verification:** Test-driven development, randomized testing, assertion-based verification

The single-cycle design is intentionally simple, but it contains the seeds of every concept needed for modern CPU design. Understanding this thoroughly makes pipelining, caching, out-of-order execution, and other advanced topics much more accessible.

**The journey from "I don't know what a CPU is" to "I built one" is one of the most rewarding paths in engineering. Welcome to the other side.** 🎉

---

*These notes were created as a supplement to the HOLY_CORE single-cycle edition tutorial. For the full HDL code and implementation details, refer to the original tutorial.*
