# WebAssembly Constant-Time Extension

This specification describes the _constant-time_ extension to WebAssembly, which makes it possible to write and execute secure, side-channel resistant code in WebAssembly.

## Motivation

The goal of this proposal is to make it possible for applications to ensure their cryptographic code, when compiled to WebAssembly, is secure against side-channel attacks. This proposal: (1) complements the existing work on WASI-Crypto by allowing applications to implement custom cryptographic primitives and protocols directly in WebAssembly; and, (2) addresses a key limitation when writing cryptographic code in WebAssembly today: _"WebAssembly doesn't provide any guarantees against resistance to side-channel attacks"_ (see [this WASI-Crypto issue](https://github.com/WebAssembly/wasi-crypto/issues/21)).

## Overview

CT-Wasm extends WebAssembly with new _secret types_, distinguishing secret values (and memories) from normal _public_ values (and memories), and instructions on secrets. To ensure secrets are not leaked via timing side channels, these instructions should be implemented in constant-time, i.e., their execution time should not depend on the secret operants. This proposal defines a set of instructions which can be translated to efficient constant-time native code on commodity hardware.

> **Note**: Certain instructions are inherently unsafe. For example, branching on secrets or accessing memory at secret indexes is unsafe. Other instructions, like floating-point operations can be implemented to run in constant-time (see, e.g., [this paper](https://cseweb.ucsd.edu/~dstefan/pubs/andrysco:2018:towards.pdf)) but are not efficient. We conservatively restrict the instructions to the subset that is constant-time on common hardware, efficiently implementable, and sufficient for writing typical applications. 
 
### Type extensions

CT-Wasm extends Wasm's types to make secret explicit.

- Two new numeric types: `s32` and `s64` representing secret integer values:

  ```
  numtype := i32 | i64 | f32 | f64 | s32 | s64
  ```

- Memory type has been expanded to include a flag indicating that it contains secret values:

  ```
  memtype := secret? limits
  ```


#### Why types?

The original CT-Wasm [paper](https://arxiv.org/pdf/1808.01348.pdf) and [implementation](https://github.com/PLSysSec/ct-wasm) introduced these type extensions and instructions. During the Phase 1 community discussion we considered simplifying this specification to only introduce constant-time instructions. While this seems simpler at first, it makes implementations more complex (and gives up on CT-Wasm's strong formal security guarantees). First, compilation passes must ensure to never optimize code based on secret data. And, second, in embedding environments like the browser and Node.js, the garbage collector must not trace secrets on the stack ([since this can be abused](https://bugs.chromium.org/p/chromium/issues/detail?id=1144662)). Both are hard to get right when secrets are implicit---types make secrets explicit. 

### New/extended instructions

- For each numeric operation which can be implemented in constant time on common hardware, there is now a corresponding operation for `s32` and `s64` (e.g. `sNN.add`, `sNN.sub` etc.). Notably, operations like `div` and `rem` are excluded.

- For all memory operations which load or store integers there is now a corresponding operation for secret integers:

  ```
  instr :=  ... | sNN.load | sNN.store | sNN.store8 | ...etc
  ```

  These instructions are only valid on memories whose memtype is `secret`.

- New instructions for explicitly classifying and declassifying integers:

  ```
  iNN.declassify : sNN -> iNN
  sNN.classify   : iNN -> sNN
  ```

Together, these instructions preserve the secrecy of data between explicit classification and declassification, even as values flow in and out of memory. A full list of instructions can be found [here][opcode_table].

> **Note:** Control-flow operations such as `if` are **not** extended to support `s32` and `s64`, because branching on a secret can reveal its value. Similarly loads and stores do **not** allow secrets as indexes (even though secrets can be loaded and stored in the secret-annotated memory).

[opcode_table]: ...
