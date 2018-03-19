## Core stack ISA:

The following is the most stripped down core of a stack ISA reasonably possible.

1. add
2. subtract
3. logical left shift
4. logical right shift
5. and
6. or
7. not
8. read
9. write
10. pc write
11. immediate

### Copy

The core ISA is primarily severely flawed because of the inability to copy values from lower in the stack. This will result in needing to write to memory to read multiple times. The next most optimal ISA is one with a copy instruction added.

12. copy

### Carry

The lack of a carry bit complicates 64-bit integer handling on 32-bit systems. For 64-bit systems this is not a signifcant concern since any value above 64-bits is likely not a counter and is some sort of GUID.

