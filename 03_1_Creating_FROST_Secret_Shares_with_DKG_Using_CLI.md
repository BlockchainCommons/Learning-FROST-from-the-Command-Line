# 3.1: Creating FROST Secret Shares with DKG Using CLI

[§2.2: Creating FROST Secret Shares with
TDG](02_2_Creating_FROST_Secret_Shares_with_TDG.md) overviewed the
simplest way to create FROST shares, using Trusted Dealer
Generation. However, the full power of FROST comes with the use of
Distributed Key Generation (DKG). With this methodology, the full
signing key never exists in a single place—not when you're creating
the signing shares, and not when you're using them.

## Creating DKG Shares the Hard Way

Creating secret shares with a Trusted Dealer is relatively simple: a
single server creates a secret, it shards it, and it sends those
shares out to the participants.

Creating secret shares with Distributed Key Generation is more
difficult because you have to engage in the two-round ceremony
described in [§1.2: The FROST Signature
Process](01_2_FROST_Signature_Process.md), first broadcasting
commitments to all participants, then sending to each
individually. Each member needs to both send and receive "n" messages
(1 broadcast message and "n-1" individual messages), where "n" is the
number of participants. This can be very logistically tiresome and
prone to error if a user has to do it all themself.

> :book: ***What is DKG?*** In Distributed Key Generation, multiple
servers work together. At the end of the ceremony, each server will
hold its own secret share without the original secret having been
instantiated at any point.

