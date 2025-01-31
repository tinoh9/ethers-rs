Your ethers application interacts with the blockchain through a [`Provider`](ethers_providers::Provider) abstraction. [`Provider`](ethers_providers::Provider) is a special type of [`Middleware`](ethers_providers::Middleware) that can be composed with others to obtain a layered architecture. This approach promotes "Open Closed Principle", "Single Responsibility" and composable patterns. The building process happens in a wrapping fashion, and starts from a [`Provider`](ethers_providers::Provider) being the first element in the stack. This process continues having new middlewares being pushed on top of a layered data structure.


## Available Middleware

- [`Signer`](./signer/struct.SignerMiddleware.html): Signs transactions locally, with a private key or a hardware wallet.
- [`Nonce Manager`](./nonce_manager/struct.NonceManagerMiddleware.html): Manages nonces locally. Allows to sign multiple consecutive transactions without waiting for them to hit the mempool.
- [`Gas Escalator`](./gas_escalator/struct.GasEscalatorMiddleware.html): Bumps transactions gas price in the background to avoid getting them stuck in the memory pool. A [`GasEscalatorMiddleware`](crate::gas_escalator::GasEscalatorMiddleware) supports different escalation strategies (see [GasEscalator](crate::gas_escalator::GasEscalator)) and bump frequencies (see [Frequency](crate::gas_escalator::Frequency)).
- [`Gas Oracle`](./gas_oracle/struct.GasOracleMiddleware.html): Allows getting
  your gas price estimates from places other than `eth_gasPrice`, including REST based gas stations (i.e. Etherscan, ETH Gas Station etc.).
- [`Transformer`](./transformer/trait.Transformer.html): Allows intercepting and
  transforming a transaction to be broadcasted via a proxy wallet, e.g.
  [`DSProxy`](./transformer/struct.DsProxy.html).

## Stacking middlewares using a builder

Each [`Middleware`](ethers_providers::Middleware) implements the trait [MiddlewareBuilder](crate::MiddlewareBuilder). This trait helps a developer to compose a custom [`Middleware`](ethers_providers::Middleware) stack.

The following example shows how to build a composed [`Middleware`](ethers_providers::Middleware) starting from a [`Provider`](ethers_providers::Provider):

```rust
use ethers_providers::{Middleware, Provider, Http};
use std::sync::Arc;
use std::convert::TryFrom;
use ethers_signers::{LocalWallet, Signer};
use ethers_middleware::{*,gas_oracle::*};

fn builder_example() {
    let key = "fdb33e2105f08abe41a8ee3b758726a31abdd57b7a443f470f23efce853af169";
    let signer = key.parse::<LocalWallet>().unwrap();
    let address = signer.address();
    let gas_oracle = EthGasStation::new(None);

    let provider = Provider::<Http>::try_from("http://localhost:8545")
        .unwrap()
        .gas_oracle(gas_oracle)
        .with_signer(signer)
        .nonce_manager(address); // Outermost layer
}
```

The [wrap_into](crate::MiddlewareBuilder::wrap_into) function can be used to wrap [`Middleware`](ethers_providers::Middleware) layers explicitly. This is useful when pushing [`Middleware`](ethers_providers::Middleware)s not directly handled by the builder interface.

```rust
use ethers_providers::{Middleware, Provider, Http};
use std::sync::Arc;
use std::convert::TryFrom;
use ethers_signers::{LocalWallet, Signer};
use ethers_middleware::{*,gas_escalator::*,gas_oracle::*};

fn builder_example_wrap_into() {
    let key = "fdb33e2105f08abe41a8ee3b758726a31abdd57b7a443f470f23efce853af169";
    let signer = key.parse::<LocalWallet>().unwrap();
    let address = signer.address();
    let escalator = GeometricGasPrice::new(1.125, 60_u64, None::<u64>);

    let provider = Provider::<Http>::try_from("http://localhost:8545")
        .unwrap()
        .wrap_into(|p| GasEscalatorMiddleware::new(p, escalator, Frequency::PerBlock))
        .wrap_into(|p| SignerMiddleware::new(p, signer))
        .wrap_into(|p| GasOracleMiddleware::new(p, EthGasStation::new(None)))
        .wrap_into(|p| NonceManagerMiddleware::new(p, address)); // Outermost layer
}
```


## Stacking middlewares manually
A [`Middleware`](ethers_providers::Middleware) stack can be also constructed manually. This is achieved by explicitly wrapping layers.

```rust no_run
use ethers_providers::{Provider, Http};
use ethers_signers::{LocalWallet, Signer};
use ethers_middleware::{
    gas_escalator::{GasEscalatorMiddleware, GeometricGasPrice, Frequency},
    gas_oracle::{GasOracleMiddleware, EthGasStation, GasCategory},
    signer::SignerMiddleware,
    nonce_manager::NonceManagerMiddleware,
};
use ethers_core::rand;
use std::convert::TryFrom;

// Start the stack
let provider = Provider::<Http>::try_from("http://localhost:8545").unwrap();

// Escalate gas prices
let escalator = GeometricGasPrice::new(1.125, 60u64, None::<u64>);
let provider = GasEscalatorMiddleware::new(provider, escalator, Frequency::PerBlock);

// Sign transactions with a private key
let signer = LocalWallet::new(&mut rand::thread_rng());
let address = signer.address();
let provider = SignerMiddleware::new(provider, signer);

// Use EthGasStation as the gas oracle
let gas_oracle = EthGasStation::new(None);
let provider = GasOracleMiddleware::new(provider, gas_oracle);

// Manage nonces locally
let provider = NonceManagerMiddleware::new(provider, address);

// ... do something with the provider
```
