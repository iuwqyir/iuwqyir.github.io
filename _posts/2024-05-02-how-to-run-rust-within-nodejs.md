---
title: How to run Rust code within NodeJS
date: 2024-05-02 16:00:00 +0300
categories: [Web3, Smart Contracts]
tags: [nodejs, rust]
---

Sometimes you just feel that need for speed…  

When we're talking about code performance, Rust is blazing fast compared to NodeJS. At Blocktorch, we were compiling EVM smart contracts on the fly and experienced really bad performance - some contracts took around 1 minute to compile. Then we rewrote the compilation part in Rust and got it down to _6 seconds_. No biggie, just a breezy 10x performance boost.  

So if you're working on optimizing a core part of your product, but your services are in NodeJS, it's not the end of the world. Turns out, you can pretty easily give a great performance boost to your service by running part of it in Rust.

---

_This is the second part of a 2-part blog post. If you want to check out how we got the Rust code we'll be working with, [check out part 1]({% post_url 2024-05-01-how-to-compile-smart-contracts-fast %})._

## How did we do it?
We used Napi-RS, which is a great library. It's easy to use and actively maintained. Let's actually run the smart contract compilation code we wrote in Rust, in NodeJS.

First, let's install the Napi CLI tool
```bash
npm install -g @napi-rs/cli
```

Then, let's create a project. You can do this in an existing NodeJS project or create a new one and publish it as a package. Just make sure you're using node version >=18.12.0.

```bash
napi new
```

Go through the prompts, give it a name, path, choose targets and choose whether to enable GH actions or not (for this example, you don't need it).

You should see a new directory that you named with boilerplate files inside.

## Adding Rust code
Let's copy the code from the previous post and make some necessary changes.

Open `src/lib.rs` and remove the boilerplate `sum` function. Instead copy all the Rust code we wrote previously. If you didn't follow the first part, the code is available [here](https://github.com/iuwqyir/evm-smart-contract-compiler/tree/compile_contract_in_rust){:target="_blank"}.

We need to change the `main` function though. Let's give it a name `compile_contract`, make it public and change to procedural macro from `tokio::main` to `napi`.

```rust
#[napi]
pub async fn compile_contract() -> napi::Result<String> {
  let contract_info: ContractInfo = fetch_contract_source_code("0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984").await.expect("Failed to fetch contract code");
  let compiler_input: CompilerInput = get_compiler_input(&contract_info);
  let compiler_version: String = extract_compiler_version(&contract_info);

  let solc: Solc = Solc::find_or_install_svm_version(&compiler_version).unwrap();
  let compiler_output: CompilerOutput = solc.compile_exact(&compiler_input).expect("Failed to compile contract");
  let compiler_output_as_string = serde_json::to_string(&compiler_output).expect("Failed to parse result to string");
  Ok(compiler_output_as_string)
}
```
{: file="rust-evm-compiler/rust-compiler/src/lib.rs" }

As you can see, we also changed the return type to `String` and we are creating a JSON string from the `compiler_output` and returning it.

## Handling dependencies
You'll see a lot of undefined reference errors, meaning that we are missing dependencies. Open the `Cargo.toml` file in the napi project and add all the dependencies that we had previously. Keep everything that is already there.

```toml
[dependencies]
# Default enable napi4 feature, see https://nodejs.org/api/n-api.html#node-api-version-matrix
napi = { version = "2.12.2", default-features = false, features = ["napi4", "tokio_rt"] }
napi-derive = "2.12.2"
foundry-compilers = { version = "0.3.13", features = ["svm-solc"] }
tokio = { version = "1.36.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0.114"
anyhow = "1.0.81"
reqwest = { version = "0.11.26", features = ["json"] }
```
{: file="rust-evm-compiler/rust-compiler/Cargo.toml" }

We also need to add `tokio_rt` as a feature to the napi dependency so it works with tokio.

## Running the code
Let's build the napi project by going into the directory (whatever you named it) and running

```bash
npm run build
```

It will take some time and build the project for you and now you have a NodeJS executable!

Let's test it out by creating a script. Move to the parent directory and create an `index.js` file.

```Javascript
const { compileContract } = require('./rust-compiler');

(async () => {
  const result = await compileContract();
  console.log(JSON.parse(result));
})()
```
{: file="rust-evm-compiler/index.js" }

I named the Napi project `rust-compiler`, if you named it something else, change the import path.

Now, all that's left to do is to run it.

```bash
node index.js
```

And you're done! You will see the compilation output printed to the console. And it's blazing fast!

---

You learned how to write any code in Rust and run it in NodeJS. This is a great skill to speed up your services, so use it wisely.  

This code is available on Github.

All the code is available on my [Github](https://github.com/iuwqyir/evm-smart-contract-compiler){:target="_blank"}.