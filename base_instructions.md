## Core stack ISA:

The following is the most stripped down core of a stack ISA reasonably possible. Here it is assumed the PC is just at memory address `0`.

- add
- logical left shift
- logical right shift
- and
- or
- not
- read
- write
- immediate

### Branches

It is possible to branch with the core ISA, but it would require some bit manipulation and is not particularly easy. This is one of the most important problems and should be dealt with first.

- branch equal
- branch less than unsigned
- branch less than signed

### Copy

The core ISA is primarily severely flawed because of the inability to copy values from lower in the stack. This will result in needing to write to memory to read multiple times. The next most optimal ISA is one with a copy instruction added.

- copy

### Calling

With the current instruction set, to make a call you would first have to have absolute code positions and then do `immediate immediate pc_write`. Then to return you would do `pc_write`, but if you had to return data on the stack then since a `rotate` instruction is not available, it would either need to `write` the return address to memory (which without stack registers would prevent efficient recursion) and then re-read it later or it would require using `copy` on the return address and the caller disposing of the return address, which probably takes two instructions with this system. To avoid this mess, in particular because returning values on the stack is essential, instructions for calling and returning need to be added.

- call
- return

### Registers

Currently, it is possible to write to the lower addresses of memory to use them like registers. One potential problem is that, when dealing with accumulators, a caller or callee has to push the registers it is using to a stack. This can be mitigated by allowing some registers to be pushed automatically to the callstack.

### Carry

The lack of a carry bit complicates 64-bit integer handling on 32-bit systems. For 64-bit systems this is not a signifcant concern since any value above 64-bits is likely not a counter and is some sort of GUID. Since 32-bit (and maybe even 16-bit) use cases are still common, it may be necessary to add a carry bit that is preserved on the call stack. To use this bit, the instruction `carry` can be added which adds a 1 to the TOS if the carry bit is set. This allows multi-word addition with one extra instruction. We can similarly define (if desired) an additional `borrow` instruction which subtracts one and then adds the carry, which would allow borrowing during subtraction in one instruction, but it can be done in two instructions otherwise by decrementing and then carrying.

- carry
- borrow
- branch carry

### Loops

Any time we are in a loop, we are probably going to be working with registers. This is necessary because a repetitive process has to work on some accumulators that end up in the same stack orientation every iteration. This is a weak point of stack architectures, because the loop iterates a dynamic amount of times. A recursive solution would be "ideal", but that can result in stacks growing to absurd heights to store data just so it can be processed in the right order (the weakness of a stack architecture, data ordering). In a loop, it is preferable to process the data in the order required and utilize registers, while also utilizing the stack for large expressions and other things which are favorable for it. Some things can be done to alleviate the issue loops present.

One of those things could be to push a set of registers onto the stack at the beginning of the loop in the order for processing. If these registers are overwritten during the course of the loop, that can be utilized to modify the values that enter in again at the beginning. This is similar to a fold/reduce expression in that accumulators, variables, and constant values come back in at the beginning. We can implement this using a special instruction that copies n registers from the beginning of memory onto the stack. Since the first register is the PC, this means that at the end of the loop one can just jump or branch back to this special instruction location, triggering the loop again. Since this instruction will need to have several variants for each number of registers to copy, it will take up several instruction slots.

If we use a dedicated loop instruction, we can make it so any time we move back to the beginning of the loop that the loop puts the registers on the stack. This is especially helpful because the first iteration of the loop gets the initial inputs, just as in the above solution, but even more useful because now we can automatically return to the beginning without specifically requiring the PC, which is more useful for branching paths conditionally exiting the loop or doing other things. This does make returning take an extra instruction to `unloop` the loop, which is not otherwise necessary since no loop stack or any special loop state exists.

Another solution is to use tail-call recursion to implement all loops. In this case, the loop control flow is controlled using existing control-flow instructions. To break the loop, one only needs to return. To continue the loop, one only needs to jump to the beginning. In this case we only need a special jump instruction that copies registers onto the stack to avoid manual register reading on every iteration. This solution is the most practical as it avoids adding new instructions and calls in stack machines are cheap.

- jmpload
  - Jump and load `n` non-pc registers to the stack.

### Rotate

### Coroutines