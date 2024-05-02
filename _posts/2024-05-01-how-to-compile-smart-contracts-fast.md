---
title: How to compile EVM smart contracts fast in NodeJS… but it's actually Rust?
date: 2024-05-01 16:00:00 +0300
categories: [Web3, Smart Contracts]
tags: [evm, nodejs, rust, compilation]
---

Smart contract compilation is not something you might come across often, but when you do, you want it to be fast. Especially if it's part of a request a user makes.  

This is exactly what we stumbled upon when building an EVM transaction debugger at Blocktorch. We were compiling smart contracts in NodeJS, but it took a long time. For a pretty standard Account Abstraction transaction, it took around 1 minute. No one will wait that long and if you're using AWS ApiGateway for example, you might not even be able to due to its 30 second timeout limit.

## Why use NodeJS?
Yes, it's not known as the most performant language, but it has its pros, especially for a start-up. It's lightweight, easy, low-effort, quick to develop and excellent for developing web applications (both front-end and back-end). So 99% of the time in a fast paced environment, it does the job.  

## But why compile smart contracts at all?
Reasons for compiling smart contracts are similar to any other compiled language. It will convert to human-readable code into machine-readable code executed in the EVM. Every smart contract is actually compiled before deploying on-chain and it is only the result of the compilation (the bytecode, plus some metadata) that will be uploaded to the blockchain.  

Here's a few reasons you'd need to compile smart contracts:
- Deploying a smart contract
- Debugging an EVM transaction
- Getting the ABI of a contract
- Smart contract verification

Since we were working on a debugger for EVM transactions, we needed to compile the contracts in the transaction.  

## Compiling a smart contract in Rust
For the sake of simplicity, let's say we start with an address. If you start with a transaction, it is pretty trivial to detect which addresses were called as part of the transaction flow.  

Now, what do we need to compile the EVM smart contract that corresponds to this address?  
- Language the contract was written in
- Sources: Files included with the smart contract and their source code
- Settings: Compiler settings
- A Solidity compiler

If you are the creator of the smart contract, you probably have all of these at hand. Since for this example we are not, we need to rely on a block explorer (like Etherscan) or Sourcify. In this example, we'll use Etherscan.  

Let's go through the whole process step-by-step.  

### 1. Install Rust

