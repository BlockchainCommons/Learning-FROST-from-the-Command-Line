# 3.3 ZF FROST in a Nutshell

The previous chapters demonstrated a few different ways to create
shares and to create signatures. Following are some diagrams and
discussions of how they work.

## Share Creation

Creating signing shares is the first step in FROST.

### TDG Creation

TDG share creation requires a centralized, trusted authority, who
unilaterally splits a secret and shards it out to participants.

See [§2.2](02_2_Creating_FROST_Secret_Shares_with_TDG.md) for more.

<p align="center">
  <a href="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/zf-tdg.png">
    <img src="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/zf-tdg.png" style="border: 1px solid black; width: 400px;">
  </a>
</p>


### DKG by Hand

DKG is in contrast trustless, but also more complex, as most trustless
protocols are.  When managing DKG by hand, each member has to
communicate with every other member. (They do one broadcast each for
round 1, then send an individual message to each other paricipant for
round 2. This is the heart of how FROST share creation works.)

See [§3.1](03_1_Creating_FROST_Secret_Shares_with_DKG_Using_CLI.md) for more.

<p align="center">
  <a href="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/zf-dkg-1.png">
    <img src="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/zf-dkg-1.png" style="border: 1px solid black; width: 400px;">
  </a>
</p>


### DKG with Server

A server can take care of that communication, and greatly simplify
things, and it doesn't have to be trusted, offering the
best-of-both-worlds.

See [§3.2](03_2_Creating_FROST_Secret_Shares_with_DKG_Using_Server.md) for more info.

<p align="center">
  <a href="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/zf-dkg-2.png">
    <img src="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/zf-dkg-2.png" style="border: 1px solid black; width: 500px;">
  </a>
</p>

## Signing

Once shares have been created, members of a FROST group can sign
whenever they want.

### TDG Signing

ZF FROST manages TDG signing by running a `coordinator` server than
the signers all connect with using their `participant` client.

See [§2.3](2_3_Creating_FROST_Signature.md) for more.

<p align="center">
  <a href="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/zf-signing-1.png">
    <img src="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/zf-signing-1.png" style="border: 1px solid black; width: 500px;">
  </a>
</p>

### DKG Signing with Hand-Created Shares

The exact same process can be used for signing with hand-created
shares under ZZF FROST.

### DKG Signing with Server-Created Shares

The signing process for shares created with ZF FROST is a stacked
affair. The same `frostd` server that managed communication during
share creation now runs a signing `coordinator`.

See [§3.2](03_2_Creating_FROST_Secret_Shares_with_DKG_Using_Server.md) for more info.

<p align="center">
  <a href="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/zf-signing-2.png">
    <img src="https://raw.githubusercontent.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/refs/heads/main/images/zf-signing-2.png" style="border: 1px solid black; width: 600px;">
  </a>
</p>

## Summary: ZF FROST in a Nutshell

ZF FROST offers a number of different ways to create shares and sign
with them. This section lays them out graphically, in part for
clarity, and in part to suggest models that other developers might
use.

## What's Next

Though it's not available in the ZF FROST tools, we want to overview
one other advanced FROST capability in [§3.4: Refreshing FROST
Shares](03_4_Refreshing_FROST_Shares.md)
