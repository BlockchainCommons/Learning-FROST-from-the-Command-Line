# 1.2: The FROST Signature Process

So how does FROST actually work? This section describes the process
for both methods of key generation and both methods of signing,
without delving into the mathematical specifics. Read as much of it as
you find useful: the important thing is just understanding the
generalities.

For more of the math, see [the 2020 FROST
paper](https://eprint.iacr.org/2020/852.pdf) and one of its
foundations, Genarro et. al.'s ["Secure Distributed Key Generation for
Discrete-Log Based
Cryptosystems"](https://link.springer.com/content/pdf/10.1007/s00145-006-0347-3.pdf).

## The Key Generation Process

There are two major methods by which FROST keys can be created:
Trusted Dealer Generation (TDG) and Distributed Key Generation
(DKG). The first is simpler, the second is more secure.

### The Trusted Dealer Generation (TDG) Process

TDG depends on a naive security assumption: trust in a centralized
server. The TDG process is equally naive and simple. It creates a key
and shards it with Shamir's Secret Sharding before sharing out the
shards:

1. Trusted Dealer creates private key.
2. Trusted Dealer shards private key for "n" members with a threshold of "m".
3. Trusted Dealer sends private key shares & group verifying key to FROST group members.
4. Trusted Dealer (hopefully) erases secret & shares.

<a href="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/keygen-tdg.png">
    <img src="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/keygen-tdg.png" style="border: 1px solid black; margin: auto !important; display: block !important; width: 600px;">
</a>

The secret could also be generated in other ways, such as a member
creating the secret and handing it to the dealer for distribution.
    
### The Distributed Key Generation (DKG) Process

The FROST paper suggests PedPoP for Distributed Key Generation (DKG)
in FROST: Pedersen DKG with Proofs of Possession, which is [Torben
Pedersen's DKG
system](https://link.springer.com/chapter/10.1007/3-540-46766-1_9) as
modified by Rosario Gennaro and others to support [Verifiable Secret
Sharing
(VSS)](https://link.springer.com/article/10.1007/s00145-006-0347-3). But
since, more variants have been offered, such as
[ChillDKG](https://github.com/BlockstreamResearch/bip-frost-dkg),
which further modifies PedPoP into SimplPedPop, simplifying PedPoP and
also making it more securely usable with Bitcoin Taproot.

> :warning: **WARNING:** The lesson here is that DKG is not fully
defined for use with FROST. There are a number of different DKG
methodologies that could be used. They're all legitimate as long as
they generate a set of key shares in a distributed manner that is
secure for your security assumptions.

The specifics likely don't manner to you if you're using a library or
CLI that has already determined how to generate its shares using
DKG. However, here's the general process utilized by the ZF FROST
package that's used in this tutorial. It's laid out as "inputs" to the
ZF FROST software, "outputs" from the ZF FROST software, and
"transmits" among members.

**Prep:**

1. Via some means, the participants agree on member count ("n") and
threshold ("m") for the signature (an "m of n" threshold signature).

**Round 1:**

2. Each member inputs their identifier, member count ("n") and
threshold ("m").
3. Each member outputs public data and secret data.
   * Secret data includes two polynomials, which contain the member's secret.
   * Public data is a commitment to that secret, in the form of a Pedersen-VSS share.
4. Each member transmits their public data to all other members via a broadcast.
   * At this point, the joint secret has been fixed.
   
**Round 2:**

5. Each member inputs their secret data from round 1 and the public
data from all the other members from round 1, keyed to the identifier
for each member.
6. Each member outputs a second set of secret data.
   * Secret data now includes the member's share of the joint secret: their private key share.
7. Each member outputs response data for each other member.
   * Response data includes a commitment using Feldmann-VSS.
8. Each members transmits the response data to each other member via an authenticated and encrypted personal communication channel.

**Post:**

9. Each members inputs their second set of secret data, the public
data from all other members received at the start of Round 2, and the
response packages received back from all members at the end of Round
2.
10. Each member outputs their share of the private key and the full
group verifying key.

<a href="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/keygen-dkg.png">
  <img src="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/keygen-dkg.png" style="border: 1px solid black">
</a>

## The Signing Process

Signing with FROST is a two-round process, but the first round can be
pre-processed to lower overhead at signing time.

### The Two-Round Process

The two-round FROST signature process is best described in
[§5](https://datatracker.ietf.org/doc/html/rfc9591#name-two-round-frost-signing-pro)
of [RFC 9591](https://datatracker.ietf.org/doc/html/rfc9591). Round 1
is commitment and round two is signing. A signing coordinator is
involved in the process by default: they don't have any special
authority or knowledge, but they do take care of the logistics, and as
a result could censor the process if they wanted.

**Round 1:**

1. Each member creates two nonces and commitments to those nonces.
2. Each member transmits the commitments to the coordinator.

**Round 2:**

3. Coordinator transmits message to chosen members to sign.
4. Coordinator transmits relevant commitments to chosen members.
5. Each member checks this information.
6. Each member uses their secret key share and nonce to sign.
7. Each member transmits the resultant signature share to coordinator.
8. Each member deletes the nonce used and corresponding commitment.

**Post:**

9. Coordinator checks signature shares.
10. Coordinator aggregates signature shares to produce signature.

<a href="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/signing-1.png">
  <img src="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/signing-1.png" style="border: 1px solid black">
</a>

### The Pre-Processed Process

FROST signatures can occur in a single round if the nonces and
signatures are preprocessed:

**Preprocessing:**

1. Each member generates a list of several nonces and commitments.
2. Each member publishes their commitments in some way.

**Round 1:**

3. Coordinator selects next commitments from list for chosen members.
4. Coordinator transmits message to chosen members to sign.
5. Coordinator transmits relevant commitments to chosen members.
6. Each member checks this information.
7. Each member uses their secret key share and nonce to sign.
8. Each member transmits the resultant signature share to coordinator.
9. Each member deletes the nonce used and corresponding commitment.

**Post:**

10. Coordinator checks signature shares.
11. Coordinator aggregates signature shares to produce signature.

<a href="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/signing-2.png">
  <img src="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/signing-2.png" style="border: 1px solid black">
</a>

Algorithmically, there is no difference between the two-round and
pre-processing variants of FROST signatures. It's simply a question of
whether the nonces and commitments are generated and stored in advance
(pre-processing) or generated on the fly (two-round).

## Summary: The FROST Signature Process

There are two ways to create FROST private key shares: through a
standard use of Shamir's Secret Sharing or through a more complex
Distributed Key Generation sequence that may vary from one FROST
library to another.

There are two ways to sign with FROST: through a two-round process or
through an equivalent process where the first-round has been
pre-processed and stored.

You likely don't need to know the specifics of how each works, but you
should understand the two main variations for key generation and
signing, so that you can recognize the options offered to your by
various libraries.

## What's Next

Continue your "Introduction to FROST" by learning more about the
underlaying Schnorr signing system and the biggest competitor to FROST
in [§1.3: FROST vs MuSig2](01_3_FROST_vs_MuSig.md).

Or, if you've seen enough about the nuts and bolts, jump straight to
[Chapter Two: Signing with FROST](02_0_Signing_with_FROST.md) for the
hands-on tutorial.
