---
author: "tianyi"
date: "2024-09-14"
labels: ["zink", "evm"]
description: "Do we really need constructor?"
title: "Constructor JIT in Zink?"
---

The constructor implementation of zink was following solidity at the beginning till I found that I actually 
missed the tests of it, and the result is, my implementation does not work at all... So I have to back to
fix the constructor in zink to continue the ERC20 implementation.

While doing this dirty work, I realized that the constructor implementation in solidity is actually overkilled 
for zink since it is a bit too verbose for adapting corner cases.

## Constructor Layout in Solidity

According to [Layout of Call Data][sol-calldata], solidity compiler needs to process `codecopy`, `mload`, etc.
for loading parameters from the end of the bytecode and passing them to the constructor function, for example:

```yul
// @program: set storage in constructor and get it in the deployed contract.

// init code
//
// 1. Copy parameters to memory
// 2. Load parameter to stack
// 3. store parameter in storage
PUSH1, 0x01, PUSH1, 0x17, PUSH1, 0x1f, CODECOPY, PUSH0, MLOAD, PUSH0, SSTORE

// return logic - calculate the position of runtime bytecode and return it
PUSH1, 0x02, PUSH1, 0x15, PUSH0, CODECOPY, PUSH1, 0x02, PUSH0, RETURN

// runtime bytecode - Load the parameter set in constructor from storage
PUSH0, MLOAD

// parameters - 0x42
PUSH1, 0x42
```

The instructions could also be like below if we compile the init code just in time 

```yul
// init code - just push the parameter
PUSH1, 0x42, PUSH0, SSTORE

// return logic - same
PUSH1, 0x02, PUSH1, 0x0d, PUSH0, CODECOPY, PUSH1, 0x02, PUSH0, RETURN

// runtime bytecode - same
PUSH0, MLOAD

// parameters - none
```

But if so, why solidity takes so much for loading constructor parameters instead of compiles them on creation? If 
I'm not mistaken, they may just want to adapt the case that deploying contracts based on on-chain data, block number, 
timestamp, etc. or the state of deployed contracts.

## No Constructor but `Constructor` in Zink

Considering about the constructor is only used for presetting storage most of the time, pro-macro `constructor` in 
zink is finally removed in [#229][#229], and `Constructor` is introduced instead for wrapping runtime bytecode on 
demand, as a code snippet:

```rust
use zinkc::Constructor;

fn to_bytecode(runtime_bytecode: &[u8]) -> Result<Vec<u8>> { 
  let mut constructor = Constructor::default();
  let bytecode = constructor
    .storage(&[(b"storage_key", b"storage_value")].collect())?
    .finished(runtime_bytecode);
}
```

There will be no constructor but `Constructor` in zink... 


## Future Plans

I'm now pushing the ERC20 implementation straightforwardly since it has been the next milestone since last year... I 
have to admit that I failed this dreaming project in the past days of 2024, 3 months, let's see what we'll archive in 2024.


[#229]: https://github.com/zink-lang/zink/pull/229
[sol-calldata]: https://docs.soliditylang.org/en/latest/internals/layout_in_calldata.html#layout-of-call-data
