# 18.4: Accessing Bitcoind with Rust

## Abstract

The objective of this exercise is to access `bitcoind` in `Rust` and send a transaction
on the `Regtest network`. It is helpful to read the chapter [Setting up a Bitcoin regtest](A2_0_Setting_Up_a_Bitcoin_Regtest.md) before but not necessary.

## Preparing the Environment

We'll need `Rust` and `Cargo`. If you don't have them yet, see [instructions on installation](https://doc.rust-lang.org/cargo/getting-started/installation.html).

To set `Bitcoin Regtest` network up and allow communication with our Rust program we
will be using the following `bitcoind` configuration in `bitcoin.conf`

```vim
regtest=1
server=1
rpcuser=bitcoin
rpcpassword=password
[test]
rpcport=18443
```

> Note: Never use a simple password like that when on Bitcoin Mainnet!

## Create a New Project

We create a new project with `cargo new btc_test`:

```vim
gorazd@gorazd-MS-7C37:~/Projects/BlockchainCommons$ cargo new btc_test
     Created binary (application) `btc_test` package
```

We move into the newly created project `btc_test`. We notice a "hello world" example
with the source code in `src/main.rs` and a `Cargo.toml` file. Let's run it with `cargo run`:

```vim
gorazd@gorazd-MS-7C37:~/Projects/BlockchainCommons/btc_test$ cargo run
   Compiling btc_test v0.1.0 (/home/gorazd/Projects/BlockchainCommons/btc_test)
    Finished dev [unoptimized + debuginfo] target(s) in 0.14s
     Running `target/debug/btc_test`
Hello, world!
```

> Note: if you run into error “linker ‘cc’ not found”, you'll have to install a
C compiler. If on Linux, go ahead and install the [development tools](https://www.ostechnix.com/install-development-tools-linux/).


We will use `bitcoincore-rpc` library, therefore we add it to our `Cargo.toml`
under section `dependencies` like so:

```rust
[dependencies]
bitcoincore-rpc = "0.11.0"
```

Let's run our example again. This will install our new library and 
its dependencies.

```vim
gorazd@gorazd-MS-7C37:~/Projects/BlockchainCommons/btc_test$ cargo run
    Updating crates.io index
   ...
   Compiling bitcoin v0.23.0
   Compiling bitcoincore-rpc-json v0.11.0
   Compiling bitcoincore-rpc v0.11.0
   Compiling btc_test v0.1.0 (/home/gorazd/Projects/BlockchainCommons/btc_test)
    Finished dev [unoptimized + debuginfo] target(s) in 23.56s
     Running `target/debug/btc_test`
Hello, world!
```

## Coding

Let us create a new Bitcoin `RPC client` and modify the `main.rs`:

```rust
use bitcoincore_rpc::{Auth, Client};

fn main() {
    let rpc = Client::new(
        "http://localhost:18443".to_string(),
        Auth::UserPass("bitcoin".to_string(), "password".to_string()),
    )
    .unwrap();
}
```

`Cargo run` should successfully compile and run the example with one warning 
`warning: unused variable: rpc`


Now, lets generate our first bitcoin address:

```rust
// Generate a new address
let myaddress = rpc
    .get_new_address(None, Option::Some(json::AddressType::Bech32))
    .unwrap();
println!("my address: {:?}", myaddress);
```

The compiler will tell us to include traits into scope. So lets add them:

```rust
use bitcoincore_rpc::{json, Auth, Client, RpcApi};
```

If our properly configured `bitcoind` is running, executing our example should
result in getting our new address:

```vim
gorazd@gorazd-MS-7C37:~/Projects/BlockchainCommons/btc_test/src$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `/home/gorazd/Projects/BlockchainCommons/btc_test/target/debug/btc_test`
my address: bcrt1q780lygz52h6atxncws0yn5z3d3ls53w8jg6wc3
```

We would like to have some bitcoins to our newly generated address. Since 
we are on the `Regtest` network we can generate them ourselves:

```rust
// Get some bitcoins
let _ = rpc.generate_to_address(101, &myaddress);
```

Next, we set the recipient's address and the amount of 10 BTC which we want to send.

```rust
let recipient = Address::from_str("bcrt1q6rhpng9evdsfnn833a4f4vej0asu6dk5srld6x").unwrap();
println!("recipient: {:?}", recipient);

let amount = Amount::from_btc(10.0).unwrap();
```

This will require to put more traits into scope:

```rust
use bitcoincore_rpc::bitcoin::{Address, Amount};
use std::str::FromStr;
```

Let's can to send it:

```rust
// Send bitcoin to recipient
let txid = rpc
.send_to_address(&recipient, amount, None, None, None, None, None, None)
.unwrap();
println!("txid: {:?}", txid);
```

If the command was successfully run, `bitcoind` has given us a transaction ID:

```vim
gorazd@gorazd-MS-7C37:~/Projects/BlockchainCommons/btc_test/src$ cargo run
   Compiling btc_test v0.1.0 (/home/gorazd/Projects/BlockchainCommons/btc_test)
    Finished dev [unoptimized + debuginfo] target(s) in 0.76s
     Running `/home/gorazd/Projects/BlockchainCommons/btc_test/target/debug/btc_test`
my address: bcrt1qcxn4xvgfsw9mxyxg42aqk2awruxs7c3mxscqmm
recipient: bcrt1q6rhpng9evdsfnn833a4f4vej0asu6dk5srld6x
txid: 5284eab1bf451e24727a6f65d1a70658cba7593e0b47ca565d3ea1f7a615aa57
```

This transaction must be in the `mempool` now. It hasn't reached any 
miners yet because, remember, we are controlling the network.

```rust
// Tx must be in the mempool
assert!(rpc.get_raw_mempool().unwrap().contains(&txid));
```

If we decide to mine a block, we can expect our transaction to be mined 
and so no longer in the `mempool`:

```rust
// Generate a block
let _ = rpc.generate_to_address(1, &myaddress);

// Tx is no longer in the mempool. It must have been mined.
assert!(rpc.get_raw_mempool().unwrap().contains(&txid) == false);
```

Well, let's see for ourselves if our transaction is really in the block:

```rust
// Get the latest block
let block_hash = rpc.get_best_block_hash().unwrap();

// Check if our transaction is in the block
let block = rpc.get_block(&block_hash).unwrap();
for x in block.txdata {
    if x.txid() == txid {
        println!(
            "Transaction {:?} has been mined. \n\nCongratulations!\n",
            txid
        );
    }
}
```

The final output should look like this:

```vim
gorazd@gorazd-MS-7C37:~/Projects/BlockchainCommons/btc_test/src$ cargo run
   Compiling btc_test v0.1.0 (/home/gorazd/Projects/BlockchainCommons/btc_test)
    Finished dev [unoptimized + debuginfo] target(s) in 0.86s
     Running `/home/gorazd/Projects/BlockchainCommons/btc_test/target/debug/btc_test`
my address: bcrt1q46axunsnmcfksqsylyzwyv5as998nesq3hsxxe
recipient: bcrt1q6rhpng9evdsfnn833a4f4vej0asu6dk5srld6x
txid: 570f6e7ac992d614f0b2714d0af2daa8f8628817e62c5f990733d798c9a1aa19
Transaction 570f6e7ac992d614f0b2714d0af2daa8f8628817e62c5f990733d798c9a1aa19 has been mined. 

Congratulations!

```

Great, we have successfully sent a transaction on the `Bitcoin Regtest` network!

## Code Snippet

Here is the complete code snippet:

```rust
use bitcoincore_rpc::bitcoin::{Address, Amount};
use bitcoincore_rpc::{json, Auth, Client, RpcApi};
use std::str::FromStr;

fn main() {
    let rpc = Client::new(
        "http://localhost:18443".to_string(),
        Auth::UserPass("bitcoin".to_string(), "password".to_string()),
    )
    .unwrap();

    // Generate a new address
    let myaddress = rpc
        .get_new_address(None, Option::Some(json::AddressType::Bech32))
        .unwrap();
    println!("my address: {:?}", myaddress);

    // Get some bitcoins
    let _ = rpc.generate_to_address(101, &myaddress);

    let recipient = Address::from_str("bcrt1q6rhpng9evdsfnn833a4f4vej0asu6dk5srld6x").unwrap();
    println!("recipient: {:?}", recipient);

    let amount = Amount::from_btc(10.0).unwrap();

    // Send bitcoin to recipient
    let txid = rpc
        .send_to_address(&recipient, amount, None, None, None, None, None, None)
        .unwrap();
    println!("txid: {:?}", txid);

    // Tx must be in the mempool
    assert!(rpc.get_raw_mempool().unwrap().contains(&txid));

    // Generate a block
    let _ = rpc.generate_to_address(1, &myaddress);

    // Tx is no longer in the mempool. It must have been mined.
    assert!(rpc.get_raw_mempool().unwrap().contains(&txid) == false);

    // Get the latest block
    let block_hash = rpc.get_best_block_hash().unwrap();

    // Check if our transaction is in the block
    let block = rpc.get_block(&block_hash).unwrap();
    for x in block.txdata {
        if x.txid() == txid {
            println!(
                "Transaction {:?} has been mined. \n\nCongratulations!\n",
                txid
            );
        }
    }
}
```