Since we are compiling in Rust, we also need to install it.  
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
Most up to date instructions will be on the [Rust homepage](https://www.rust-lang.org/tools/install){:target="_blank"}.

### 2. Install Solidity version manager
Instead of choosing a specific version of a Solidity compiler, a Solidity compiler version manager is a wonderful tool. It will let us install different solidity compilers and switch between them seamlessly.

```bash
cargo install svm-rs
```

### 3. Create a new Rust project
Let's create a new project, naming it `rust-evm-compiler` (or whatever you want)

```bash
cargo new rust-evm-compiler
```

You'll see a new folder created with the project name and a bunch of files. What you'll care about is the `src/main.rs` file, which will hold the main logic of your Rust script and the `Cargo.toml` file, where you can define the dependencies of the project.

If you run `cargo run` in the terminal you should see "_Hello, world!_" printed out to the console.

### 4. Fetch smart contract data from Etherscan
Let's define a new function above the existing main function.  
```rust
use serde::{Serialize,Deserialize};
use reqwest::Client;

#[derive(Serialize, Deserialize, Debug)]
struct EtherscanResponse {
    status: String,
    message: String,
    result: Vec<ContractInfo>
}

#[derive(Serialize, Deserialize, Debug, Clone)]
struct ContractInfo {
  SourceCode: String,
  ABI: String,
  ContractName: String,
  FileName: Option<String>,
  CompilerVersion: String,
  OptimizationUsed: String,
  Runs: String,
  ConstructorArguments: String,
  EVMVersion: String,
  Library: String,
  LicenseType: String,
  Proxy: String,
  Implementation: String,
  SwarmSource: String
}

async fn fetch_contract_source_code(address: &str) -> Result<ContractInfo, anyhow::Error> {
  let client: Client = Client::new();
  let response: EtherscanResponse = client.get("https://api.etherscan.io/api")
    .query(&[("module", "contract"), ("action", "getsourcecode"), ("address", address)])
    .send()
    .await?
    .json()
    .await?;

  match response.result.get(0) {
      Some(contract_info) => Ok(contract_info.clone()),
      None => Err(anyhow::Error::msg("Unable to fetch contract from Etherscan")),
  }
}
```
{: file="rust-evm-compiler/src/main.rs" }

Okay, looks like a lot of code, but it's actually pretty simple. `fetch_contract_source_code` takes a smart contract address and will fetch data about that smart contract from Etherscan.

We define `EtherscanResponse` and `ContractInfo` structs so we can work with the Etherscan response as there are no predefined structs for it.

You should see a few errors in the code though, since we are using external dependencies, but did not declare them. Let's fix that.

Add the following dependencies to `Cargo.toml`
```toml
# ...other stuff...

[dependencies]
tokio = { version = "1.36.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
anyhow = "1.0.81"
reqwest = { version = "0.11.26", features = ["json"] }
```
{: file="rust-evm-compiler/Cargo.toml" }

Now let's try it out by changing the main function a little bit. We'll be using the [Uniswap token](https://etherscan.io/address/0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984){:target="_blank"} address as an example.

```rust
#[tokio::main]
async fn main() -> Result<(), anyhow::Error> {
  let contract_info: ContractInfo = fetch_contract_source_code("0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984").await?;
  print!("{:?}", contract_info);
  Ok(())
}
```
{: file="rust-evm-compiler/src/main.rs" }

Running `cargo run` should now print a bunch of contract data to the console. Success!

### 6. Parsing Etherscan data to compiler input
We will use the `foundry-compilers` library to compile the smart contract and it expects a certain input. Now we'll convert the data the get from Etherscan to this input format.

Let's add a new function called `get_compiler_input`.

```rust
use foundry_compilers::artifacts::{CompilerInput, Settings, Sources, Source};
use std::collections::BTreeMap;
use std::str::FromStr;
{% raw %}
fn get_compiler_input(contract_info: &ContractInfo) -> CompilerInput {
  if contract_info.SourceCode.starts_with("{{") {{% endraw %}
    return serde_json::from_str(&contract_info.SourceCode[1..contract_info.SourceCode.len()-1]).expect("Failed to parse SourceCode JSON string")
  } else {
    let file_name: String = contract_info.FileName.clone().unwrap_or_else(|| {
      format!("{}.sol", contract_info.ContractName)
    });
    let mut sources: Sources = BTreeMap::new();
    sources.insert(file_name.into(), Source { content: contract_info.SourceCode.clone().into() });
    let mut settings: Settings = Settings::default();
    if contract_info.EVMVersion.to_lowercase() == "default" {
      settings.evm_version = None;
    } else {
      settings.evm_version = Some(EvmVersion::from_str(&contract_info.EVMVersion).unwrap());
    }
    CompilerInput {
      language: "Solidity".to_string(),
      sources,
      settings
    }
  }
}
```
{: file="rust-evm-compiler/src/main.rs" }

Etherscan can return a JSON string for some contracts and this is what we check in the first `if` clause. If that's the case, the data is already in the correct format and we just need to parse it into an object. Otherwise, we will build `CompilerInput` ourselves.

If we're building the input ourselves, we need to build the path of the contract from its name and set the correct evm version.

We'll also need to add 2 new dependencies
```toml
[dependencies]
# ...previous dependencies...
foundry-compilers = { version = "0.3.13", features = ["svm-solc"] }
serde_json = "1.0.114"
```
{: file="rust-evm-compiler/Cargo.toml" }

Now let's update the main function to use this new function.

```rust
#[tokio::main]
async fn main() -> Result<(), anyhow::Error> {
  let contract_info: ContractInfo = fetch_contract_source_code("0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984").await?;
  let compiler_input: CompilerInput = get_compiler_input(&contract_info);
  print!("{:?}", compiler_input);
  Ok(())
}
```
{: file="rust-evm-compiler/src/main.rs" }

After running `cargo run` now, you should see the compiler input printed out in the console with `language`, `sources` and `settings`.

One side-note - if you want to use the compiler output in debugging, you should also change the `output_selection` in the settings of the input and make sure to include `ast` always.

### 7. Compiling the contract
Okay, we're almost there. We have the compiler input, now we need extract the compiler version we need to use.

```rust
fn extract_compiler_version(contract_info: &ContractInfo) -> String {
  let compiler_ver_without_commit = contract_info.CompilerVersion.split('+').next().unwrap_or("");
  let parsed_compiler_version = if compiler_ver_without_commit.starts_with('v') {
      &compiler_ver_without_commit[1..]
  } else {
      compiler_ver_without_commit
  };
  parsed_compiler_version.to_string()
}
```
{: file="rust-evm-compiler/src/main.rs" }

This will remove the v from the beginning and the commit hash at the end of the version.

Now to put it all together:
```rust
use foundry_compilers::{CompilerOutput, Solc};

#[tokio::main]
async fn main() -> Result<(), anyhow::Error> {
  let contract_info: ContractInfo = fetch_contract_source_code("0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984").await?;
  let compiler_input: CompilerInput = get_compiler_input(&contract_info);
  let compiler_version: String = extract_compiler_version(&contract_info);

  let solc: Solc = Solc::find_or_install_svm_version(&compiler_version).unwrap();
  let compiler_output: CompilerOutput = solc.compile_exact(&compiler_input).expect("Failed to compile contract");
  print!("{:?}", compiler_output);
  Ok(())
}
```
{: file="rust-evm-compiler/src/main.rs" }

We'll use `find_or_install_svm_version` to handle all possible Solidity compiler versions and then print out the result.

**Success!** We can now compile any verified smart contract in Rust in just a few seconds!

---

Awesome, in this article you just learned how to compile a smart contract in Rust. [In the next part]({% post_url 2024-05-02-how-to-run-rust-within-nodejs %}), we'll look at how to run this Rust code in NodeJS.

All the code is available on my [Github](https://github.com/iuwqyir/evm-smart-contract-compiler/tree/compile_contract_in_rust){:target="_blank"}.