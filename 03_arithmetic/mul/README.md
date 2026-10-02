# MUL Operation and EFLAGS

The `mul` instruction performs unsigned multiplication. For 8-bit `mul`, AL is multiplied by the operand and the 16-bit result goes in AX. **CF and OF** are set if the upper half of the result (AH) is non-zero, and cleared otherwise. **SF, ZF, AF and PF are undefined** per the Intel manual, so they cannot be explained from the result. IF is set by the operating system and is not affected by `mul`.

## Program 1: mul1.asm (8-bit unsigned multiply)

**What it does:** multiplies 25 (AL) by 10 (memory byte) using 8-bit `mul`.

**Result:** AX = 0x00FA (250), so AL = 0xFA and AH = 0x00

**EFLAGS before `mul`:** `0x202` = `[ IF ]`
**EFLAGS after `mul`:** `0x202` = `[ IF ]`

```
  0001 1001   (25 = 0x19)
x 0000 1010   (10 = 0x0A)
-----------
  0000 0000 1111 1010   (250 = 0x00FA)
```

| Flag | Status after mul | Why |
|------|------------------|-----|
| CF | 0 | AH = 0, so the product fits in the lower half (AL). Nothing spilled into AH |
| OF | 0 | Same condition as CF: the upper half of the result is zero |
| SF | 0 (undefined) | Undefined after `mul`, so not derived from the result |
| ZF | 0 (undefined) | Undefined after `mul`. The result is non-zero, but `mul` does not set ZF from it |
| PF | 0 (undefined) | Undefined after `mul` |
| AF | 0 (undefined) | Undefined after `mul` |

**Note:** CF and OF are the only meaningful flags here. On this CPU the undefined flags happened to stay unchanged, but that is not guaranteed. 250 would be -6 as a signed byte, but `mul` is unsigned, and 250 fits in 8 bits unsigned (max 255), so there is no spill into AH. The later `xor ebx, ebx` would overwrite the flags, so they were read immediately after `mul`.

### GDB output
```
(gdb) break _start
Breakpoint 1 at 0x8049000: file mul1.asm, line 11.
(gdb) run
Breakpoint 1, _start () at mul1.asm:11
11          mov al, [num1]      ; al = first operand
(gdb) info registers eflags
eflags         0x202               [ IF ]
(gdb) si
12          mul byte [num2]     ; ax = al * num2 (25 * 10 = 250)
(gdb) si
13          mov [result], ax    ; store result in memory
(gdb) info registers eflags
eflags         0x202               [ IF ]
(gdb) p/x $ax
$1 = 0xfa
(gdb) p/x $al
$2 = 0xfa
```
## Program 2: mul2.asm (16-bit unsigned multiply, result spills into DX)

**What it does:** multiplies 3000 (AX) by 200 (memory word) using 16-bit `mul`.

**Result:** DX:AX = 0x000927C0 (600,000), so DX = 0x0009 and AX = 0x27C0

**EFLAGS before `mul`:** `0x202` = `[ IF ]`
**EFLAGS after `mul`:** `0xa03` = `[ CF IF OF ]`

```
  0000 1011 1011 1000   (3000 = 0x0BB8)
x 0000 0000 1100 1000   ( 200 = 0x00C8)
---------------------
  0000 0000 0000 1001  0010 0111 1100 0000   (600,000 = 0x000927C0)
  |------ DX = 9 ---|  |------ AX = 0x27C0 ---|
```

| Flag | Status after mul | Why |
|------|------------------|-----|
| CF | 1 | DX = 9 is non-zero, so the product (600,000) does not fit in 16 bits and spilled into DX |
| OF | 1 | Same condition as CF: the upper half of the result is non-zero |
| SF | 0 (undefined) | Undefined after `mul`, so not derived from the result |
| ZF | 0 (undefined) | Undefined after `mul`. The result is non-zero, but `mul` does not set ZF from it |
| PF | 0 (undefined) | Undefined after `mul` |
| AF | 0 (undefined) | Undefined after `mul` |

**Note:** CF and OF are the only meaningful flags. Compared with mul1 (CF = OF = 0 because AH = 0), here the result needed 20 bits, so DX is non-zero and both flags are set. The later `xor ebx, ebx` would overwrite the flags, so they were read immediately after `mul`. IF is set by the operating system.

### GDB output
```
(gdb) break _start
Breakpoint 1 at 0x8049000: file mul2.asm, line 11.
(gdb) run
Breakpoint 1, _start () at mul2.asm:11
11          mov ax, [num1]      ; moving the value from num1 to ax
(gdb) info registers eflags
eflags         0x202               [ IF ]
(gdb) si
12          mul word [num2]     ; DX:AX = AX * num2
(gdb) si
13          mov [result], ax    ; lower 16 bits
(gdb) info registers eflags
eflags         0xa03               [ CF IF OF ]
(gdb) p/x $ax
$1 = 0x27c0
(gdb) p/x $dx
$2 = 0x9
```

## Summary

| Program | Operation | Result | CF / OF | Why |
|---|---|---|---|---|
| mul1.asm | 25 * 10 (8-bit) | AX = 0x00FA | 0 / 0 | Upper half (AH) is zero |
| mul2.asm | 3000 * 200 (16-bit) | DX:AX = 0x000927C0 | 1 / 1 | Upper half (DX) is non-zero |

SF, ZF, AF and PF are undefined after `mul` in both programs.