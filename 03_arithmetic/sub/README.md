# SUB Operation and EFLAGS

The `sub` instruction subtracts the source from the destination and updates CF, OF, SF, ZF, PF and AF. CF reports an unsigned borrow (first number smaller than the second), and OF reports signed overflow. IF in the GDB output is set by the operating system and is not affected by `sub`.

## Program 1: sub1.asm (8-bit, unsigned borrow)

**What it does:** subtracts 80 (0x50) from 50 (0x32) in AL using 8-bit arithmetic.

**Result:** AL = 0xE2 (226 unsigned, -30 signed)

**EFLAGS before `sub`:** `0x202` = `[ IF ]`
**EFLAGS after `sub`:** `0x287` = `[ CF PF SF IF ]`

```
  0011 0010   (50 = 0x32)
- 0101 0000   (80 = 0x50)
-----------
  1110 0010   (0xE2)
```

| Flag | Status | Why |
|------|--------|-----|
| CF | 1 | Unsigned 50 < 80, so a borrow out of bit 7 is needed and the result wraps to 226 |
| OF | 0 | Signed 50 - 80 = -30 fits in the range -128 to +127, so no signed overflow |
| SF | 1 | Bit 7 of 0xE2 (11100010) is 1 |
| ZF | 0 | The result is not zero |
| PF | 1 | Low byte 11100010 has four 1-bits, an even count |
| AF | 0 | Low nibble 2 - 0 needs no borrow from bit 4 |

**Note:** the same bytes read as unsigned (226) or signed (-30). CF reports the unsigned view (wrapped) and OF the signed view (no problem), which is why CF = 1 while OF = 0. The later `xor ebx, ebx` would overwrite the flags, so they were read immediately after `sub`.

### GDB output
```
(gdb) break _start
Breakpoint 1 at 0x8049000: file sub1.asm, line 11.
(gdb) run
Breakpoint 1, _start () at sub1.asm:11
11          mov al, [num1]
(gdb) info registers eflags
eflags         0x202               [ IF ]
(gdb) si
12          sub al, [num2]       ; al = 50 - 80
(gdb) si
13          mov [result], al   ;
(gdb) info registers eflags
eflags         0x287               [ CF PF SF IF ]
(gdb) p/x $al
$1 = 0xe2
```
## Program 2: sub2.asm (16-bit, unsigned borrow)

**What it does:** subtracts 2000 (0x07D0) from 1000 (0x03E8) in AX using 16-bit arithmetic.

**Result:** AX = 0xFC18 (64,536 unsigned, -1000 signed)

**EFLAGS before `sub`:** `0x202` = `[ IF ]`
**EFLAGS after `sub`:** `0x287` = `[ CF PF SF IF ]`

```
  0000 0011 1110 1000   (1000 = 0x03E8)
- 0000 0111 1101 0000   (2000 = 0x07D0)
---------------------
  1111 1100 0001 1000   (0xFC18)
```

| Flag | Status | Why |
|------|--------|-----|
| CF | 1 | Unsigned 1000 < 2000, so a borrow out of bit 15 is needed and the result wraps to 64,536 |
| OF | 0 | Signed 1000 - 2000 = -1000 fits in the 16-bit signed range, so no signed overflow |
| SF | 1 | Bit 15 of 0xFC18 is 1 |
| ZF | 0 | The result is not zero |
| PF | 1 | PF checks only the low byte: 0x18 = 00011000 has two 1-bits, an even count |
| AF | 0 | Low nibble 8 - 0 = 8 needs no borrow from bit 4 |

**Note:** as in sub1, CF reports the unsigned view (wrapped) and OF the signed view (valid -1000). The later `xor ebx, ebx` would overwrite the flags, so they were read immediately after `sub`.

### GDB output
```
(gdb) break _start
Breakpoint 1 at 0x8049000: file sub2.asm, line 11.
(gdb) run
Breakpoint 1, _start () at sub2.asm:11
11          mov ax, [num1]
(gdb) info registers eflags
eflags         0x202               [ IF ]
(gdb) si
12          sub ax, [num2]       ; AX = 1000 - 2000
(gdb) si
13          mov [result], ax
(gdb) info registers eflags
eflags         0x287               [ CF PF SF IF ]
(gdb) p/x $ax
$1 = 0xfc18
```

## Summary

| Program | Operation | Result | Flags set | Flags cleared |
|---|---|---|---|---|
| sub1.asm | 50 - 80 (8-bit) | 0xE2 | CF, PF, SF | OF, ZF, AF |
| sub2.asm | 1000 - 2000 (16-bit) | 0xFC18 | CF, PF, SF | OF, ZF, AF |

Both programs borrow in the unsigned view (CF = 1) but are valid in the signed view (OF = 0).
