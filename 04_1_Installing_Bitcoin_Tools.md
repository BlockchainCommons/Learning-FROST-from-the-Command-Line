# 4.1: Installing Bitcoin Tools

Though signing Bitcoin transactions with a FROST group is now
possible, none of the command line tools do quite what's
required. Doing so instead requires using a branch of the ZF FROST
Tools and a few hand constructed tools, along with the excellent
Bitcoin command-line tool, BDK, the Bitcoin Development Kit.

## Equipment Inventory

This section will take you through the installation of the following
tools, which together will align you to sign and send a Bitcoin
transaction:

* The Bitcoin Core client & server
* The Bitcoin Development Kit
* Our version of the ZF FROST Tools with Secp256K1-TR Ciphersuite Support
   * ... Using the Taproot Tweak Branch
* The `sighash-helper` Tool   
* The `psbt-sig-attach` Tool

Obviously, this is a complex process, though this tutorial will take
you right through it. In walking through this process, you'll see not
only how everything works, but also why it's very important to make
this type of procedure much easier, to aid in the accessibility and so
the usage of FROST.

## Installing Bitcoin Core

You will need Bitcoin Core to simulate the creation and signing of
transactions.

We would usually suggest [Gordian
Server](https://github.com/BlockchainCommons/GordianServer-macOS) as a
best solution for running a Bitcoin node, but it is limited to Macs,
and it offers more features than is really necessary for the simple
regtest example here.

So instead we suggest downloading the appropriate version of Bitcoin
Core for your OS:

[https://bitcoin.org/en/download](https://bitcoin.org/en/download)

Some pacakage managers may make this install even easier. For example,
the following will install Bitcoin Core on your Mac if you use the
Homebrew package installer:

```
% brew install bitcoin
```

This should install both `bitcoind` (the server) and `bitcoin-cli` (a
command-line client).

## Installing bdk-cli

Though Bitcoin has its own command-line app (`bitcoin-cli`), this
example also uses `bdk-cli`, the [Bitcoin Dev Kit
CLI](https://github.com/bitcoindevkit/bdk-cli), which provides access
to some more complex functionality.

This is a `bdk-cli` crate, making this an easy install.
```
% cargo install bdk-cli --features rpc
```

Note that the `rpc` feature must be included to allow connection to
the `bitcoind` server that you just installed.

> :book: ***What is RPC?*** RPC is the "Remote Procedure Call", used
to communicate between processes on different systems (or even on the
same system). It includes an authorization protocol, which allows the
remote server to authenticate a user connecting to it, using a user
name and password..

## Installing the Blockchain Commons Version of FROST

Blockchain Commons has patched the ZF FROST tools to offer the
improvements required to sign Bitcoin transactions. This comes in two
parts: one adds secp256k1-tr as a ciphersuite; and the other
supplements that by incorporating the Taproot methodology for tweaking
keys.

> :book: ***What is secp256k1?*** Bitcoin uses elliptic curve
cryptography (ECC) for its core cryptographic functions, such as
creating a public key from a private key. Any ECC calculations are
done with a specific elliptic curve. For Bitcoin, the curve used for
ECC calculations is secp256k1.

> :book: ***What is Taproot?*** Taproot was an upgrade to Bitcoin
introduced in November 2021. It allows for the use of Schnorr
signatures (including FROST signatures) with a new P2TR type of
transaction. The P2TR transaction allows public keys and scripts to be
secretly combined into a single `scriptPubKey` that locks a Bitcoin
transaction. This is done with a "tweak".

> :book: ***So what is secp256k1-tr?*** It's a Schnorr signature using
the secp256k1 curve that supports Taproot (and so FROST).

You can install the updated version of the ZF FROST tools that
includes both of these elements as follows:
```
% git clone https://github.com/BlockchainCommons/zcash-frost-tools
% cd zcash-frost-tools 
% git checkout taproot-tweak
% cargo install --path frost-client && cargo install --path frostd
% cd ..
```

This will entirely replace the version of the ZF FROST tools that you
installed in [§2.1: Installing the FROST
Tools](02_1_Installing_FROST_Tools.md). You should messages like the
following indicating this during the installation:
```
Replacing /Users/ShannonA/.cargo/bin/coordinator
Replacing /Users/ShannonA/.cargo/bin/dkg
Replacing /Users/ShannonA/.cargo/bin/frost-client
Replacing /Users/ShannonA/.cargo/bin/participant
Replacing /Users/ShannonA/.cargo/bin/trusted-dealer

...

Replacing /Users/ShannonA/.cargo/bin/frostd
Replaced package `frostd v0.1.0 (/Users/ShannonA/Documents/GitHub/FROST/frost-tools/frostd)` with `frostd v0.1.0 (/Users/ShannonA/Documents/GitHub/Blockchain-Commons/zcash-frost-tools/frostd)` (executable `frostd`)
```
The version numbers will be the same, but looking at your copies of the apps should indicate an update via new timestamps:
```
% ls -lagh `which frost-client`
-rwxr-xr-x  1 staff    10M Nov  4 11:13 /Users/ShannonA/.cargo/bin/frost-client
% ls -lagh `which frostd`
-rwxr-xr-x  1 staff   7.0M Nov  4 11:14 /Users/ShannonA/.cargo/bin/frostd
```

## Creating the `sighash-helper` Helper Tool   

Even `bdk-cli` and `bitcoin-cli` don't have everything required to
sign a FROST transaction!

To start with, there's no way to extract a signature hash from a
transaction in a format suitable for FROST signing. The
`sighash-helper` tool is a simple Rust program that does so.

First, create a Rust project:
```
% cargo new sighash-helper --bin
Creating binary (application) `sighash-helper` package
```
This will create two files: `Cargo.toml`, which details the program
and its dependencies; and `src/main.rs`, which contains the actual
code for the program.

You then need to add the following to the `[dependencies]` section of
the `Cargo.toml`. If the section is currently empty (likely the case
for a default file), you can do the following:
```
% cd sighash-helper
% cat >> Cargo.toml
bdk = { version = "0.30.2", default-features = false, features = ["std"] }
```
You'll need to hit ^D at the end to close out the `cat`.

Afterward your file should look something like:
```
[package]
name = "sighash-helper"
version = "0.1.0"
edition = "2024"

[dependencies]
bdk = { version = "0.30.2", default-features = false, features = ["std"] }
```
You should then input the following into `src/main.rs`:
```rust
use std::io::{self, Read};
use std::str::FromStr;

use bdk::bitcoin::psbt::PartiallySignedTransaction as Psbt;
use bdk::bitcoin::sighash::{Prevouts, SighashCache, TapSighashType};

fn main() {
    // 1. read the base‑64 PSBT from stdin
    let mut b64 = String::new();
    io::stdin().read_to_string(&mut b64).unwrap();
    let psbt = Psbt::from_str(b64.trim()).expect("valid base64 PSBT");

    // 2. assume exactly one input (enforced by the demo)
    let tx = &psbt.unsigned_tx;

    // 3. clone the single witness_utxo so it lives long enough
    let txout = psbt.inputs[0]
        .witness_utxo
        .as_ref()
        .expect("witness_utxo present")
        .clone();
    let prevouts_arr = [txout];                 // concrete array on the stack
    let prevouts = Prevouts::All(&prevouts_arr);

    // 4. compute Taproot key‑path sighash
    let mut cache = SighashCache::new(tx);
    let msg = cache
        .taproot_key_spend_signature_hash(0, &prevouts, TapSighashType::Default)
        .expect("sighash");

    // 5. print the 32‑byte message as hex
    println!("{:064x}", msg);
}
```
You can enter this with `cat > src/main.rs` or use your favorite editor.

You're now ready to install your `sighash-helper`
```
% cargo install --path .
% cd ..
```

## Creating the `psbt-sig-attach` Helper Tool

There's also no way to add an arbitrary signature into a PSBT. This
requires the creation of another helper tool, `psbt-sig-attach`.

```
% cargo new psbt-sig-attach --bin
Creating binary (application) `psbt-sig-attach` package
```

Again, add dependencies:
```
% cd psbt-sig-attach
% cat >> Cargo.toml
bdk    = { version = "0.30.2", default-features = false, features = ["std"] }
base64 = "0.22.1"
hex    = "0.4"
bitcoin = "0.30.2"
```
Afterward, `Cargo.toml` should look like this:
```
[package]
name = "psbt-sig-attach"
version = "0.1.0"
edition = "2024"

[dependencies]
bdk    = { version = "0.30.2", default-features = false, features = ["std"] }
base64 = "0.22.1"
hex    = "0.4"
bitcoin = "0.30.2"
```
Then, replace `src/main.rs` with the following:
```rust
use std::io::{self, Read};
use std::str::FromStr;

use bitcoin::{
    psbt::PartiallySignedTransaction as Psbt,
    secp256k1::schnorr::Signature as SchnorrSig,
    sighash::TapSighashType,
    taproot::Signature as TaprootSignature,
};
use hex::FromHex;

fn main() {
    // r||s hex comes as first CLI arg
    let sig_hex = std::env::args()
        .nth(1)
        .expect("usage: psbt-sig-attach <64-byte r||s hex>");
    let sig_vec = Vec::<u8>::from_hex(sig_hex.trim()).expect("hex sig");
    assert_eq!(sig_vec.len(), 64, "need 64-byte Schnorr sig");

    // wrap in TaprootSignature with default sighash flag
    let tap_sig = TaprootSignature {
        sig: SchnorrSig::from_slice(&sig_vec).expect("schnorr sig"),
        hash_ty: TapSighashType::Default,
    };

    // read base-64 PSBT from stdin
    let mut b64 = String::new();
    io::stdin().read_to_string(&mut b64).unwrap();
    let mut psbt = Psbt::from_str(b64.trim()).expect("base64 PSBT");

    // insert signature in input 0
    psbt.inputs[0].tap_key_sig = Some(tap_sig);

    // write updated PSBT as base-64
    println!("{}", psbt.to_string());
}
```
Finally, install and leave the project behind:
```
% cargo install --path .
% cd ..
```

## Checking Your Inventory

You can check your inventory on most systems with the `which`
command. If anything is missing, you'll need to either consult the
appropriate section, above, for how to install, or perhaps just make
sure your $PATH includes where the install happened.

```
% which bitcoind
/opt/homebrew/bin/bitcoind
% which bitcoin-cli
/opt/homebrew/bin/bitcoin-cli
% which bdk-cli
/Users/ShannonA/.cargo/bin/bdk-cli
% which frost-client
/Users/ShannonA/.cargo/bin/frost-client
% which frostd
/Users/ShannonA/.cargo/bin/frostd
5 which sighash-helper
/Users/ShannonA/.cargo/bin/sighash-helper
% which psbt-sig-attach
/Users/ShannonA/.cargo/bin/psbt-sig-attach
```

## Summary Bitcoin Tools

To get ready for FROST signing of a Bitcoin transaction, you need to
install:

* **bitcoind:** The server that will run your `regtest` environment.
* **bitcoin-cli:** Bitcoin's app, which will provide basic functionality.
* **bdk-cli:** The Bitcoin Dev Kit app, which provides advanced functionality.
* **ZF FROST Tools:** An updated version of `frostd` and `frost-client`.
* **Helper Tools:** Two small apps, `sighash-helper` and `psbt-sig-attach`, which are required to extract info from a PSBT and then import into it.

## What's Next

You'll put this to use in [§4.2: Signing PSBTs with
FROST](04_2_Signing_PSBTs_with_FROST.md).
