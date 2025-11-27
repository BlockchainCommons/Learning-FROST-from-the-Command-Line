# 1.3: FROST vs MuSig2

FROST combines Schnorr signatures, various DKG methods, and a specific
signing protocol to create a powerful threshold signing system. This
section offers some more details on the underlying Schnorr signature
system and also compares FROST to the other major Schnorr signing
system, MuSig2.

## The Power of Schnorr

FROST defines a methodology for using private key shares to generate
signing shares that can then be aggregated into a single signature.
The full private key is never reconstructed in the process. The
ability to sign and aggregate in this way comes from the Schnorr
signature algorithm that is the heart of FROST.

> :book: ***What is a Schnorr signature?*** A Schnorr signature is a
digital signature based on Claus Schnorr's signature algorithm. It was
under patent until 2010, which resulted in other algorithms such as
RSA and ECDSA being used, even by those who considered them inferior
to Schnorr signatures. Schnorr signatures are based on the discrete
log problem over finite fields rather than on elliptic curves or prime
numbers, but that (along with the precise math) are details that
aren't necessary for understanding FROST.

> :book: ***What is the security level of a Schnorr signature?***
Traditionally, a Schnorr signature has been understood to require a
4t-bit security level: 128 bits of security require a 512 bit
signature. More recent work has suggested that a 3t-bit security level
may be sufficient.

The biggest advantage of Schnorr signatures over other signature
systems is that they're aggregateable. If you have a signature A and a
signature B, you can add them together to create a new signature AB
that is the same size as A or B. This is particularly notable when
multiple parties are signing multisigs or threshold signatures, where
the signature will verify when enough people have signed it. This
creates some of the advantages that were mentioned in
[§1.1](01_1_Introducing_FROST.md).

* **Small Signatures.** Schnorr Signatures are always the same size, no matter
how many are aggregated into a threshold or multisig.
* **Private Signatures.** When a threshold signature is created using
Schnorr, you can't tell who's actually signed it, just that it's
valid!

The aggregatability of Schnorr allows for the creation of adaptor
signatures where a signature is "tweaked" by adding some secret value
to it. The tweaked signature can be verified, but the secret value
must be revealed to make the signature valid.

It also allows for the creation of blind signatures, where someone can
sign something without knowing its content. (This is obviously
something that must be done with real care, but there are nonethless
use cases for it.)

Beyond the various advantages related to its aggregatability, Schnorr
also has advantages of simplicity (the math is straightforward),
linearity (aggregatability means the signatures simply add and
subtract), and scalability (courtesy of the compact signatures).

> :book: ***How Does Schnorr Integrate with Bitcoin?*** Schnorr
signatures were introduced to Bitcoin with block 709,632, mined on
November 12th, 2021. The details of the integration of Schnorr
signatures with Bitcoin can be found in [BIP
340](https://en.bitcoin.it/wiki/BIP_0340). Taproot and Tapscript,
which support Schnorr, were adopted at the same time.

Though Schnorr is the heart of FROST, FROST _isn't_ the only way to
take advantage of Schnorr.

## The Power of MuSig

The main alternative to FROST is MuSig, an alternative Schnorr signing
protocol that now exists in its second major incarnation, called
MuSig2.

> :book: ***What is MuSig?*** MuSig is a Schnorr-based multisignature
protocol designed by Gregory Maxwell, Andrew Poelstra, Yannick Seurin,
and Pieter Wuille that creates aggregatable signatures that are
simpler and more efficient than traditional ECDSA multsigs.

> :book: ***What is MuSig2?*** MuSig was originally released as a
three-round protocol. When it was revamped into a second version,
MuSig2, the number of signing rounds was reduced to two. A further
variant called MuSig-DN protects against a specific sort of attack.

The biggest difference between FROST and MuSig is that FROST is
natively a threshold signature system (m of n where m≤n) while MuSig
is by default a multisig signature system (n of n), with threshold
signatures only accessible through the construction of Merkle trees
where each leaf holding a different n-of-n signature.

The other big difference between the two is on the question of
privacy. The privacy of FROST means that you have _deniability_: it's
impossible to see who signed an m-of-n threshold signature unless
multiple parties conspire to reveal information. In contrast, MuSig
has _accountability_: even if using a Merkle Tree to mimic a threshold
signature, you can always see exactly who signed.

Here's a chart of the major differences:

| | FROST | MuSig2 |
|---|---|---|
| **Signature Type** | Threshold (m-of-n) | Multisig (n-of-n) |
| **Signer Privacy** | Deniable | Accountable |
| **Bitcoin Integration** | Usable | Integrated |
| **Signing Rounds** | 2 or Preprocess | 2 |

Generally, you might use FROST for situations focused on threshold
signing and MuSig2 for Schnorr-based signatures on Bitcoin (though
this tutorial will demonstrate how to do FROST signing of Bitcoin
transactions).

## Summary: The FROST Signature Process

FROST is built on Schnorr, a powerful signature system whose biggest
advantage is aggregation, which allows the addition of signatures to
create combined signatures that are indistinguisable from single
signatures.

MuSig2 is another major signature protocol that also uses
Schnorr. MuSig's biggest difference is that it's not a threshold
system, though thresholds can be modeled by the use of Merkle
trees. FROST is likely a better choice if you require threshold
signatures, while MuSig2 might get the nod for Bitcoin integration.

## What's Next

You're ready to begin the hands-on tutorial! Continue onward to
[Chapter Two: Signing with FROST](02_0_Signing_with_FROST.md) to
beginn experimenting with FROST tools.

