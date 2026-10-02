# ADD Operation and EFLAGS

The `add` instruction updates CF, OF, SF, ZF, PF and AF based on the result.
CF reports unsigned overflow (carry out of the top bit), and OF reports signed overflow.
IF in the GDB output is set by the operating system and is not affected by `add`.

## Program 1: add1.asm (8-bit signed overflow)

**What it does:** adds 120 (0x78) and 10 (0x0A) in AL using 8-bit arithmetic.

**Result:** AL = 0x82 (130 unsigned, -126 signed)

**EFLAGS after `add`:** `0xa96` = `[ PF AF SF IF OF ]`

```
  0111 1000   (120 = 0x78)
+ 0000 1010   ( 10 = 0x0A)
-----------
  1000 0010   (130 = 0x82)
```

| Flag | Status | Why |
|------|--------|-----|
| CF | 0 | Unsigned 120 + 10 = 130 fits in 8 bits (max 255), so no carry out of bit 7 |
| OF | 1 | Positive + positive gave a negative (0x78 + 0x0A = 0x82). The signed result 130 exceeds +127 |
| SF | 1 | Bit 7 of 0x82 (10000010) is 1 |
| ZF | 0 | The result is not zero |
| PF | 1 | Low byte 10000010 has two 1-bits, an even count |
| AF | 1 | Low nibble 8 + A = 18, which carries from bit 3 into bit 4 |

**Note:** IF is set by the operating system, not by the `add` instruction.

### GDB output
```
(gdb) break _start
Breakpoint 1 at 0x8049000: file add1.asm, line 17.
(gdb) run
Breakpoint 1, _start () at add1.asm:17
17          mov al, [num1]
(gdb) si
18          add al, [num2]       ; al = num1 + num2        10000010
(gdb) si
19          mov [result], al
(gdb) info registers eflags
eflags         0xa96               [ PF AF SF IF OF ]
(gdb) p/x $al
$1 = 0x82
```

## Program 2: add2.asm (16-bit addition, no arithmetic flags set)

**What it does:** adds 32000 (0x7D00) and 500 (0x01F4) in AX using 16-bit arithmetic.

**Result:** AX = 0x7EF4 (32500)

**EFLAGS after `add`:** `0x202` = `[ IF ]` (all arithmetic flags cleared)

```
  0111 1101 0000 0000   (32000 = 0x7D00)
+ 0000 0001 1111 0100   (  500 = 0x01F4)
---------------------
  0111 1110 1111 0100   (32500 = 0x7EF4)
```

| Flag | Status | Why |
|------|--------|-----|
| CF | 0 | Unsigned 32,500 fits in 16 bits (max 65,535), so there is no carry out of bit 15 |
| OF | 0 | Positive + positive = positive. 32,500 is within the signed 16-bit range (max 32,767), so no signed overflow |
| SF | 0 | Bit 15 of 0x7EF4 is 0, so the result is not negative |
| ZF | 0 | The result (32,500) is not zero |
| PF | 0 | PF only checks the low byte: 0xF4 = 11110100 has five 1-bits, which is an odd count |
| AF | 0 | Low nibbles: 0x0 + 0x4 = 0x4, which fits in 4 bits, so nothing carries from bit 3 into bit 4 |

**Note:** IF is set by the operating system, not by `add`. Bit 1 is a reserved bit that is always 1, which is why the value is `0x202`. The later `xor ebx, ebx` in the program would overwrite the flags, so EFLAGS was read immediately after the `add` instruction.

### GDB output
```
(gdb) break _start
Breakpoint 1 at 0x8049000: file add2.asm, line 11.
(gdb) run
Breakpoint 1, _start () at add2.asm:11
11          mov ax, [num1]
(gdb) si
12          add ax, [num2]       ; AX = num1 + num2
(gdb) si
13          mov [result], ax
(gdb) info registers eflags
eflags         0x202               [ IF ]
(gdb) p/x $ax
$1 = 0x7ef4
```

## Summary

| Program | Operation | Result | Flags set | Flags cleared |
|---|---|---|---|---|
| add1.asm | 120 + 10 (8-bit) | 0x82 | PF, AF, SF, OF | CF, ZF |
| add2.asm | 32000 + 500 (16-bit) | 0x7EF4 | none | CF, OF, SF, ZF, PF, AF |

add1 shows a signed overflow (OF = 1) without an unsigned carry (CF = 0). add2 shows a clean result where no arithmetic flag is raised.