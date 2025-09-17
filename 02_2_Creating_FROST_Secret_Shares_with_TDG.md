# 2.2: Creating FROST Secret Shares with TDG

The ZF FROST Tools allow for the easy creation of FROST secret shares
using either the Trusted Dealer Generation (TDG) or the Distributed
Key Generation (DKG) methodology. Of these, TDG is the quickest and
simplest.

## Creating TDG Shares

Trusted Dealer Generation (TDG) can be conducted using the
`trusted-dealer` binary, which should now be in your `.cargo/bin`
directory.

> :book: ***What is TDG?*** In Trusted Dealer Generation, a single
server shards a secret (which it may create itself or import) and
distributes the secret shares to signers.

> :book: ***How Does ZF FROST Tools Support TDG?*** The
`trusted-dealer` binary in the ZF FROST tools can import a key or
create its own. It will then generate `n+1` files: one file for each
signer, containing an identifier, secret share, and commitment (which
verifies the secret share); and one global file, containing the
verifying key and each of the verifying shares. It is up to the user
to actually distribute the shares.

The `trusted-dealer` binary may be run with the following flags:

| Flag | Description | Default | Options |
| -----|---------|---------|
| -C | Ciphersuite | ed25519 | redpallas |
| --cli | Interactive | <no> |
| -k | Key Filenames| key-package-{}.json |
| --key | Secret for Splitting | <random> |
| -n | Maximum Signers | 3 | 2+ & t+ |
| -P | Public Key Filename | public-key-package.json |
| -t | Threshold Signers | 2 | 2+ |

`trusted-dealer` wwill usually be run with the `-t` and `-n`
arguments. The following uses them to create a default 2-of-3 secret share:

```
% trusted-dealer -t 2 -n 3 Generating 3 shares with threshold 2...
Public key package written to public-key-package.json Key package for
participant
0100000000000000000000000000000000000000000000000000000000000000
written to key-package-1.json Key package for participant
0200000000000000000000000000000000000000000000000000000000000000
written to key-package-2.json Key package for participant
0300000000000000000000000000000000000000000000000000000000000000
written to key-package-3.json ``` It generates four files: ``` % ls
key-package-1.json key-package-3.json key-package-2.json
public-key-package.json
```

All of the key material is stored as files:
```
% ls
key-package-1.json	key-package-3.json
key-package-2.json	public-key-package.json
```

The `public-key-package.json` has all the verifying information:
```
{
  "header": {
    "version": 0,
    "ciphersuite": "FROST-ED25519-SHA512-v1"
  },
  "verifying_shares": {
    "0100000000000000000000000000000000000000000000000000000000000000": "1d593a814d4063bd2b7085934f3a19f4cf58865d5b3e52bb1aa7943b68cd6de0",
    "0200000000000000000000000000000000000000000000000000000000000000": "1a9da435c3b7ddc34742173be9def22eacbd502cb330b40e96adf4ed7a054486",
    "0300000000000000000000000000000000000000000000000000000000000000": "c4b841960ad23cc1acfb658eabbe471ff06686e50e9ff4c1ce8107cacb1e235a"
  },
  "verifying_key": "f9bee7f2faf1b2ff0117914fe16aead54b91702feb8848d4150bed3cb518e0e7"
}
```
The `key-package-#.json` files each contain the secret share for one user:
```
{
  "header": {
    "version": 0,
    "ciphersuite": "FROST-ED25519-SHA512-v1"
  },
  "identifier": "0100000000000000000000000000000000000000000000000000000000000000",
  "signing_share": "a07560ec07da786e3746d97775d3cb54fe0b4bb8480a0771f8da7957a8866508",
  "commitment": [
    "f9bee7f2faf1b2ff0117914fe16aead54b91702feb8848d4150bed3cb518e0e7",
    "7fdbe463164a4bf9fd4afaa2b43e5d4c0e9ed3e26a1a4c35a12eb3a124c0ead0"
  ]
}
```
Each user should be sent their own `key-package` file as well as the
`public-key-package`.
```
% secure-send alice@crypto-example.com < key-package-1.json public-key-package.json
% secure-send bob@crypto-example.com < key-package-2.json public-key-package.json
% secure-send carol@crypto-example.com < key-package-3.json public-key-package.json
```
Afterward, all of the files should be securely deleted from the
Trusted Dealer: though the shares were generated in a single place,
they should never be available in a single place again, to take full
advantage of the security of FROST signing.

## Summary: Creating FROST Secret Shares with TDG

Trusted Dealer Generation is the simplest but also least secure way to
use FROST. It allows quick and easy generation of secret shares, but
the Trusted Dealer must actually be trusted and it must be secure, as
it represents a Single Point of Failure (SPOF) until the shares have
been distributed and securely removed.

(Nonetheless, it's a great way to get a first hint at how FROST works.)

## What's Next

Continue onward with "Signing with FROST" by [§2.2: Creating FROST
Secret Shares with
TDG](02_2_Creating_FROST_Secret_Shares_with_TDG.md).