> :book: ***Why is DKG Better than TDG?*** DKG is considerably more
secure than TDG. With Trusted Dealer Generation, you have to trust the
trusted dealer (hence the name!). You have to trust not just that
they're not going to steal your secret, but also that they're secure
(so that someone else can't steal the secret), that their network is
secure, and that they will properly (and securely!) delete the secret
after it's sharded and distributed. None of this trust is required for
DKG! Because the secret never exists in one place, the _only_ way to
compromise it is to steal multiple shares (assuming the threshold is
larger than 1), and that should be difficult because the shares should
be securely stored on multiple sites.

> :book: ***How Do the ZF FROST Tools Support DKG?*** ZF FROST offers
two methods for generating DKG shares. One uses the `dkg` CLI to
generate shares, but requires the participants to do the exchange of
commitments themselves. The other uses the `server` app, which takes
some effort to setup, but which takes the logistical burden from the
user. The CLI method is described in this section, the server-based
functionality in the [next
section](03_2_Creating_FROST_Secret_Shares_with_DKG_Using_Server.md)..

This section demonstrates how to create DKG shares using the `dkg` CLI
that you already installed as part of the ZF FROST `frost-client`
package. It requires you to take care of all of the logistical
exchange of information.

To start with, each participant will run `dkg` from the command line
of their own machine, entering in the threshold and signing number
(which must match across all participants) and their own identifer
(which must be unique for each participant:
```
% dkg
The minimum number of signers: (2 or more)
2
The maximum number of signers:
3
Your identifier (this should be an integer between 1 and 65535):
1
```

This example demonstrates Alice's console, but Bob is doing the same
(with identifier `2`) as is Eve (with identifier `3`).

Each user will then be given a hex string version of their identifier
and a broadcast commitment that they must send to all the other
participants.

Here's what that looks like for Alice:
```
=== ROUND 1: SEND PACKAGES ===

Round 1 Package to send to all other participants (your identifier: "0100000000000000000000000000000000000000000000000000000000000000"):

{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"commitment":["f6b7158d8140d5503a2a4c46d6d3e7bfb52f132362a1464156ae553b0846fe7d","6e929c2076cb52266ff05e5406512881d098ff5666a85fc51487842be05076d5"],"proof_of_knowledge":"0624a9059dda9eb17a0ddad6a035eba27e951512495efb155f9fc4d9fddbcdf077ece4768ae32cecc6df9f98461a0d6a7f6a6a0ac26eb76ce814fe5dc820a70d"}
```

Alice should send her packets out, but she should also receive
packages from Bob and Eve. For each package that she receives, she
enters the identifier of the sender as well as the package into her
own `dkg` console:

```
=== ROUND 1: RECEIVE PACKAGES ===

Input Round 1 Packages from the other 2 participants.

The sender's identifier (hex string):
0200000000000000000000000000000000000000000000000000000000000000
Their JSON-encoded Round 1 Package:
{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"commitment":["272617337510d27c1d82792c2168b002770eb14fe3dd807a2571ce40099c3e14","09e8de0f722ff2b61be60ba6858e4c673386a23669b9f88c611dd3a8478a1f9e"],"proof_of_knowledge":"793f45332587cbd8419fd0e72e5ecf45084aef3bdb4d868355bb106d1b7224f7e399869a99f66fdb1a7c53b30c3870c9a64b340ec35cffaba22879835a994507"}

The sender's identifier (hex string):
0300000000000000000000000000000000000000000000000000000000000000
Their JSON-encoded Round 1 Package:
{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"commitment":["95817e076aeb2c4de3d506719da927c2cc99bd793f32d4bdf18795468b7e296b","679b7ae4066cceba11c790e1493e15614ea4eabe54ab0ae4ba6b6f14d7668d2a"],"proof_of_knowledge":"d0313c8ced787cfecccfba375a4b8a7142d82ec7102dbbd842e7443e647c290113422400464e363c5dfb9097a54919978714634341d50909e83a51954bac5c0d"}
```

When a participant has entered all of the packages from the other
participants, they'll be shown their round 2 packets, which must be
sent individually and securely to each participant:

```
=== ROUND 2: SEND PACKAGES ===

Round 2 Package to send to participant "0200000000000000000000000000000000000000000000000000000000000000" (your identifier: "0100000000000000000000000000000000000000000000000000000000000000"):

{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"signing_share":"36c4a4c744d69fba78516fb6cddf03d618ef77ac0a97de1e343d67b9f7eda00a"}

Round 2 Package to send to participant "0300000000000000000000000000000000000000000000000000000000000000" (your identifier: "0100000000000000000000000000000000000000000000000000000000000000"):

{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"signing_share":"62746f98a7626ffed4bf767ad971c655529ac6410b2e312970c7de8931b99c0d"}
```

They also will receive packages from each participant, which they must
again enter alongside the identifier for the other participant:

```
=== ROUND 2: RECEIVE PACKAGES ===

Input Round 2 Packages from the other 2 participants.

The sender's identifier (hex string):
0200000000000000000000000000000000000000000000000000000000000000
Their JSON-encoded Round 2 Package:
{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"signing_share":"3c18c0d51bffe439728569b2f3f2906a669458197ec0cbe1dc609b42e56d740b"}

The sender's identifier (hex string):
0300000000000000000000000000000000000000000000000000000000000000
Their JSON-encoded Round 2 Package:
{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"signing_share":"eca669a661c61dacf0a4b148eb37bfc02ca939b7905ea90513eeec30abd48700"}
```

As soon as a participant has entered all of the packages from the
other users (and even before those other users necessarily finish the
process), they'll be given their share of the signing key as well as
the public key package:

```
=== DKG FINISHED ===
Participant key package:

{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"identifier":"0100000000000000000000000000000000000000000000000000000000000000","signing_share":"45ff0d1645acc004a9708b4ac27eb26c7281bbe7181f01fce701785c4e65a103","verifying_share":"32bb144163d32dd166ab7534ee53a01fc6eac5dba9f2057ed2b38a842762cb9f","verifying_key":"67b2d1d24ed2f2cb78ffefe6699c4114e479f95621d5ab2720687dcf593bdcfe","min_signers":2}

Participant public key package:

{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"verifying_shares":{"0100000000000000000000000000000000000000000000000000000000000000":"32bb144163d32dd166ab7534ee53a01fc6eac5dba9f2057ed2b38a842762cb9f","0200000000000000000000000000000000000000000000000000000000000000":"1ae2f34429c0cf7b7757c2f537eada432e8573f06f7258f9510a013ff926159f","0300000000000000000000000000000000000000000000000000000000000000":"5276b15b25f29f7dd42e47cf5272f320e8a4d5ed9f27943d240dc29259c22985"},"verifying_key":"67b2d1d24ed2f2cb78ffefe6699c4114e479f95621d5ab2720687dcf593bdcfe"}
```

This information must be saved to files. The "Participant key package"
contains the secret info and should be protected accordingly. In these
examples it's stored as `key-package-#.json`. The "Participant public
key package" is the public info, including the verifying key, which
doesn't need to be secured. It should be stored as
`public-key-package.json`.

Whew!

You can follow along with this tutorial simply by running `dkg` in
three different windows (even on the same machine, though that
wouldn't be the case in a real-world deployment) and exchanging
information among them. It's a great hands-on example of how the
exchange detailed in [§1.2](01_2_FROST_Signature_Process.md) works,
but it also demonstrates how cumbersome the exchanges are for a user,
especially as the number of partipants increase. If you make a single
mistake, entering in the wrong participant or the wrong package at any
point, the entire process will fail, and you'll only know at the end!

That's why engineers need to design better processes that automate the
exchanges of the two rounds, which is exactly what happens when using
ZF FROST's server-based method of DKG creation, in
[§3.2](03_2_Creating_FROST_Secret_Shares_with_DKG_Using_Server.md).

## Signing with DKG Shares

If you saved all the data to files and named them appropriately, you
can now sign with your DKG shares using the same process you used for
signing with the TDG shares in [§2.3: Creating a FROST
Signature](02_3_Creating_FROST_Signature.md).

On the server:
```
% coordinator -m board-meeting-250917.txt -s board-meeting-250917.sig -n 2
```
On Alice's machine:
```
% participant -k key-package-1.json
```
On Bob's machine:
```
% participant -k key-package-2.json
```
The result should be just as before. Here's what the `coordinator`'s console shows:
```
Received: {"IdentifiedCommitments":{"identifier":"0100000000000000000000000000000000000000000000000000000000000000","commitments":{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"hiding":"a2e21b5a1963c011e2a1d9229ebcfccfc55d38f65347de2aa24852d154565476","binding":"f0258062753269bbecac60c49b4b6e7ed1c13b567ce603cc2919375c611e7055"}}}
Received: {"IdentifiedCommitments":{"identifier":"0200000000000000000000000000000000000000000000000000000000000000","commitments":{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"hiding":"db16d12c421c6779ed2bf059a4f15794b86d35f70a549b02981817602b55ffe5","binding":"90b477e71ab324183286d57397b19988d5aa186f61a70fbe088b9a3e62796544"}}}
Sending SigningPackage to participants...
Waiting for participants to send their SignatureShares...
Received: {"SignatureShare":{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"share":"7769b727b2d2cb4b34ea9c89fb05a14448ff925603fbd4dd34e76fea043ddf06"}}
Client disconnected
Received: {"SignatureShare":{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"share":"01460dba3c43ae9e481523568a95ac7cd6a2c2676ef23c673d3ce7b11574d00a"}}
Raw signature written to board-meeting-250917.sig
```

## Summary: Creating FROST Secret Shares with DKG Using CLI

Creating shares with Distributed Key Generation (DKG) can take some
work because it's a two-round process where each of `n` participants
need to send `n` messages, resulting in `n^2` total
communications. That's somewhat unwieldly for three participants and
totally unmanageable for many more. Nonethless, walking through it
once is a great example of the how the DKG process works.

## What's Next

For a more user-friendly example of DKG, see [§3.2: Creating FROST
Secret Shares with DKG Using
Server](03_2_Creating_FROST_Secret_Shares_with_DKG_Using_Server.md).
