# Core stack ISA

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

## Branches

It is possible to branch with the core ISA, but it would require some bit manipulation and is not particularly easy. This is one of the most important problems and should be dealt with first.

- branch equal
- branch less than unsigned
- branch less than signed

## Copy

The core ISA is primarily severely flawed because of the inability to copy values from lower in the stack. This will result in needing to write to memory to read multiple times. The next most optimal ISA is one with a copy instruction added.

- copy

## Calling

With the current instruction set, to make a call you would first have to have absolute code positions and then do `immediate immediate pc_write`. Then to return you would do `pc_write`, but if you had to return data on the stack, then since a `rotate` instruction is not available, it would either need to `write` the return address to memory (which without stack registers would prevent efficient recursion) and then re-read it later or it would require using `copy` on the return address and the caller disposing of the return address, which probably takes two instructions with this system. To avoid this mess, in particular because returning values on the stack is essential, instructions for calling and returning need to be added.

- call
- return

## Registers

Currently, it is possible to write to the lower addresses of memory to use them like registers. One potential problem is that, when dealing with accumulators, a caller or callee has to push the registers it is using to a stack. This can be mitigated by allowing some registers to be pushed to the stack on a call. This requires no additional instructions, but an additional instruction is required to be able to update registers below in the call stack. We can just treat the lowest memory addresses specially.

- update

## Carry (optional)

The lack of a carry bit complicates 64-bit integer handling on 32-bit systems. For 64-bit systems this is not a signifcant concern since any value above 64-bits is likely not a counter and is some sort of GUID, bit vector, or SIMD In Register (SIR) construct. Since 32-bit (and maybe even 16-bit) use cases are still common, it may be necessary to add a carry bit that is preserved on the call stack. To use this bit, the instruction `carry` can be added which adds a 1 to the TOS if the carry bit is set. This allows multi-word addition with one extra instruction. We can similarly define (if desired) an additional `borrow` instruction which subtracts one and then adds the carry, which would allow borrowing during subtraction in one instruction, but it can be done in two instructions otherwise by decrementing and then carrying.

- carry
- borrow
- branch carry

## Loops

Any time we are in a loop, we are probably going to be working with registers. This is necessary because a repetitive process has to work on some accumulators that end up in the same stack orientation every iteration. This is a weak point of stack architectures, because the loop iterates a dynamic amount of times. A recursive solution would be "ideal", but that can result in stacks growing to absurd heights to store data just so it can be processed in the right order (the weakness of a stack architecture, data ordering). In a loop, it is preferable to process the data in the order required and utilize registers, while also utilizing the stack for large expressions and other things which are favorable for it. Some things can be done to alleviate the issue loops present.

One of those things could be to push a set of registers onto the stack at the beginning of the loop in the order for processing. If these registers are overwritten during the course of the loop, that can be utilized to modify the values that enter in again at the beginning. This is similar to a fold/reduce expression in that accumulators, variables, and constant values come back in at the beginning. We can implement this using a special instruction that copies n registers from the beginning of memory onto the stack. Since the first register is the PC, this means that at the end of the loop one can just jump or branch back to this special instruction location, triggering the loop again. Since this instruction will need to have several variants for each number of registers to copy, it will take up several instruction slots.

If we use a dedicated loop instruction, we can make it so any time we move back to the beginning of the loop that the loop puts the registers on the stack. This is especially helpful because the first iteration of the loop gets the initial inputs, just as in the above solution, but even more useful because now we can automatically return to the beginning without specifically requiring the PC, which is more useful for branching paths conditionally exiting the loop or doing other things. This does make returning take an extra instruction to `unloop` the loop, which is not otherwise necessary since no loop stack or any special loop state exists.

Another solution is to use tail-call recursion to implement all loops. In this case, the loop control flow is controlled using existing control-flow instructions. To break the loop, one only needs to return. To continue the loop, one only needs to jump to the beginning. In this case we only need a special jump instruction that copies registers onto the stack to avoid manual register reading on every iteration. This solution is the most practical as it avoids adding new instructions and calls in stack machines are very cheap.

We could also imagine adding an instruction to speed up tail-call recursive loops by performing a register update, comparison, and jump load/return in one instruction. This is something that may be helpful to have after analysis of output assembly from programs, but it is likely to be useful since iteration over any data structure is likely to require this common routine.

- jump load
  - Jump and load `n` non-pc registers to the stack.

## Rotate

It is possible to rotate items on the stack to the top above a certain depth. This allows re-ordering of the stack. Re-ordering the stack is useful for creating accumulators that reside on the stack that are updated multiple times. For values that are computed once and used multiple times, but never updated, copying is useful.

Rotation is expensive:

- It requires the top registers on the stack to be able to shift down conditionally.
- It disallows the top registers to be stored or cached in a larger memory due to modifying all elements simultaneously.
- It takes more die area to implement than copy.
- It has a bad effect on power consumption compared to copy.
- It takes a lot of instruction space or makes decoding harder by forcing more instructions to be dynamically sized.
- It means the stack ordering will have to be tracked when compiling in a more stateful way, increasing compile times.
- It would be slower to accumulate than a register machine because the critical path of lookup and accumulate is longer.
- It would require more instructions than a register machine because every accumulate requires a rotate.

