# RV32I

Implement a simplified RV32I emulator. Define your own instruction enum, decode each fetched word into it, and execute the decoded instruction using pattern matching. Submit `main.rs` and any additional source files you need. Use Rust 1.88 and the standard library.

## Input

The first token is a decimal program count `t` (`1 <= t <= 100`). For each program, read a decimal word count `n` (`0 <= n <= 16384`), followed by `n` instruction words. Each word has exactly eight hexadecimal digits, without `0x`; either letter case is accepted. Tokens are separated by whitespace. The input format is valid, but instruction words may contain unsupported encodings.

## Machine

Each program starts with 32 zeroed 32-bit registers, `pc = 0`, and 65536 zeroed bytes of memory. Register `x0` always reads as zero; writes to it are discarded. Place the input words in memory starting at address zero, in little-endian order. Stores may change these bytes, and later instruction fetches must read the changed memory.

Use the [RV32I instruction semantics and encodings](https://docs.riscv.org/reference/isa/v20260120/unpriv/rv32.html), with these rules:

- Arithmetic and address calculations wrap modulo 2^32. Signed operations interpret a register as a two's-complement `i32`.
- Instructions occupy four bytes. Normally advance `pc` by four. Branch and `JAL` offsets are relative to the current instruction's address. `JAL` and `JALR` save `pc + 4`; `JALR` clears bit zero of its target. Read source operands before writing the destination, including when `JALR` uses the same source and destination register.
- Register shifts use only the lowest five bits of the shift count. Ordinary 12-bit immediates are sign-extended, including the immediate of `SLTIU`.
- Byte, halfword and word memory accesses require alignment to 1, 2 and 4 bytes, respectively. Every accessed byte must be inside memory. `LB` and `LH` sign-extend; `LBU` and `LHU` zero-extend. Loads into `x0` still perform the access and can fail.
- `FENCE`, `ECALL` and `EBREAK` are no-ops. Also accept `FENCE.I` as a no-op. For opcode `0x0f`, accept function field `funct3` equal to 0 or 1 and ignore the other fields. For opcode `0x73`, accept only the exact words `00000073` and `00100073`.
- All other unsupported encodings cause an instruction error when fetched. Unreachable words need not encode valid instructions.

Before every instruction, perform these checks in order:

1. If `pc == 4*n`, finish successfully, including for an empty program.
2. If `pc` is not divisible by four or lies outside `[0, 4*n)`, report a PC error. Check jump and branch targets when the next fetch is attempted.
3. If 100000 instructions have already executed, report a limit error.
4. Fetch, decode and execute the next instruction.

Thus a program may finish after exactly 100000 instructions. A failed instruction ends only the current program; continue with the next input program.

## Encoding reference

Register fields are `rd = word[11:7]`, `rs1 = word[19:15]`, and `rs2 = word[24:20]`. The opcode is `word[6:0]`, `funct3` is `word[14:12]`, and `funct7` is `word[31:25]`. Braces below concatenate bit fields; `sext` means sign extension to 32 bits.

| Immediate | Bit fields |
| --- | --- |
| I | `sext(word[31:20])` |
| S | `sext({word[31:25], word[11:7]})` |
| B | `sext({word[31], word[7], word[30:25], word[11:8], 0})` |
| U | `{word[31:12], twelve zero bits}` |
| J | `sext({word[31], word[19:12], word[20], word[30:21], 0})` |

Opcode and function values in this table are hexadecimal.

| Instructions | Opcode | Function fields |
| --- | --- | --- |
| `ADD SUB SLL SLT SLTU XOR SRL SRA OR AND` | `33` | `funct3`: `0 0 1 2 3 4 5 5 6 7`; `funct7` is `20` for `SUB`/`SRA`, otherwise `00` |
| `ADDI SLTI SLTIU XORI ORI ANDI` | `13` | `funct3`: `0 2 3 4 6 7`; I-immediate |
| `SLLI SRLI SRAI` | `13` | `funct3`: `1 5 5`; `funct7`: `00 00 20`; shift count in bits `[24:20]` |
| `LB LH LW LBU LHU` | `03` | `funct3`: `0 1 2 4 5`; I-immediate |
| `SB SH SW` | `23` | `funct3`: `0 1 2`; S-immediate |
| `BEQ BNE BLT BGE BLTU BGEU` | `63` | `funct3`: `0 1 4 5 6 7`; B-immediate |
| `LUI` | `37` | U-immediate |
| `AUIPC` | `17` | U-immediate, added to the current `pc` |
| `JAL` | `6f` | J-immediate |
| `JALR` | `67` | `funct3 = 0`; I-immediate |
| `FENCE FENCE.I` | `0f` | `funct3`: `0 1`; no-op |
| `ECALL EBREAK` | `73` | Exact words given above; no-op |

## Output

Print one line per program. On success, print `OK`, followed by every nonzero register from `x1` through `x31`, in increasing register order. Each entry has the form ` xN=hhhhhhhh`, with a decimal register number and exactly eight lowercase hexadecimal digits. Omit zero-valued registers. If all registers are zero, print only `OK`.

On failure, print one of these lines, without a register dump:

- `ERROR INSTRUCTION hhhhhhhh`: an unsupported instruction at this `pc`.
- `ERROR MEMORY hhhhhhhh`: an unaligned or out-of-range load/store at this `pc`.
- `ERROR PC hhhhhhhh`: this attempted instruction address is invalid.
- `ERROR LIMIT hhhhhhhh`: the next instruction would exceed the execution limit; this is its `pc`.

Every address in an error line has exactly eight lowercase hexadecimal digits. A memory error reports the instruction address, not the data address.

## Examples

### 1. Addition and x0

Input:

```text
1
4
00500093 00700113 002081b3 06300013
```

Expected output:

```text
OK x1=00000005 x2=00000007 x3=0000000c
```

### 2. Wrapping and signed comparisons

Input:

```text
1
6
fff00093 00100113 002081b3 0020a233 0020b2b3 fff03313
```

Expected output:

```text
OK x1=ffffffff x2=00000001 x4=00000001 x6=00000001
```

### 3. Logical and arithmetic shifts

Input:

```text
1
7
800000b7 01f00113 0020d1b3 4020d233 002112b3 4040d313
0040d393
```

Expected output:

```text
OK x1=80000000 x2=0000001f x3=00000001 x4=ffffffff x5=80000000 x6=f8000000 x7=08000000
```

### 4. Loop: sum from 1 to 5

Input:

```text
1
5
00500093 00000113 00110133 fff08093 fe009ce3
```

Expected output:

```text
OK x2=0000000f
```

### 5. Call and return

Input:

```text
1
5
00700513 008000ef 00c0006f 00550513 00008067
```

Expected output:

```text
OK x1=00000008 x10=0000000c
```

### 6. Little-endian memory and signed loads

Input:

```text
1
10
10000093 fff00113 00208023 00008183 0000c203 00209123
00209283 0020d303 0020a223 0040a383
```

Expected output:

```text
OK x1=00000100 x2=ffffffff x3=ffffffff x4=000000ff x5=ffffffff x6=0000ffff x7=ffffffff
```

### 7. Upper immediates and the current PC

Input:

```text
1
4
00001097 00000117 123451b7 67818193
```

Expected output:

```text
OK x1=00001000 x2=00000004 x3=12345678
```

### 8. Fence and environment no-ops

Input:

```text
1
6
00100093 0ff0000f 0000100f 00000073 00100073 00108093
```

Expected output:

```text
OK x1=00000002
```

### 9. Unsupported instruction

Input:

```text
1
1
ffffffff
```

Expected output:

```text
ERROR INSTRUCTION 00000000
```

### 10. Unaligned load

Input:

```text
1
1
00202083
```

Expected output:

```text
ERROR MEMORY 00000000
```

### 11. Unaligned jump target

Input:

```text
1
1
0020006f
```

Expected output:

```text
ERROR PC 00000002
```

### 12. Infinite loop

Input:

```text
1
1
0000006f
```

Expected output:

```text
ERROR LIMIT 00000000
```
