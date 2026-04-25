# ☥ sigil

[![Crates.io](https://img.shields.io/crates/v/sigil.svg)](https://crates.io/crates/sigil)
[![Docs.rs](https://docs.rs/sigil/badge.svg)](https://docs.rs/sigil)
[![CI](https://github.com/agilov/sigil/actions/workflows/ci.yml/badge.svg)](https://github.com/agilov/sigil/actions)

Fast, unforgeable, roughly sortable 64-bit ID generation in Rust.

Standard Snowflake IDs are guessable, allowing attackers to scrape your data or forge IDs. UUIDs are unforgeable, but take up 128 bits and destroy database index locality. 

`sigil` provides the best of both worlds: a 64-bit integer that sorts chronologically, but includes a peppered checksum that instantly rejects ~98.4% of forged or guessed IDs without a database lookup.

## Performance
- **Generation:** ~5–40 ns (depending on OS clock syscalls).
- **Validation:** ~1 ns (nearly 1 billion validations/sec per core).
- **Memory:** Zero allocations. Pure arithmetic.

## Bit Layout

```text
 63                    20 19           6 5           0
 |------- time (44b) ------|-- rand (14b) -|- check (6b) -|
```

- **44 bits:** Millisecond timestamp (valid for ~557 years).
- **14 bits:** Xorshift random tail (16,384 slots per ms tick).
- **6 bits:** Checksum seeded with a secret pepper.

## Usage

Add `sigil` to your `Cargo.toml`:
```toml
[dependencies]
sigil = "0.1.0"
```

Basic generation and validation:
```rust
fn main() {
    // Generate a new 64-bit ID
    let id = sigil::generate();
    
    // Validate an incoming ID instantly
    if sigil::is_valid(id) {
        println!("Valid ID: {}", id);
    }
}
```

### Securing the Checksum
At application startup, inject a secret pepper. Without this pepper, an attacker cannot generate a valid checksum, meaning you can drop bad traffic before it ever hits your database.

```rust
fn main() {
    // Set this once at startup using a securely generated random u64
    sigil::set_pepper(0x8F2A_9B3C_4D5E_6F70);
    
    let secure_id = sigil::generate();
}
```

### Configuration
You can adjust the balance between throughput and security at runtime. Every bit moved from the checksum to the random tail doubles your throughput but halves your fake-ID rejection rate.

```rust

fn main() {
    // Move to 12 bits of randomness (4,096 IDs/ms) and 8 bits of checksum (99.6% rejection)
    sigil::configure(0x8F2A_9B3C_4D5E_6F70, 12, 8);
}
```
