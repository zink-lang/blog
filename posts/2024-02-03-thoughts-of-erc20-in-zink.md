---
author: "cydonia"
date: "2024-02-03"
labels: ["cydonia", "rust"]
description: "Thoughts about the storage implementation in zink."
title: "Thoughts of ERC-20 in Zink"
---

Have been stuck in the development of zink for around 2 months, feeling my life is worthless though these days since 
I haven't done anything that I'm feeling proud of.

The ERC20 implementation in zink is a big picture which leads zink to a real programming language of EVM when it gets
completed, I'm too scared to see that something **solidity** can do but **zink** can not that I could not even push one 
more commit in zink these days.

## Splitting Problems

Another night without sleeping, tired of video games again finally, get up and check the code base of zink, I found that
I have already cleaned the code structure of the code generation module last time which makes it easier to catch up my 
previous work this time.

What if I just start my work on ERC20, splitting the problems out into issues like always, the layout of [ERC20][erc20] is 
pretty simple, ERCs in zink will be implemented in rust traits without doubts. 

```solidity
abstract contract ERC20 is Context, IERC20, IERC20Metadata, IERC20Errors {
    mapping(address account => uint256) private _balances;
    
    // ...
    
    /**
     * @dev Sets the values for {name} and {symbol}.
     *
     * All two of these values are immutable: they can only be set once during
     * construction.
     */
    constructor(string memory name_, string memory symbol_) {
        _name = name_;
        _symbol = symbol_;
    }
}
```


## Storage

Mapping storage is missing in zink for now bcz I haven't got a perfect idea for passing bytes from rust to evm yet.

```solidity
mapping(address account => uint256) private _balances;
```

But for doing it in hard way, I have already got an idea from the design of my old friend [ink][ink], for the [Mapping][ink-mapping]
storage implementation in **ink**, they are mapping pairs into storage slots, mb it doesn't look clever but it seems that this is 
the best solution anyway.

```rust
pub struct Mapping<K, V: Packed, KeyType: StorageKey = AutoKey> {
    #[allow(clippy::type_complexity)]
    _marker: PhantomData<fn() -> (K, V, KeyType)>,
}
```

I was actually struggling with how to use rust's **BTreeMap** in zink before, using **BTreeMap** is friendly to
rust developers because we are familiar with it, however there are two big problems in it:

- The methods of **BTreeMap** or any **Iterator** requires allocation from memory which will embed tons of code
to support it.
- Even if we are okay with the tons of allocation code, store a encoded **BtreeMap** in evm takes a lot anyway,
bcz we need to handle encoding/decoding stuffs which burns gas a lot.

In conclusion, due to the two points above, it is very unfortunate that using Rust's **BtreeMap** in zink is not 
a proper solution.

### Store `bytes` in zink!

Zink currently only support `i32` in storage, bcz I haven't got a solution passing bytes elegantly yet, but for moving forward
to the goal of ERC20, I compromise to implement it anyway, the current solution requires an `Abi` trait _(I hate the naming `Abi`, 
because it can describe too many interfaces in zink xd)_.

```rust
pub trait Abi {
    fn write(&self) -> Result<()>;
}

// ... impl Abi for number types
impl Abi for i32 {}

// ... impl Abi for bytes 
impl Abi for [u8; 0] {}
impl Abi for [u8; 1] {}
// ...
impl Abi for [u8; 32] {}
```

`sstore` requires two stack inputs as key and value, for simplifying the problem, for the types which implements `Abi`, we only need to 
make sure that they can be transformed into bytes under the length 32 at the moment.

```yul
PUSH0 // key
PUSH0 // value
SSTORE
```

Thus we need generate matched FFI for these stuffs as well, which is closed to the assembly implementation,

```rust
mod ffi { 
  fn store_key_i32_value_i32(key: i32, value: i32) {}
  fn store_key_i32_value_bytes(key: i32, ptr: i32, length: i32) {}
  // ... everything
}
```

I love writing macros in rust but not the generated code like above looks really ugly, but seems I have to do it now anyway.


### Store maps in zink

I don't like the naming `mapping` because it is too long, so if I can choose, I'll use `map` because it is shorter, typescript is using `Map` 
or `Record` as well, so I don't understand why solidity is using the keyword `mapping`, mb because they have token `=>` in the storage declaration,
and they want to express **ing**.

So for `Map` in zink, it will follow the well-designed `Mapping` in `ink`, provided `Key`, `Value`, and `Prefix`, the problem will be concatenating
the storage key in zink.

And the solution is using macro, again:

```rust
mod ffi {
  fn store_map_key_i32_value_i32(prefix: i32, key: i32, value: i32) {}
  fn store_map_key_i32_value_bytes(prefix: i32, key: i32, ptr: i32, length: i32) {}
}
```
However, we can fix `i32` as prefix this time, because the storage keys of a contract may never reach the max limit of `i32`. 

## Interfaces

After solving the storage problem, the next one is the design of `interface` in zink, like mentioned above, we can use trait without doubts, but the 
problem is that we need to export the methods provided by traits to WASM as well, hmm, this problem leads us to a derive macro,

```rust

#[derive(Erc20)]
struct MyContract {}

// Which generates

#[no_mangle]
extern "C" fn total_supply() {}
```

Looks weird, but it works, should zink ask users to use `MyContract` to define the namespace is a problem as well, maybe we can provide different 
solutions for this first, but as the experience from apple, we'd better only give users the best solution finally at this kind of points.

## Errors and Events

Errors and events are about to be refactored as well since now we have a solution for passing bytes in zink now, even it is ugly, but it is best
solution for now ))

[ink]: https://github.com/paritytech/ink
[ink-mapping]: https://docs.rs/ink/latest/ink/storage/struct.Mapping.html
[erc20]: https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol
