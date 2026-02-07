---
name: multiversx-project-architecture
description: Production-grade project structure patterns for MultiversX smart contracts. Use when starting a new contract project, refactoring an existing one, or building multi-contract systems with shared code.
---

# MultiversX Project Architecture

Production-tested folder structures and module composition patterns for MultiversX contracts.

## Single Contract Structure

For contracts up to ~1000 lines of business logic:

```
my-contract/
├── Cargo.toml
├── src/
│   ├── lib.rs              # #[contract] trait — ONLY trait composition + init/upgrade
│   ├── storage.rs           # All #[storage_mapper] definitions
│   ├── views.rs             # All #[view] endpoints
│   ├── config.rs            # Admin configuration endpoints
│   ├── events.rs            # #[event] definitions
│   ├── validation.rs        # Input validation helpers
│   ├── errors.rs            # Static error constants
│   └── helpers.rs           # Pure business logic functions
├── meta/
│   ├── Cargo.toml
│   └── src/main.rs
├── wasm/
│   ├── Cargo.toml
│   └── src/lib.rs
└── tests/
    └── integration_test.rs
```

## Multi-Contract Workspace Structure

For protocols with multiple interacting contracts:

```
my-protocol/
├── Cargo.toml               # [workspace] members
├── common/                   # Shared crate across all contracts
│   ├── Cargo.toml
│   ├── constants/
│   │   ├── Cargo.toml
│   │   └── src/lib.rs       # Protocol-wide constants
│   ├── errors/
│   │   ├── Cargo.toml
│   │   └── src/lib.rs       # Shared error constants
│   ├── structs/
│   │   ├── Cargo.toml
│   │   └── src/lib.rs       # Shared data types
│   ├── math/
│   │   ├── Cargo.toml
│   │   └── src/math.rs      # Shared math module trait
│   └── events/
│       ├── Cargo.toml
│       └── src/lib.rs       # Shared event definitions
├── contract-a/               # First contract
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs
│   │   ├── storage/
│   │   │   └── mod.rs        # Local + proxy storage
│   │   ├── cache/
│   │   │   └── mod.rs        # Drop-based cache
│   │   ├── views.rs
│   │   ├── config.rs
│   │   ├── events.rs
│   │   ├── validation.rs
│   │   └── helpers/
│   │       └── mod.rs
│   ├── meta/
│   ├── wasm/
│   └── tests/
├── contract-b/
│   └── ...                   # Same structure
└── proxy-definitions/        # Optional: shared proxy traits
    ├── Cargo.toml
    └── src/lib.rs
```

## lib.rs Pattern: Trait Composition Only

The main contract file should ONLY compose modules — no business logic:

```rust
#![no_std]

multiversx_sc::imports!();

pub mod cache;
pub mod config;
pub mod events;
pub mod helpers;
pub mod storage;
pub mod validation;
pub mod views;

#[multiversx_sc::contract]
pub trait MyContract:
    storage::StorageModule
    + config::ConfigModule
    + views::ViewsModule
    + events::EventsModule
    + validation::ValidationModule
    + helpers::HelpersModule
    + common_math::SharedMathModule
{
    #[init]
    fn init(&self, /* params */) {
        // Only initialization logic
    }

    #[upgrade]
    fn upgrade(&self) {
        // Only upgrade migration logic
    }

    #[endpoint]
    fn main_operation(&self) {
        // Delegate to helpers/validation
        self.validate_payment(&payment);
        let mut cache = cache::StorageCache::new(self);
        self.process_operation(&mut cache, &payment);
        // cache drops and commits
    }
}
```

## errors.rs Pattern

Use `pub static` byte string references for gas-efficient error messages:

```rust
pub static ERROR_NOT_ACTIVE: &[u8] = b"Contract is not active";
pub static ERROR_INVALID_AMOUNT: &[u8] = b"Invalid amount";
pub static ERROR_UNAUTHORIZED: &[u8] = b"Unauthorized";
pub static ERROR_NOT_SUPPORTED: &[u8] = b"Not supported";
pub static ERROR_INSUFFICIENT_BALANCE: &[u8] = b"Insufficient balance";
pub static ERROR_ZERO_AMOUNT: &[u8] = b"Amount must be greater than zero";
```

Usage:
```rust
require!(amount > 0u64, ERROR_ZERO_AMOUNT);
```

This is more gas-efficient than inline string literals because the compiler deduplicates static references.

## events.rs Pattern

Define events as a separate module trait:

```rust
#[multiversx_sc::module]
pub trait EventsModule {
    #[event("deposit")]
    fn deposit_event(
        &self,
        #[indexed] caller: &ManagedAddress,
        #[indexed] token: &TokenIdentifier,
        amount: &BigUint,
    );

    #[event("withdraw")]
    fn withdraw_event(
        &self,
        #[indexed] caller: &ManagedAddress,
        #[indexed] token: &TokenIdentifier,
        amount: &BigUint,
    );
}
```

## validation.rs Pattern

Centralize all input validation:

```rust
#[multiversx_sc::module]
pub trait ValidationModule: crate::storage::StorageModule {
    fn validate_payment(&self, payment: &EgldOrEsdtTokenPayment<Self::Api>) {
        self.require_token_supported(&payment.token_identifier);
        self.require_amount_positive(&payment.amount);
    }

    fn require_token_supported(&self, token: &EgldOrEsdtTokenIdentifier) {
        require!(self.supported_tokens().contains(token), ERROR_NOT_SUPPORTED);
    }

    fn require_amount_positive(&self, amount: &BigUint) {
        require!(amount > &BigUint::zero(), ERROR_ZERO_AMOUNT);
    }
}
```

## When to Create a Common Workspace Crate

| Signal | Action |
|---|---|
| Same struct used in 2+ contracts | Move to `common/structs/` |
| Same math function in 2+ contracts | Move to `common/math/` |
| Same error messages across contracts | Move to `common/errors/` |
| Same event definitions | Move to `common/events/` |
| Protocol constants (precision, limits) | Move to `common/constants/` |

## Workspace Cargo.toml

```toml
[workspace]
members = [
    "contract-a",
    "contract-a/meta",
    "contract-b",
    "contract-b/meta",
    "common/structs",
    "common/math",
    "common/errors",
    "common/constants",
    "common/events",
]
```

Common crate Cargo.toml:
```toml
[package]
name = "common-structs"
version = "0.0.0"
edition = "2024"

[dependencies.multiversx-sc]
version = "0.64.0"
```

## Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Contract crate | kebab-case | `liquidity-pool` |
| Module files | snake_case | `storage.rs`, `helpers.rs` |
| Storage keys | camelCase | `"totalSupply"`, `"feeRate"` |
| Error constants | SCREAMING_SNAKE | `ERROR_INVALID_AMOUNT` |
| Module traits | PascalCase | `StorageModule`, `ValidationModule` |
| Endpoint names | snake_case | `fn deposit_tokens(&self)` |
| View names | camelCase (ABI) | `#[view(getBalance)]` |
