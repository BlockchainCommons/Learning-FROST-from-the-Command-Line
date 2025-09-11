# 1.1: Introducing FROST

FROST stands for Flexible Round-Optimized Schnorr Threshold
Signatures. As the name indicates, it's a style of Schnorr signature
that supports M of N threshold signatures. Though there is a [2020
paper](https://eprint.iacr.org/2020/852.pdf) by Chelsea Komlo and Ian
Goldberg that gives all the specifics, this section is intended to
offer a more accessible introduction to the technology.

## The FROST Overview

FROST is a threshold signature scheme. That means that a group of
participants each hold a share of a private key. The private key does
not exist in full, but only in parts, as these shares. The shares can
be used together to create signatures for the private key, with only a
"threshold" of shares required to generate the signature.

This threshold is determined when the key is generated. It might be
the whole group of signers or some smaller number. Common thresholds
include "1 of 2", "2 of 3", and "3 of 5". These mean that one out of
two shares is required to sign, or two out of three, or three out of
five, respectively.

There is also a public key, which does exist in full, and can be used
to validate signatures made by the private key shares. It's sometimes
called a _group verifying key_.

> :book: ***What is a Public Key?*** A public key is a publicly
distributed cryptographic code that might act as an address or an
identifier. It is generated from a private key and can be used to
validate signatures made by the private key.

> :book: ***What is a Private Key?*** A private key is a secret
cryptographic code. It is often used to digitally sign things. Doing
so can prove that the owner controls a specific public key.

> :book: ***What is a Share?*** A key can be sharded with a system
such as Shamir's Secret Sharing. This generates a set of "n"
shares. The original key can then be reconstructed from "m" of the
shares where "m≤n", but if fewer than "m" of the shares are brought
together, they tell nothing about the secret.

> :book: ***What is VSS?*** VSS, or Verifiable Secret Sharing, is a
variant of Shamir's Secret Sharing. It allows the verification of
shares, so that recipients know they actually define a secret, which
is crucial to some FROST methods of key generation. (VSS shares can
also prove the continued existence of the shares without actually
revealing them, which is great for secret resilience, but less
important for FROST itself.)

Beyond being a threshold signature system, FROST is also a Schnorr
signature system, which refers to the exact process (algorithm) used
to generate the digital signature. Schnorr has some advantages over
traditional signatures related to privacy and efficiency, as discussed
below.

## FROST without the Math

The FROST signature scheme has two main elements.

First, the members of a FROST group must generate shares of a private
key that they will then use to sign messages. They do so by first
determining the number of group members and the threshold required for
signing, and then engaging in a key-generation ceremony. This can be
done in one of two ways:

**Trusted Dealer Generation (TDG):** This is a traditional method. One
trusted, centralized server generates a private key (or is given a
key), shards it, and sends out the shares to the members of the FROST
group. The disadvantage is that the server _must_ be very trusted, and
the key exists in a single place for at least some time, allowing it
to potentially be stolen.

**Distributed Key Generation (DKG):** This is a more secure, but also
newer and more complex method. It takes advantage of Secure
Multi-Party Computing (MPC) and VSS. The members of the FROST group
work together to generate their shares of the private key, with each
member creating their personal share over the course of the ceremony,
but with the combined private key never coming into existence, and no
member ever learning any share but their own.

Whichever way the individual shares are generated, the process will
also generate a combined public key that can be used to validate
signatures made by the private key shares.

Once the members of FROST group have created their signing shares,
they can then sign. This is done by creating a commitment to a nonce,
then creating a signature. Again, there are two ways to do this:

**Pre-Processing:** Each member of the FROST group generates a set of
nonces and a set of commitments for those nonces, and stores the
commitments on a centralized server. The signing process can then be
done in a single-round signing process

**Two-Round Process:** The members of the FROST group engage in a
first round to generate nonces (which they hold) and commtiments
(which they share), and then a second round to generate the
signature. This makes the process more complex, but removes the
dependence on a single server.

The signing process is usually overseen by a _signature aggregator_
(or _cooridnator_) who passes everything to the participants without
having any special privileges. Again, the literal middle-man can be
cut out if desired, in which case all elements of the signing process
must be broadcast in a secure way.

The next chapter, [§1.2: The FROST Signature
Process](01_2_FROST_Signature_Process.md), details these specifics in
more detail. The original paper, ["FROST: Flexible Round-Optimized
Schnorr Threshold Signatures"](https://eprint.iacr.org/2020/852.pdf),
by Chelsea Komlo and Ian Goldberg, may be consulted for even more
specifics. But you only need to know enough to be comfortable with
using FROST! So dive only as deep you like; just be sure that you're
comfortable with the difference between TDG and DKG and the fact that
key shares are first generated and then used for signing.

## The Usage of FROST

_So why would you use FROST?_ Threshold signatures have generally been
used to give a group of people control over cryptocurrencies. However,
FROST threshold signatures can also be used to authenticate any
consensus protocol. It could allow a board of directors to sign off on
directives for their organization, or a set of members to make
decisions for a DAO.

Generally, if you have a group of people and you want to get
permission from either the whole group or alternatively some subset,
then a threshold signature is the way to go.

## The Advantages of FROST

FROST has a number of advantages that set it aside from classic
threshold signature systems, such as Bitcoin's multisig. Some of these
are due to the use of Schnorr signatures, some of them are due to the
specific design of FROST.

**Small Signatures.** FROST signatures are aggregateable. The
individual signatures are effectively added together. As a result, the
final multi-signature is the same size as a single signature! This
makes it much smaller than most traditional multisigs, which instead
require each and every participant to append their signature to an
ever-growing message. That means that very large thresholds (say, 101
of 200, to represent consensus for a group) are easy under FROST,
where they were totally unfeasible under traditional systems.

**Private Signatures.** You also can't distinguish between a single
signature and a multi-signature. It's not just that they're the same
size, but also that they look just the same. This makes FROST
signatures much more private because a third party doesn't know if a
signature represents a single individual or multiple individuals. And
if it's multiple individuals, the third party does know which members
of the FROST group contributed to the threshold.

**Efficient Communication.** FROST was built specifically to make the
signing process simple and efficient. When pre-processing is used to
register commitments in advance, signing can be done in a single
round: signers can act asynchronously and all each needs to do is
reply to a single message to create a signature. This lowers network
load and makes threshold signatures possible on network-limited
devices. It also allows for use cases where some participants are
offline or airgapped, increasing security.

**Repair & Refresh Capabilities.** FROST shares can be repaired
(allowing the recovery of lost shares or the creation of new shares)
or refreshed (allowing the changing of current shares). This not only
allows the shares to change over time, but also allows the threshold
to change. A 2 of 3 threshold could, for example, become a 3 of 5 with
a few repair operations. (However, it should be noted, that old shares
and even old participants still remain trusted by the FROST group; if
this is not actually the case, then a FROST group needs to be entirely
replaced with a new one.)

**Strong Security with DKG.** If DKG is used, the private key never
exists in a single place. This provides very strong security because
there is no Single Point of Compromise (SPOC) if the threshold is 2 or
greater. Instead, an attacker would have to steal multiple shares from
multiple locations.

The biggest trade-off for all of these advantages is a reduction in
_robustness_. Some existing threshold signature systems are better at
keep malicious actors from censoring (blocking) the use of a
multisig. FROST can still do so, but only by recognizing a bad actor
and purposefully excluding them from the process. A wrapper called
[ROAST](https://eprint.iacr.org/2022/550.pdf) can resolve the issue of
robustness, but it's not necessary in many use cases.

## Using FROST

FROST can be used anywhere that you need to make a signature.

This tutorial focused on the [ZF
FROST](https://frost.zfnd.org/index.html) project that was initiated
by the [Zcash Foundation](https://zfnd.org/), which has wide
applicability. (It's not just for Zcash.)

They have a [Rust FROST
library implementation](https://github.com/ZcashFoundation/frost/), which
supports several different ciphersuites using either TDG or DGK.

They also have [CLI
tools](https://github.com/ZcashFoundation/frost-tools) to demo FROST
that are used in this tutorial (along with some of our own tweaks and
the [Bitcoin Dev Kit](https://bitcoindevkit.org/).

## Summary: Introducing FROST

FROST is a threshold signature system built using Schnorr
signatures. It is intended to be more efficient than other similar
protocols by minimizing the number of rounds of communication required
for signatures. It also has advantages in size, efficiency, and
privacy and provides additional capabilities because it can rewrite
thresholds and groups after the fact.

## What's Next

Continue your "Introduction to FROST" by learning some of the
specifics of how FROST works in ["§1.2: The FROST Signature
Process"](01_2_FROST_Signature_Process.md).

Or, if you've seen enough about the nuts and bolts, jump straight to
[Chapter Two: Signing with FROST](02_0_Signing_with_FROST.md) for the
hands-on tutorial.