If we can help it, we want to avoid rotation. Backing the stack with real memory is very important for blazing fast threading and coroutines in stack processors. With copy instructions, anything below the top two elements will never be modified, so little stack memory will ever need to be flushed to main memory and the hardware to mediate caching and writing out the stack will be simpler.

How can rotation be avoided? The register file is another way to store accumulators that doesn't require rotation. However, to accumulate to the register file, we must perform a read-accumulate-write (RAW) operation, which is currently a prohibitive 3 (or 4 if you count the instruction space of the index) instructions. Although this operation could be executed faster than a rotate-accumulate in theory, it increases the instruction decoding logic complexity greatly.

## Efficient accumulation

We need to be able to accumulate to the register file ASAP and with as few instructions as possible. Firstly, we don't want to expand the instruction space to include accumulate variants for every register if possible. To reuse existing accumulate operations, it is necessary to combine the writeback step of the accumulation with the existing operation using a stateful register that says where we will writeback to. We could implement this by adding an instruction which fetches the register and then forces the next operation to write its result to the register file, but if we do this then it becomes impossible to do any accumulation operation that requires more than one instruction or requires the accumulator to be the second element on the stack.

An example of needing the accumulator to be the second element is when the accumulator is a **minuend** and the current top-of-stack (TOS) is the **subtrahend**. In this case, the algorithm is subtracting from some number repeatedly, which requires the **subtrahend** to be TOS.

This can be solved by:

1. Creating alternate versions of every non-commutative instruction that flips the stack arguments.
    - Doesn't help anything else as we can produce stack values in the correct order.
    - May require expanding the base instruction width (not good).
2. Creating a second register accumulate instruction that inserts the register underneath the TOS.
    - This doesn't significantly increase decoder complexity as both of these instructions can share the same upper bits.

This may not even need to be solved. The primary instructions to worry about would be subtraction and shifting. Subtraction can be solved through optimization since the **subtrahend** can be accumulated using addition. Shifting is interesting since many workloads don't use a significant amount of shifting, but some use it incredibly heavily.

An important thing to note is that both subtraction and shift accumulate operations will almost always want the accumulator to be under the TOS. Since no previous requirement made them the operand order they are, they can simply be swapped, solving the problem.

This implementation is also good because a simpler design is still possible which executes the read and accumulate in seperate cycles if die area is prioritized over performance. It also doesn't have state if the accumulate is always performed in one cycle, which means dedicated threading and coroutines can still swap without any additional state having to be written or read on a single-cycle notice.

We can also have additional accumulates that wait multiple cycles before performing the write-back.

- accumulate
  - Read a register to the TOS and stores the ALU output of the next instruction to the register.
- accumulate after n
  - Like `accumulate`, but waits more than one instruction before accumulating.

## Hashing

Hashing is incredibly critical on modern systems. There are different types of hashes required for different things. For instance, to hash sensitive data you may need a cryptographic hash. When the hash itself is not accessible to an attacker, you might only need a Message Authentication Code (MAC). You might need a hash for a hashtable where an attacker cannot control the input data, in that case only speed is a concern so you could use a non-cryptographic hash. We also sometimes want to make hashes that cause collisions on purpose, such as Locality Sensitive Hashing (LSH). All of these techniques are used for different things.

On almost all systems and all situations, it is common to use tables for lookup. The hash table is so common that, to support its use case, a processor should probably provide primitives for its hashing algorithm. For this, we would like to support two use cases:

- Input data is controlled by potential attackers that wish to cause DoS attacks.
- Input data is trusted/cannot be controled by attackers.

For the first case, SipHash seems to be the current state-of-the-art. It takes about 100-200 cycles on most desktop and server class processors for 8-64 bytes. The SipHash paper (Jean-Philippe Aumasson and Daniel J. Bernstein) suggests that it should be possible to implement a maximally fast SipHash with `1/8` cycles per byte (`64` bits per cycle) and a single cycle for finalization. We can imagine that we have a single instruction that implements a few rounds of SipHash. If it implements 2 rounds, then we can implement SipHash-2-4 by simply running the instruction for each 64-bit input, doing special handling for the last byte padding (if necessary),
and then (after xoring 0xFF) just running two rounds of the instruction for the finalization. We could also just make the instruction implement 4 rounds of SipHash so that we can do SipHash-4-4 with one instruction for finalization. Either way, this proves that it is possible to add a DoS-attack resistant hash to a hashtable that can be executed in 14 cycles:

- Four inital XOR (8 cycles)
  - One cycle to fetch/copy the item
  - One cycle to perform the constant XOR
- One load and multiple SipRound (2 cycles)
  - Load requires a cycle
  - The multiple SipRound are all part of the one SipHash instruction
