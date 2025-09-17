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
    "0100000000000000000000000000000000000000000000000000000000000000": "b496e15c0cd0438440b9890348155eb8d8616e2a3ec8815100aa0c18822ba924",
    "0200000000000000000000000000000000000000000000000000000000000000": "68befcb80ac14775e602245a209f45c7a41dc740aeb066eb43a9f1e636ab3482",
    "0300000000000000000000000000000000000000000000000000000000000000": "d764598583ebc633689d7819d56aa449bd280a32cc1f63c586d5036da24454a1"
  },
  "verifying_key": "8f9ee5e7f2642ed8b7c4eb1c7342ecbf69e7284b07e18434e577ad580a1412ed"
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
  "signing_share": "e958bbfac93847ee2b444a62541f75e37316d988728382d1f923c0d95c86de0d",
  "commitment": [
    "8f9ee5e7f2642ed8b7c4eb1c7342ecbf69e7284b07e18434e577ad580a1412ed",
    "e18234fd12212514e53e4b20c34f104bc5ef84fff3a8ac5673133dd3d766b329"
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



