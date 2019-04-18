# Assembly

`u1` needs to have an assembly language so that it is possible to test optimizations in instruction size.

The assembly format should take highly generalized instructions. For instance, we do not want to include an `increment` instruction initially, even though it is likely we might need it. This is so that we make no assumptions. The backend can decide to lower `1 add` to become `increment` at the ISA-level. These sorts of small changes to the instructions should happen in the backend and not the assembly until they are finalized.

## Emulator

To test the correctness of written assembly using unit tests, we need to create an emulator. It does not need to run the actual ISA, just the assembly files, since it is only being used to test the consistency of the assembly based on the architecture spec.

## Benchmark assembly

To test various backends, several benchmarks will be created. Each of these benchmarks should tell us something about the performance of an actual real-world program.

- Basic N-body (from benchmark game)
- Fannkuch-redux (from benchmark game)
- k-nucleotide (from benchmark game)
- CSV parsing