- Constant 0xFF load and multiple SipRound (1 cycle)
- Final output by XORing all 4 64-bit numbers (3 cycles)

By adding two specialized instructions, this process could also take a total of 5 cycles:

- Load two halves of the key (2 cycles)
- Load first word (1 cycle)
- Initialize SipHash (1 cycle)
- Finalize SipHash (1 cycle)

This would require:

- An instruction that consumes 3 words to initialize the SipHash and run one round of it
- An instruction that consumes 1 word and runs one round of SipHash
- An instruction that spits out the finalized hash

You could also potentially go even further and add another processor word that initializes the hash and finalizes it in one cycle, which would take a total of 4 cycles, but investigation would need to be done to see if that really made any difference.

At this point there would be no good reason to write a high speed hash since SipHash is fast enough, therefore you no longer need to worry about the tradeoff between hashtable performance and security.

For good measure, the state of the SipHash should be stored on the call stack. We could store it on the main stack, but then it would be an accumulator that is taking up space, so this is not an option. We can store it in the registers, but then we have to choose which registers to store it in. We could store it alongside the general registers, but then we have an additional 4 registers to push to the call stack. I believe storing either one of these ways has tradeoffs and should be evaluated in an actual system with actual software to see which is better. Either way, the instructions are the same:

- sip init
- sip round
- sip finalize

## Coroutines

Coroutines are trivial to implement on a stack machine as they would use the same pathways as calls would, but they would also need to point at the coroutine's value and call stacks (context) in memory. When a coroutine yields, it needs to be able to access the context of the coroutine that called it to switch to it. It could do this by simply letting the caller pass this on the stack. In that case, every coroutine context switch would require the callee to track the new context of the coroutine. It is probably preferable then that accessing the main memory addresses of the contexts is done automatically by a special coroutine instruction which swaps the current coroutine addresses with the ones on the stack. Since it will always update after execution, it is the responsibility of the caller to update any old contexts (which are now invalid) with the new context information.

- coroutine call
  - After execution, the coroutine will leave its 2 addresses on the TOS with any return values from yielding below it.

## Threading

`coroutine call` alone cannot allow threading because it writes the context to the stack of the target coroutine. That requires the target coroutine to know it is being called. To solve this, the coroutine system must allow the retrieval and assigning of context. An interrupt actually happens in the same context as the currently executing coroutine, so it just needs to retrieve its own context. The context retrieval always assumes you want the context of the caller because that will be the interrupted coroutine, but it can't know the beginnig of the value stack of the caller, so it uses the TOS address when the instruction is executed. If the interrupt executes this with an empty stack it will get exactly the correct context, but it will otherwise require modifying the value stack pointer.

- set context
  - After execution, the coroutine will leave nothing on the TOS, meaning the target coroutine can be resumed.
- get context
  - Allows a coroutine to get its own context, but using the call stack below instead. The value stack address is TOS.
  - This allows interrupts to perform context switches by calling `get context` before making any calls.

## UARC Async and Sync

To support UARC communication, talking over a bus is important. We can communicate in two modes: word and stream.

In word mode, the core should be able to synchronize on word arrival. This can be supported even when threading because the context will say to return to the synchronize instruction which simply only consumes stack parameters and advances the PC once it has recieved something. Waiting for word arrival synchronously shouldn't cause a context switch because if it was expected that the I/O operation would take a while then they would use async methods instead.

- recieve synchronous
- send synchronous

Async word support would actually be controlled by interrupt handlers. Since each core only executes one thread at a time, checking if something came in over a bus can be done by checking a flag in main memory. This doesn't even require a coroutine since it is such a simple check. However, coroutines will be useful if multiple I/O operations have to happen back-to-back with processing inbetween. If there in an operating system then it should potentially allowing multiple interrupt handlers to be called on an interrupt using its own handler, but for an application running without an OS it should be able to directly set the handlers.

- set recieve interrupt handler

Streaming mode should always be asynchronous, but be accepted synchronously. Sending/recieving streams and sending words asynchronously all have the same problem: an asynchronous operation is initiated, but we probably want to know when it completes. To handle this problem, we will use interrupts as we did before.

- stream out
- stream in
- send asynchronous
- set in interrupt handler
- set out interrupt handler
- set send interrupt handler

Note that all interrupt handlers can set interrupts on a per-bus basis. To turn the interrupts off again, set the handler to `-1`.

## Extras

Some instructions are missing that would be helpful to add to the ISA (from greatest performance/program density impact to least):

- multiplication
- subtraction (not 1 add add)
- floating point addition/subtraction/multiplication (not division)
- negation (not 1 add)

Some instructions that are simple enough to go in the ISA, but might be better suited for an application-specific core:

- SIMD
- rotate left/right (matters more for crypto)
- advanced bit manipulation (matters more for I/O and crypto)

Some instructions that would be good to add to an on-die coprocessor talking over a UARC bus (in no particular order):

- float and integer division
- trigonometry
- square root
- exponentiation
- logarithm