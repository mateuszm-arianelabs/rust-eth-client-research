## web3 rs
https://crates.io/crates/web3

Ethereum JSON-RPC multi-transport client. Rust implementation of the Web3.js library. This is the oldest package, allowing contract calls, batch requests, and sending transactions. It supports the async runtime `tokio` and transport via HTTP, IPC, and WebSockets.

If you want to deploy a contract, you need ready bin and ABI files, as the library doesn't provide any solution for that. In the documentation, they recommend using `solc` outside of Rust.

The library provides contract types, transaction, transaction receipt, block, work, and syncing but doesn't generate types and methods from ABI. When you want to call a contract method, you need to pass the method name as a string and parameters in a tuple.

___Library status___
As they typed on the crate page, this package is barely maintained, and they are looking for an active maintainer. They recommend using `ethers-rs` for new projects, but for now, it's not recommended either, as will be discussed later.

__Docs__
In my opinion, the documentation is weak, but at least the library provides some examples of the most popular use cases.

__Examples:__

Basic interaction with chain:
```rust
#[tokio::main]  
async fn main() -> web3::Result<()> {  
    let transport = web3::transports::Http::new("...")?;  
    let web3 = web3::Web3::new(transport);  
  
    println!("Calling accounts.");  
    let mut accounts = web3.eth().accounts().await?;  
    println!("Accounts: {:?}", accounts);  
    accounts.push("...".parse().unwrap());  
  
    println!("Calling balance.");  
    for account in accounts {  
        let balance = web3.eth().balance(account, None).await?;  
        println!("Balance of {:?}: {}", account, balance);  
    }  
    Ok(())  
}
```

Deploy and interact with contract:
```rust
use hex_literal::hex;
use web3::{
    contract::{Contract, Options},
    types::U256,
};

#[tokio::main]
async fn main() -> web3::contract::Result<()> {
    let _ = env_logger::try_init();
    let http = web3::transports::Http::new("...")?;
    let web3 = web3::Web3::new(http);

    let my_account = hex!("...").into();
    // Get the contract bytecode for instance from Solidity compiler
    let bytecode = include_str!("...").trim_end();
    // Deploying a contract
    let contract = Contract::deploy(web3.eth(), include_bytes!("..."))?
        .confirmations(0)
        .options(Options::with(|opt| {
            opt.value = Some(5.into());
            opt.gas_price = Some(5.into());
            opt.gas = Some(3_000_000.into());
        }))
        .execute(
            bytecode,
            (U256::from(1_000_000_u64), "My Token".to_owned(), 3u64, "MT".to_owned()),
            my_account,
        )
        .await?;

    let result = contract.query("balanceOf", (my_account,), None, Options::default(), None);
    // Make sure to specify the expected return type, to prevent ambiguous compiler
    // errors about `Detokenize` missing for `()`.
    let balance_of: U256 = result.await?;
    assert_eq!(balance_of, 1_000_000.into());

    // Accessing existing contract
    let contract_address = contract.address();
    let contract = Contract::from_json(
        web3.eth(),
        contract_address,
        include_bytes!("..."),
    )?;

    let result = contract.query("balanceOf", (my_account,), None, Options::default(), None);
    let balance_of: U256 = result.await?;
    assert_eq!(balance_of, 1_000_000.into());

    Ok(())
}
```

## ethers rs
https://crates.io/crates/ethers
Complete Ethereum & Celo library and wallet implementation in Rust. This crate is created to work with Ethereum and Celo chains, with rich features listed below.

__Features__
- Ethereum JSON-RPC Client
- Interacting with and deploying smart contracts
- Type-safe smart contract bindings code generation
- Querying past events
- Event monitoring as `Stream`s
- ENS as a first-class citizen
- Celo support
- Polygon support
- Avalanche support
- Optimism support
- Websockets / `eth_subscribe`
- Hardware wallet support
- Parity APIs (`tracing`, `parity_blockWithReceipts`)
- Geth TxPool API

Unlike `web3-rs`, `ethers` provides tools for compilation and abigen. The library looks more mature and more developer-friendly.

__Library status__
They write on their GitHub: "We are deprecating ethers-rs for Alloy."

__Docs__
`Ethers rs` has better documentation represented by the book at https://www.gakonst.com/ethers-rs/. However, it is incomplete and provides examples only for some parts.

__Examples:__

