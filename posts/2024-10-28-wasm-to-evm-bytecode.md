---
author: "tianyi"
date: "2024-10-28"
labels: ["zink", "evm"]
description: "How zink translate WASM to EVM bytecode"
title: "Dark magics in Zink"
---

For those who may not know, Rust is a systems programming language known for its memory safety features and performance.
EVM (Ethereum Virtual Machine) is the runtime environment for Ethereum smart contracts, responsible for executing bytecode
written in languages like Solidity.

So, what makes it possible to write Rust EVM smart contracts with Zink? Let's dive into the "dark magics" of Zink!

## 1. What is Zink?

[Zink](https://github.com/zink-lang/zink) is a rustic smart contract language for EVM, it uses WebAssembly (WASM) as an
intermediate format before generating EVM bytecode.

### 1.1. Why using WASM ?

We're choosing WASM as the intermediate representation with the following reasons:

1. WASM can be compiled from rust source code directly
2. WASM has simple instruction set, 172 instructions in the MVP implementation (127 of them are numeric instructions).
3. WASM is backed by Mozilla and the Rust Team and widely used by the leading companies in the world, it has powerful toolchains.
4. WASM is designed for stack machine, it's perfectly matched with EVM.

### 1.2. Challenges translating WASM to EVM bytecode

Even though WASM is designed for stack based machine, it can not be executed by EVM directly for several reasons:

1. **Instruction Set**: EVM has its own instruction set, which is optimized for executing smart contracts. WASM, on the other
hand, has a different instruction set that is designed for general-purpose computation.
2. **Memory Model**: EVM uses a memory model that is specific to Ethereum's blockchain architecture. WASM, being a stack-based 
virtual machine, has its own memory model that is not compatible with EVM's memory layout.
3. **Function Call Conventions**: EVM has its own function call conventions, which are designed for executing Solidity smart 
contracts. WASM, being a stack-based VM, uses different function call conventions that would need to be adapted for execution 
on the EVM.

## 2. Instruction Mappings

As mentioned above, WASM and EVM bytecode have different ISA (see [Webassembly Opcodes][wasm-set] and
[EVM Opcode Opcodes Interactive Reference][evm-set]), what we need to do first is actually mapping the 
shared opcodes from WASM to EVM directly, once all of the WASM opcodes can be mapped to EVM opcodes safely
and correctly, any of WASM can be translated to EVM bytecode.

### 2.1. Control Flow ( 0x00 - 0x1f )

| WASM Opcode Hex | WASM         | EVM          | Description                                |
|-----------------|--------------|--------------|--------------------------------------------|
| 0x00            | unreachable  | INVALID      | -                                          |
| 0x01            | nop          | JUMPDEST     | JUMPDEST in EVM performs nop as well       |
| 0x02 - 0x1f     | control flow | JUMP & JUMPI | need structured instructions based on JUMP |

As we can see in the control flow instruction set, some of the opcodes can be mapped to EVM opcodes in per opcode level, like
`unreachable` and `nop`, others are required **structured instructions**, see [select][select] as an example.

### 2.2. Memory & Locals ( 0x20 - 0x44 )

These opcodes can not be simply mapped since WASM has different memory model with EVM, they will be translated to different
instructions in different cases, see `3.2.` for more details.

1. **local** will be translated to memory or calldata operations like `MLOAD` or `CALLDATALOAD`.
2. **const** will be translated to `PUSH(n) + {value}`
3. **global** fetches bytes from data section and then do stack pushing like **const**

Some opcodes are banned since they are not adapatable, for example `global.get`, `memory.x`, once zink compiler reaches these
opcodes, panic occurs, however, zink provides different approaches for replacing the logic which can generate these opcodes,
see `3` for more details.


### 2.3. Numeric operations ( 0x44 - 0xbf )

WASM supports `i32` and `i64` as number types, for EVM, it's `u256`, and since EVM only has `u256` as number (bigger than `i64`),
all WASM numbers and their operations can be perfectly mapped to EVM in per opcode level or chained instructions.


## 3. Memory Allocator

Due to EVM and WASM have different memory models, memory operations in WASM could not be reflected to EVM, and that's why zink
bans all memory operations in rust, however, it's not that serious since reducing memory operations in EVM is also a key point
of saving gas, see our move in the followings.

### 3.1. Iteration

Iteration is banned since the iter in the core library of rust generates memory operations, instead, we need to provide customized
iterator for arraylike types used in zink ourselves for generating EVM compatible bytecode.

```rust
// This generates iter method from core library, will panic in zink compiler.
[1, 2, 3].iter();

// instead, zink will provide `Arraylike` type for the iteration feature
trait ArrayLike<T> {
  /// Memory pointer generated by the compiler
  const fn memory_slot() -> u32;
  
  // get the next ptr of this iteration
  fn next_ptr();
  
  // reset the ptr of this iteration
  fn reset_ptr();
}

impl<T> ArrayLike<T> {
  /// we can not use `Iter` or `IntoIter` trait from the core library, they will generate unexpected bytecode.
  pub fn next(&self) {
    // These block will be assembly EVM bytecode doing the following things
    //
    // 1. get the memory pointer of this array
    // 2. get the next value of the array
  }
}
```

With our asm instructions, the `ArrayLike` will generate similar code for array operations like solidity does.


### 3.2. Function Arguments

WASM has instructions of locals while EVM doesn't, in zink we do the similar stuffs like `3.1.` mentions, fill the logic with assembly
code, but since there are `calldata`, `memory` and `code` concepts in EVM, we're dispatching locals dynamically.

Zink filters locals as function arguments and local variables, arguments in function selector will use calldata and all of the others
will have their own memory slots, for example:

```rust
// (argument) slot0: x 
// (argument) slot1: y
// (local variable) slot2: z
pub fn main(x: u32, y: u32) -> u32 {
    let z = x + y;
    return z
}
```

This contract will have 3 locals in WASM ( due to the optimizations in zink, there will be no locals in this contract in the real cases,
instead, `x` and `y` will be extracted from calldata and z will be returned directly without assigning a new local ), each of them will
have their own memory slot.

`code` will only be used when we need to share large instructions to different functions without using a function.


### 3.3 Foreign Types

As we mentioned before, WASM only has `i32` and `i64` as number types, some solidity types are hard to be translated for example, `address`
and `u256`, which they are actually `[u8; 20]` and `[u8; 32]` in rust.

1. Due to `address` and `u256` have the same representation in EVM bytecode ( 32 bytes ), we can merge them just into `u256`.
2. In the bytecode level, `i32` and `i64` are `u256` as well.

This is how EVM works tbh, all types are actually `u256` in the bytecode level, so the problem we need to solve is that representing `u256`
from `rust` and `WASM` without introducing redundant bytecode.

Since our goal is compiling rust to EVM bytecode, we have to handle most generated WASM bytecode, but not all, for `u256` type, we can use
translate it with assembly code directly! 

```
#[cfg(feature = "evm")]
pub struct U256(
// The i32 here is a placeholder to identify this type in WASM
i32
);

#[cfg(feature = "evm")]
impl U256 {
  // addition of two u256
  pub fn add(&self, rhs: U256) -> U256 {
    zink::asm::add(self, rhs)
  }
}
```

And this is the wrap, you may curious how to make a real `i32` safe from WASM to EVM, the answer is that we need to append chained instructions
after operations of `i32` to confirm there's no overflows, using `U256` as the default number type will be recommend in zink as well!


## 4. Function Calls

Instruction like `address` from EVM does not exist in WASM, however, WASM's host functions are perfectly matched with these:

```
#[link(wasm_import_module = "evm")]
#[allow(improper_ctypes)]
extern "C" {
  pub fn address() -> Address;
}

let addr = address();
```

When zink compiler reaches the host function `evm::address`, it simply emits the EVM opcode of `address` to the contract bytecode.

[select]: https://docs.zink-lang.org/compiler/control-flow.html#select
[wasm-set]: https://pengowray.github.io/wasm-ops/
[evm-set]: https://www.evm.codes/
