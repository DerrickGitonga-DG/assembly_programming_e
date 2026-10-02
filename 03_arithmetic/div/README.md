# DIV Operation and EFLAGS

The `div` instruction performs unsigned division. For 8-bit `div`, the dividend is AX, the quotient goes to AL and the remainder to AH. Per the Intel manual, **CF, OF, SF, ZF, PF and AF are all undefined** after `div`. This means the flags cannot be explained from the result, and any value shown in GDB is implementation-specific. IF is set by the operating system and is not affected by `div`.

## Program 1: div1.asm (8-bit unsigned division)

**What it does:** divides 100 (in AX) by 7 (in BL) using 8-bit `div`.

**Result:** AL = 0x0E (quotient 14), AH = 0x02 (remainder 2), AX = 0x020E

**EFLAGS before `div`:** `0x202` = `[ IF ]`
**EFLAGS after `div`:** `0x212` = `[ AF IF ]`

```
100 / 7 = 14 remainder 2
AX = 0x0064  ->  AH:AL = 0x02:0x0E
```

| Flag | Status after div | Why |
|------|------------------|-----|
| CF | 0 (undefined) | Intel leaves CF undefined after `div`, so its value is not a result of the division |
| OF | 0 (undefined) | Undefined after `div` |
| SF | 0 (undefined) | Undefined after `div` |
| ZF | 0 (undefined) | Undefined after `div`. The result is non-zero, but `div` does not set ZF from it |
| PF | 0 (undefined) | Undefined after `div` |
| AF | 1 (undefined) | Undefined after `div`. It changed from 0 to 1 in this run, but this is implementation-specific behavior, not a meaningful result of 100 / 7 |

**Note:** the only flag that changed was AF (0x202 to 0x212). Since the flags are undefined after `div`, the CPU is allowed to change them in ways that carry no meaning, so I do not attribute AF = 1 to the division result. The program's later `xor ebx, ebx` would overwrite the flags, so they were read immediately after `div`. The quotient (14) fits in 8 bits, so no divide error occurred.

### GDB output
```
(gdb) break _start
Breakpoint 1 at 0x8049000: file div1.asm, line 11.
(gdb) run
Breakpoint 1, _start () at div1.asm:11
11          mov ax, [dividend]  ; ax = 100
(gdb) info registers eflags
eflags         0x202               [ IF ]
(gdb) si
12          mov bl, [divisor]   ; al = 7
(gdb) si
13          div bl              ; al = 14, ah = 2
(gdb) si
16          mov eax, 1
(gdb) info registers eflags
eflags         0x212               [ AF IF ]
(gdb) p/x $ax
$1 = 0x20e
(gdb) p/x $al
$2 = 0xe
(gdb) p/x $ah
$3 = 0x2
```

## Program 2: div2.asm (16-bit unsigned division)

**What it does:** divides the 32-bit value DX:AX = 50000 (DX = 0, AX = 0xC350) by 300 (BX = 0x012C) using 16-bit `div`.

**Result:** AX = 0x00A6 (quotient 166), DX = 0x00C8 (remainder 200)

**EFLAGS before `div`:** `0x202` = `[ IF ]`
**EFLAGS after `div`:** `0x212` = `[ AF IF ]`

```
50000 / 300 = 166 remainder 200
check: 300 * 166 + 200 = 49800 + 200 = 50000
DX:AX = 0x0000C350  ->  AX = 0x00A6 (quotient), DX = 0x00C8 (remainder)
```

| Flag | Status after div | Why |
|------|------------------|-----|
| CF | 0 (undefined) | Undefined after `div`, so its value is not a result of the division |
| OF | 0 (undefined) | Undefined after `div` |
| SF | 0 (undefined) | Undefined after `div` |
| ZF | 0 (undefined) | Undefined after `div`. The result is non-zero, but `div` does not set ZF from it |
| PF | 0 (undefined) | Undefined after `div` |
| AF | 1 (undefined) | Undefined after `div`. It changed from 0 to 1, but this is implementation-specific behavior, not a meaningful result of 50000 / 300 |

**Note:** because the flags are undefined after `div`, the values GDB shows carry no meaning about the result. As in div1, only AF changed on this CPU. The quotient (166) fits in 16 bits, so no divide error occurred. The later `xor ebx, ebx` would overwrite the flags, so they were read immediately after `div`. IF is set by the operating system.

### GDB output
```
(gdb) break _start
Breakpoint 1 at 0x8049000: file div2.asm, line 12.
(gdb) run
Breakpoint 1, _start () at div2.asm:12
12          mov ax, [dividend]  ; AX = 50000
(gdb) info registers eflags
eflags         0x202               [ IF ]
(gdb) si
13          mov dx, [highpart]  ; DX = 0
(gdb) si
14          mov bx, [divisor]   ; BX = 300
(gdb) si
15          div bx              ; AX = quotient, 166
(gdb) si
19          mov eax, 1     ;sys call number to exit
(gdb) info registers eflags
eflags         0x212               [ AF IF ]
(gdb) p/x $ax
$1 = 0xa6
(gdb) p/x $dx
$2 = 0xc8
```

## Summary

| Program | Operation | Result | Flags after (undefined) |
|---|---|---|---|
| div1.asm | 100 / 7 (8-bit) | AL = 14, AH = 2 | AF set, others clear |
| div2.asm | 50000 / 300 (16-bit) | AX = 166, DX = 200 | AF set, others clear |

In both programs the flags are undefined per the Intel manual, so none of them can be explained from the quotient or remainder.