deployment and interact with contract
```rust
use ethers::{
    contract::abigen,
    core::utils::Anvil,
    middleware::SignerMiddleware,
    providers::{Http, Provider},
    signers::{LocalWallet, Signer},
};
use eyre::Result;
use std::{convert::TryFrom, sync::Arc, time::Duration};

// Generate the type-safe contract bindings by providing the json artifact
// *Note*: this requires a `bytecode` and `abi` object in the `greeter.json` artifact:
// `{"abi": [..], "bin": "..."}` , `{"abi": [..], "bytecode": {"object": "..."}}` or
// `{"abi": [..], "bytecode": "..."}` this will embedd the bytecode in a variable `GREETER_BYTECODE`
abigen!(Greeter, "...",);

#[tokio::main]
async fn main() -> Result<()> {
    // 1. compile the contract (note this requires that you are inside the `examples` directory) and
    // launch anvil
    let anvil = Anvil::new().spawn();

    // 2. instantiate our wallet
    let wallet: LocalWallet = anvil.keys()[0].clone().into();

    // 3. connect to the network
    let provider =
        Provider::<Http>::try_from(anvil.endpoint())?.interval(Duration::from_millis(10u64));

    // 4. instantiate the client with the wallet
    let client = Arc::new(SignerMiddleware::new(provider, wallet.with_chain_id(anvil.chain_id())));

    // 5. deploy contract
    let greeter_contract =
        Greeter::deploy(client, "Hello World!".to_string()).unwrap().send().await.unwrap();

    // 6. call contract function
    let greeting = greeter_contract.greet().call().await.unwrap();
    assert_eq!("Hello World!", greeting);

    Ok(())
}
```

## alloy-core
https://crates.io/crates/alloy-core

Alloy implements high-performance, well-tested, and documented libraries for interacting with Ethereum and other EVM-based chains. It is a rewrite of [`ethers-rs`](https://github.com/gakonst/ethers-rs) from the ground up, with exciting new features, high performance, and excellent [documentation](https://docs.rs/alloy).

Alloy is a collection of several crates around the Ethereum ecosystem. For our use case, we'll use alloy-core. It's the youngest library on the list, but that doesn't mean it's the worst. On the contrary, as a rewrite of ethers-rs, many known problems in the previous libraries won't appear.

__Library status__  
It's a relatively new library but highly developed and will probably become the most popular choice soon.

__Docs__  
[Alloy documentation](https://alloy.rs/) contains rich examples of most use cases, with comments provided to explain everything line by line.

__Example__
contract deployment and interaction.
```rust
//! Example of deploying a contract at runtime from Solidity bytecode to Anvil and interacting with
//! it.

use alloy::{
    hex,
    network::{EthereumWallet, ReceiptResponse, TransactionBuilder},
    node_bindings::Anvil,
    primitives::U256,
    providers::{Provider, ProviderBuilder},
    rpc::types::TransactionRequest,
    signers::local::PrivateKeySigner,
    sol,
};
use eyre::Result;

// If you have the bytecode known at build time, use the `deploy_from_contract` example.
// This method benefits from using bytecode at runtime, e.g., from newly deployed contracts, to
// analyze the behavior.
sol! {
    #[allow(missing_docs)]
    #[sol(rpc)]
    contract Counter {
        uint256 public number;

        function setNumber(uint256 newNumber) public {
            number = newNumber;
        }

        function increment() public {
            number++;
        }
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    // Spin up a local Anvil node.
    // Ensure `anvil` is available in $PATH.
    let anvil = Anvil::new().try_spawn()?;

    // Set up signer from the first default Anvil account (Alice).
    let signer: PrivateKeySigner = anvil.keys()[0].clone().into();
    let wallet = EthereumWallet::from(signer);

    println!("Anvil running at `{}`", anvil.endpoint());

    // Create a provider with the wallet.
    let rpc_url = anvil.endpoint().parse()?;
    let provider =
        ProviderBuilder::new().with_recommended_fillers().wallet(wallet).on_http(rpc_url);

    // Deploy the `Counter` contract from bytecode at runtime.
    let bytecode = hex::decode(
        // solc v0.8.26; solc Counter.sol --via-ir --optimize --bin
        "..."
    )?;
    let tx = TransactionRequest::default().with_deploy_code(bytecode);

    // Deploy the contract.
    let receipt = provider.send_transaction(tx).await?.get_receipt().await?;

    let contract_address = receipt.contract_address().expect("Failed to get contract address");
    let contract = Counter::new(contract_address, &provider);
    println!("Deployed contract at address: {}", contract.address());

    // Set number
    let builder = contract.setNumber(U256::from(42));
    let tx_hash = builder.send().await?.watch().await?;

    println!("Set number to 42: {tx_hash}");

    // Increment the number to 43.
    let builder = contract.increment();
    let tx_hash = builder.send().await?.watch().await?;

    println!("Incremented number: {tx_hash}");

    // Retrieve the number, which should be 43.
    let builder = contract.number();
    let number = builder.call().await?.number.to_string();

    println!("Retrieved number: {number}");

    Ok(())
}
```

## Summary

`Web3-rs` is the oldest library that provides a more low-level implementation. It probably won't be used in new projects due to better alternatives.

`Ethers rs` is the most mature library, offering a full toolchain. It is currently the most popular choice, but it lacks good documentation and there are no plans for future maintenance because of Alloy, as it is currently deprecated.

`Alloy-core` is a rewrite of `Ethers rs`. It provides good documentation, rich examples, and a full toolchain that enhances the developer experience. It is the only library with a bright future and a long-term maintenance plan. The downside is that it is not as popular as `Ethers`, but we are in a situation where Alloy is recommended in the main `Ethers` repository, so this will soon change.