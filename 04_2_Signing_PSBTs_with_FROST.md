# 4.2: Signing PSBTs with FROST

You've now got all the tools you need to successfully send a Bitcoin
transaction signed by a FROST group. But it's still going to take some
work! This section will take you through the following major steps:

***Setup:***

1. Setting Up Your Work Space
2. Starting a Regtest Daemon
3. Creating and Funding a Bitcoin Wallet
4. Creating a FROST Group with DKG
5. Creating a FROST Bitcoin Wallet
6. Sending Funds to the FROST Wallet

***Signing:***

7. Creating an Unsigned PSBT for the FROST Wallet
8. Generating a Signature using the FROST Group
9. Signing the PSBT using the FROST Group
10. Finalizing and Sending the PSBT

## Step 1: Setting Up Your Work Space

First, set some shell variables that will make the rest of this
tutorial easier.
```
DEMO_DIR=`pwd`/frost-signing-demo
REG_DIR="$DEMO_DIR/regular-wallet"
FROST_DIR="$DEMO_DIR/frost-wallet"
NETWORK=regtest
RPC_URL=127.0.0.1:18443 
RPC_USER=test
RPC_PASS=test
RPC_AUTH="$RPC_USER:$RPC_PASS"
```
Then create your directories:
```
mkdir -p "$DEMO_DIR" "$REG_DIR" "$FROST_DIR"
cd "$DEMO_DIR"
```

## Step 2: Starting a Regtest Daemon

This example uses the "regtest" network as a testing environment for
Bitcoin rather than working with real funds.

> :book: ***What is Regtest?*** Bitcoin offers three different testing
environments: testnet, signet, and regtest. Testnet works much like
the regular Bitcoin network, but its coins have no value. Signet
requires blocks be signed. Regtest gives the developer complete
control over block creation. This example uses regtest for its testing
so that you don't have to go to coin faucets or wait for transactions
to be collected into blocks.

To initiate regtest, start the Bitcoin Core `bitcoind`. Set flags to
use the regtest network (`-regtest`), to run in the background
(`-daemon`), to set an RPC user and password (`-rpcuser` and
`-rpcpassword`), and to set a fallback transaction fee, so that your
transactions will go through (`-fallbackfee`).

```
bitcoind -regtest -daemon -rpcuser=$RPC_USER -rpcpassword=$RPC_PASS -fallbackfee=0.00001
```

> :warning: **WARNING:** Obviously, you would never use the `-rpcuser`
and `-rpcpassword` arguments from the command-line for a real-world
deployment, as they could be seen in the process list. But, they're
fine for a test usage with regtest.

> :information_source: **NOTE:** You'd usually set up a `bitcoin.conf`
file that would records things like your `$RPC_USER`, your
`$RPC_PASS`, and the fact that you're using the regtest network. These
examples will just include all of that material on the command line,
which makes them longer, but also means you don't have to figure out
where your configuration file is, which is probably preferred for a
short Bitcoin example.

## Step 3: Creating and Funding a Bitcoin Address

You'll first need to create your default regular Bitcoin wallet. This
used to happen by default with `bitcoin-cli`, but no longer does.

```
bitcoin-cli -rpcuser=$RPC_USER \
            -rpcpassword=$RPC_PASS \
            -regtest \
            createwallet regtest || \
bitcoin-cli -rpcuser=$RPC_USER \
            -rpcpassword=$RPC_PASS \
            -regtest \
	    loadwallet regtest

{
  "name": "regtest"
}
```

You can now create an address in that wallet:
```
REG_ADDRESS=$(bitcoin-cli -rpcuser=$RPC_USER \
  			    -rpcpassword=$RPC_PASS \
			    -regtest \
			    getnewaddress)
```
Finally, you can mine the genesis block to that address:
```
bitcoin-cli -rpcuser=$RPC_USER \
              -rpcpassword=$RPC_PASS \
	      -regtest \                               
	      generatetoaddress 1 $REG_ADDRESS
[
  "51817c46a4ba678ce1e64970eb1978e8827b7272409b877c3abe4849b01ae68b"
]
```

The `generatetoaddress` command is one of the magic regtest commands
that lets you manipulate blocks to more easily test out Bitcoin. In
this case, you are mining a (empty) block with the block reward for
doing so going to the indicated address. So this command mines 1 block
to your $REG_ADDRESS. With the 50 BTC reward for the first block,
you're rich! (In fake regtest bitcoin.)

Afterward, you will advance the blockchain by another 100 blocks:
```
bitcoin-cli -rpcuser=$RPC_USER \
              -rpcpassword=$RPC_PASS \
              -regtest \
              -generate 100
```
> :book: ***Why Mine 101 Blocks?*** The reward a miner gets for mining
a block is called the "coinbase transaction". It can only be spent
when it's 100 blocks deep in the blockchain. After you mine 101
blocks, the first block you mined is 100 blocks deep, which means you
can now spend the 50 Bitcoins you claimed for it.

At this point, mining those additional 100 blocks will send the
coinbase transactions to your wallet, even without the
`generatetoaddress` command: they'll show up as you continue to
advance blocks.

Before you continue, you should verify you have coins to spend:
```
bitcoin-cli -rpcuser=$RPC_USER -rpcpassword=$RPC_PASS -regtest getbalance
50.00000000
```

## Step 4: Creating a FROST Group with DKG

You can now create a Frost group using DKG, just like you did in
[§3.2](03_2_Creating_FROST_Secret_Shares_with_DKG_Using_Server.md)
with one notable difference: you're going to be using the
`secp256k1-tr` ciphersuite. Otherwise, everything else is identical.

Here's the rapid-fire summary (with the reminder that in a real setup,
each of the users should be on a different machine):

Create the user credential files:
```
cd frost-wallet
frost-client init -c alice.toml
frost-client init -c bob.toml
frost-client init -c eve.toml
```
Extract the public keys:
```
ALICE_PUBKEY=$(tq -f alice.toml -r 'communication_key.pubkey')
ALICE_CONTACT=$(frost-client export --name "Alice" -c alice.toml 2>&1 | tail -1)
BOB_PUBKEY=$(tq -f bob.toml -r 'communication_key.pubkey')
BOB_CONTACT=$(frost-client export --name "Bob" -c bob.toml 2>&1 | tail -1)
EVE_PUBKEY=$(tq -f eve.toml -r 'communication_key.pubkey') 
EVE_CONTACT=$(frost-client export --name "Eve" -c eve.toml 2>&1 | tail -1)
```
Have each of the users transmit the information on their public keys
to all the other users.

Have each user import the other users' info into their credentials file:
```
frost-client import -c alice.toml $BOB_CONTACT
frost-client import -c alice.toml $EVE_CONTACT

frost-client import -c bob.toml $ALICE_CONTACT
frost-client import -c bob.toml $EVE_CONTACT

frost-client import -c eve.toml $ALICE_CONTACT
frost-client import -c eve.toml $BOB_CONTACT
```

Make sure your `frostd` server is running with its certificate. If you
skipped over that in §3.2, you can jump back there to ["Prepare the
Server"](secp256k1-tr).

Alice then runs the `frost-client` with the other `PUBKEY`s and the
new `secp256k1-tr` cipher:
```
frost-client dkg -d "Bitcoin DKG: Alice, Bob, Eve" \
-s 127.0.0.1:2744 \
-S $BOB_PUBKEY,$EVE_PUBKEY \
-t 2 \
-C "secp256k1-tr" \
-c alice.toml
```
Bob runs the client without the `PUBKEY`s.:
```
frost-client dkg -d "DKG: Alice, Bob, Eve" \
-s 127.0.0.1:2744 \
-t 2 \
-C "secp256k1-tr" \
-c bob.toml
```
Eve runs the client:
```
frost-client dkg -d "DKG: Alice, Bob, Eve" \
-s 127.0.0.1:2744 \
-t 2 \
-C "secp256k1-tr" \
-c eve.toml
```
As before, the clients will do all the talking among themselves (via
the server) and then all the info will be output to the credentials files.

> :information_source: **NOTE:** You installed two updates to the ZF
FROST tools in §4.1. The first of them, which adds secp256k1-tr as a
ciphersuite, is used here. (As it happens, the second update is also
hiding out the background, but more on that when we put it to more
complete use during the signing ceremony.)

Here's what Alice's credentials now look like:
```
version = 0

[communication_key]
privkey = "51e60d2eb443d1955a862bc355e7ac5bbaea3fe171f056c4604748e9d1eea82b"
pubkey = "9ed1c448119a6d1441aa2ca2fd584691ff122339e870780b247941cfa06d8d75"

[contact.Bob]
name = "Bob"
pubkey = "61c0af67bea157aad7b0e8f4e27353281159fb747d91c48d39328dcdec4c6272"

[contact.Eve]
name = "Eve"
pubkey = "fe36fe0b20ce9eff3e2050b3e24ce0e57b093425b8b5980c222f9e6d7df14b30"

[group.02213e68e3d071ebce9c491114dcbdc1ee76ed3f52b30f05b5f27d36f6fdf37c85]
description = "Bitcoin DKG: Alice, Bob, Eve"
ciphersuite = "FROST-secp256k1-SHA256-TR-v1"
public_key_package = "00230f8ab303348fd8ec2919a79d9976d5cf79626a74e99eb1004c05b115cb72f7e89f87fb2a0388881328732b7f62744e253bc980bd605bf75cefb18116a0d856d83e04857982e6aaeb3554d0a68d930c66bf206f63dc9306d358bfa18cf4712cbe3693097adc02bc496dfe1914c323b716f788130864d181a5aa4bb0154702d501941b310986fceeec13b5314aa5b0a8ce12dc6786a92a7f571cbaf5d2de1307d78ee98a7f1d1802fc61ad07349a15ceb8c60a0ba09c6c4f50f032f815495e5d865e158f5ac83629025106c0bcb903ba03587afbf2c769b2fcee2c8518ac383ff0849abd6bf46ba7eb"
key_package = "00230f8ab3348fd8ec2919a79d9976d5cf79626a74e99eb1004c05b115cb72f7e89f87fb2a739c7d396e81a7929843d02d75aff78b5dfbe1c4c72d1c5c14eb0043b7100a910388881328732b7f62744e253bc980bd605bf75cefb18116a0d856d83e04857982025106c0bcb903ba03587afbf2c769b2fcee2c8518ac383ff0849abd6bf46ba7eb02"
server_url = "127.0.0.1:2744"
internal_key = [
    81,
    6,
    192,
    188,
    185,
    3,
    186,
    3,
    88,
    122,
    251,
    242,
    199,
    105,
    178,
    252,
    238,
    44,
    133,
    24,
    172,
    56,
    63,
    240,
    132,
    154,
    189,
    107,
    244,
    107,
    167,
    235,
]

[group.02213e68e3d071ebce9c491114dcbdc1ee76ed3f52b30f05b5f27d36f6fdf37c85.participant.348fd8ec2919a79d9976d5cf79626a74e99eb1004c05b115cb72f7e89f87fb2a]
identifier = "348fd8ec2919a79d9976d5cf79626a74e99eb1004c05b115cb72f7e89f87fb2a"
pubkey = "9ed1c448119a6d1441aa2ca2fd584691ff122339e870780b247941cfa06d8d75"

[group.02213e68e3d071ebce9c491114dcbdc1ee76ed3f52b30f05b5f27d36f6fdf37c85.participant.e6aaeb3554d0a68d930c66bf206f63dc9306d358bfa18cf4712cbe3693097adc]
identifier = "e6aaeb3554d0a68d930c66bf206f63dc9306d358bfa18cf4712cbe3693097adc"
pubkey = "fe36fe0b20ce9eff3e2050b3e24ce0e57b093425b8b5980c222f9e6d7df14b30"

[group.02213e68e3d071ebce9c491114dcbdc1ee76ed3f52b30f05b5f27d36f6fdf37c85.participant.eeec13b5314aa5b0a8ce12dc6786a92a7f571cbaf5d2de1307d78ee98a7f1d18]
identifier = "eeec13b5314aa5b0a8ce12dc6786a92a7f571cbaf5d2de1307d78ee98a7f1d18"
pubkey = "61c0af67bea157aad7b0e8f4e27353281159fb747d91c48d39328dcdec4c6272"
```
Most importantly, note that this enew FROST group uses the Bitcoin-compatible ciphersuite:
```
ciphersuite = "FROST-secp256k1-SHA256-TR-v1"
```

Again, this part of the process was identical to what was in
[§3.2](03_2_Creating_FROST_Secret_Shares_with_DKG_Using_Server.md)
other than the use of the "secp256k1-tr" ciphersuite on the
`frost-client dkg` command lines.

## Step 5: Creating a FROST Bitcoin Wallet

Though you now have a Bitcoin-ready FROST signing group, you still
need to create a Bitcoin address for your FROST key. To do so, you
must first extract some information about your FROST group.

The following does so from Alice's credentials file.

```
GROUP_ID=$(grep -oE '^\[group\.[0-9a-f]+' alice.toml | head -1 | sed 's/^\[group\.//')
PUBKEY_PKG=$(tq -f alice.toml -r group.$GROUP_ID.public_key_package)
FROST_AGG_KEY=$(echo $PUBKEY_PKG | tail -c 65)
```
Note that the last variable (`FROST_AGG_KEY`) depends on knowing that
the group verifying key is the last 32 bytes of the `public_key_package`.

With that in hand, you create a descriptor for this address:
```
FROST_EXT_DESC="tr($FROST_AGG_KEY)"
```

> :book: ***What is a Descriptor?*** In the bad-'ole days, there was a
proliferating set of methods for describing Bitcoin wallets. As
different derivation methods appeared, this became problematic because
it increased the odds of losing access to funds: you might know a seed
or an HD key, but not how to get to the derivative key where funds
were actually stored! [BIP 380](https://en.bitcoin.it/wiki/BIP_0380)
defined "output descriptors", a new methodology for defining wallets
that included key material, derivation material, and methodology.

> :book: **What is a "tr()" descriptor?** A "tr()" descriptor defines
a Taproot address, per [BIP
386](https://github.com/bitcoin/bips/blob/master/bip-0386.mediawiki). It
takes a key as an argument that may have been overlaid with a tree of
additional ways to unlock an address. It defines a P2TR address.

With that variable, you can now use `bitcoin-cli` to output the
Taproot address linked to your group verifying key. (Due to the power
of output descriptors, essentially all you're doing is asking for the
address associated with "tr(your-public-key)".)

This comes in two parts. First you have to fully qualify the
descriptor by adding a checksum: `bitcoin-cli` requires this for all
work with descriptors. Then you can ask `bitcoin-cli` to generate the
address.

You can do this with the following commands:
```
FROST_EXT_DESC_WITH_CS=$(bitcoin-cli -rpcuser=$RPC_USER -rpcpassword=$RPC_PASS -regtest getdescriptorinfo $FROST_EXT_DESC | jq -r .descriptor)
FROST_ADDRESS=$(bitcoin-cli -rpcuser=$RPC_USER -rpcpassword=$RPC_PASS -regtest deriveaddresses $FROST_EXT_DESC_WITH_CS | jq -r '.[0]')
```
That should convert your FROST group verifying key (public key) into a
valid Bitcoin address:
```
% echo $FROST_ADDRESS
bcrt1pyylx3c7sw84ua8zfzy2de0wpaemw606jkv8std0j05m0dl0n0jzsuv3mk6
```

## Step 6: Sending Funds to the FROST Wallet

You're now ready to send funds from your Bitcoin wallet to your FROST
wallet. This is simply done with the `sendtoaddress command`:
```
bitcoin-cli -rpcuser=$RPC_USER \
	    -rpcpassword=$RPC_PASS \
	    -regtest \
	    sendtoaddress $FROST_ADDRESS 10.0
```
You'll get back a transaction ID:
```
f11f5188bf65f24aac5d4f6e943e0e2def511b2188bc0aec43291b1880a5864f
```

Since you're living in a regtest world, you need to create a new block
to log that transaction:

```
bitcoin-cli -rpcuser=$RPC_USER \
              -rpcpassword=$RPC_PASS \
              -regtest \
              -generate 1
```
Before you move on, you should make sure the transfer of funds
occurred as expected.

First, you can check the balance on your regular Bitcoin wallet:
```
bitcoin-cli -rpcuser=$RPC_USER -rpcpassword=$RPC_PASS -regtest getbalance

89.99999835
```

The new balance is 89.99999835 BTC = 50 BTC (original balance) - 10
BTC (sent) - 0.00000165 BTC (fallback fee) + 50 BTC (coinbase for
block #2, since we're now on block #102).

As for the FROST wallet, you'll be using `bdk-cli` to look at that,
since `bitcoin-cli` doesn't do a good job of working with addresses
that aren't in its wallet (and since it's convenient to use two
different apps to control your two different wallets, particularly in
a tutorial).

BDK wants a bit more information on its wallets, which it divides
between an "external descriptor" (which you've already defined) and an
"internal descriptor", so you'll need to set that first.

```
FROST_INT_DESC="tr($FROST_AGG_KEY,pk($FROST_AGG_KEY))"
```

> :book: ***What are External & Internal Descriptors?*** BDK uses a
classic methodology of separating addresses that are meant to be given
out (external) and those only intended for change (internal).

> :book: **What is a "pk()" descriptor?** A "pk()" descriptor defines
an address by its public key, per [BIP
381](https://en.bitcoin.it/wiki/BIP_0381#pk()). it describes a classic
P2PK address.

First, you must ask BDK to sync a wallet with those descriptors:
```
bdk-cli \
  --network "$NETWORK" \
  --datadir "$FROST_DIR" \
  wallet \
    --client-type rpc \
    --database-type sqlite \
    --url "$RPC_URL" \
    -a "$RPC_AUTH" \
    --ext-descriptor "$FROST_EXT_DESC" \
    --int-descriptor "$FROST_INT_DESC" \
    sync
```

Your `bdk-cli` commands will generally require defining a `wallet` and
then giving it a command, like `sync`. For example, to check that the
funds have gone through, you now access `wallet ... balance`.

```
bdk-cli \
  --network "$NETWORK" \
  --datadir "$FROST_DIR" \
  wallet \
    --client-type rpc \
    --database-type sqlite \
    --url "$RPC_URL" \
    -a "$RPC_AUTH" \
    --ext-descriptor "$FROST_EXT_DESC" \
    --int-descriptor "$FROST_INT_DESC" \
    balance
```
Here's the result:
```
{
  "satoshi": {
    "confirmed": 1000000000,
    "immature": 0,
    "trusted_pending": 0,
    "untrusted_pending": 0
  }
}
```

The Bitcoin wallet controlled by your FROST group has 10 BTC! You're
down with the set up (at last!) and now ready for a threshold of FROST
members to create and sign a transaction!

## Step 7: Creating an Unsigned PSBT for the FROST Wallet

You're now ready to create a transaction sending some funds back from
the FROST wallet to the Bitcoin wallet, which will demonstrate using a
FROST group signature for a Bitcoin transaction.

To do so, you first must generate a new address for receipt on your
regular Bitcoin wallet:
```
REG_NEW_ADDRESS=$(bitcoin-cli -rpcuser=$RPC_USER \                   
                            -rpcpassword=$RPC_PASS \
                            -regtest \
                            getnewaddress)
```

> :information_source: **NOTE:** It is best practice to generate a new
address any time you receive digital funds.

You can now create a PSBT that sends 100,000,000 satoshis (1 BTC) from
your FROST wallet to that address on your regular Bitcoin wallet:

```
AMOUNT=100000000
UNSIGNED_PSBT=$(bdk-cli \
  --network "$NETWORK" \
  --datadir "$FROST_DIR" \
  wallet \
    --client-type rpc \
    --url "$RPC_URL" \
    -a "$RPC_AUTH" \
    --database-type sqlite \
    --ext-descriptor "$FROST_EXT_DESC" \
    --int-descriptor "$FROST_INT_DESC" \
    create_tx \
      --to ${REG_NEW_ADDRESS}:${AMOUNT} \
      --fee_rate 1.5 \
      --enable_rbf \
    | jq -r '.psbt')
echo "$UNSIGNED_PSBT" > frost-to-reg.psbt
```

The PSBT should look something like this:

```
cat frost-to-reg.psbt

cHNidP8BAH0CAAAAAU+GpYAYGylD7Aq8iCEbUe8tDj6Ubk9drEryZb+IUR/xAQAAAAD9////AnHopDUAAAAAIlEgFs9vu6vPBwD8EBesRxbdwciiciVtZq09kN40zOY7kYoA4fUFAAAAABYAFBViDlLjJ0fOX2EzxHimbQ9MArg2ZgAAAAABASsAypo7AAAAACJRICE+aOPQcevOnEkRFNy9we527T9Ssw8FtfJ9Nvb983yFIRZRBsC8uQO6A1h6+/LHabL87iyFGKw4P/CEmr1r9Gun6wUAuwK1jAEXIFEGwLy5A7oDWHr78sdpsvzuLIUYrDg/8ISavWv0a6frAAEFIFEGwLy5A7oDWHr78sdpsvzuLIUYrDg/8ISavWv0a6frAQYlAMAiIFEGwLy5A7oDWHr78sdpsvzuLIUYrDg/8ISavWv0a6frrCEHUQbAvLkDugNYevvyx2my/O4shRisOD/whJq9a/Rrp+slAWSQs+DhhYkKIfKdulcJ3Mk78IJyclunm7g4b6KrBgJduwK1jAAA
```

> :book: **What is a PSBT?** A PSBT is a Partially Signed Bitcoin
Transaction. The format was introduced in 2018 in Bitcoin Core 0.17
and is defined in [BIP 174](https://en.bitcoin.it/wiki/BIP_0174). A
PSBT divides up the creation of a transaction and the signing of the
transaction, allowing the transaction to be passed around while it's
unsigned or partially signed. This enables offline signatures and also
supports multisigs, such as FROST group signatures.

With a PSBT in hand, you now just need to sign it, which is the whole
point of this exercise.

## Step 8: Generating a Signature using the FROST Group

To sign a Bitcoin transaction, the first thing you have to do is
extract the sighash from the PSBT. This is done with the
`sighash-helper` tool that you built in §4.1:

```
MSG=$(sighash-helper < frost-to-reg.psbt)
```

> :book: **What is a Sighash?** When Bitcoin signs a transaction, it
actually signs a hash of the transaction: the sighash. Each
transaction has multiple inputs (where funds are coming from) and
multiple outputs (where funds are going to). Whether a sighash
contains all, some, or none of the inputs and all, some, or none of
the outputs is determined by a specific `sighash` flag, which is
applied to each signature.

You're now ready to use the ZF FROST `server` to sign your message,
using the same procedure as
[§3.2](03_2_Creating_FROST_Secret_Shares_with_DKG_Using_Server.md).

In short:

If you're already running the `frostd` server, kill it, to clear out
any previous sessions. Then, restart `frostd`
```
% frostd  --tls-cert localhost+3.pem --tls-key localhost+3-key.pem
```

Have Alice start the`coordinator` with a selected threshold of FROST
group members and the sighash as the message input (sent here from the
stdin, as shown with the `-m -` argument):

```
SIGNERS="$ALICE_PUBKEY,$BOB_PUBKEY"
printf '%s\n' "$MSG" | frost-client coordinator \
  --group "$GROUP_ID" \
  -s https://127.0.0.1:2744 \
  -S "$SIGNERS" \
  -m - \
  -o sig.raw \
  -c alice.toml
```

> :information_source: **NOTE:** Obviously, you're again using the
secp256k1-tr as a ciphersuite here, this time for signing. However,
when the signing shares are brought together in round 2 of the signing
process, you also take advantage of the other Blockchain Commons
update to the ZF FROST tools: they're aggregated with a Taproot tweak.

> :book: **What is a Taproot tweak?** A P2TR Bitcoin script combines
two elements: a locking script for a Schnorr signature and a tagged
hash of a Merkle tree which contains additional ways in which the
script may be unlocked. (This is the power of Taproot: traditional key
access and script access are combined in a single package.) The tagged
Merkle tree hash is simply added to the address thanks to the power of
Schnorr aggregation. This is a "tweak". To sign then requires that the
exact same tweak (e.g., the tagged Merkle tree hash) be added to one
of the signing shares, to make the signature valid for the tweaked
keys expected by Taproot. This happens as part of the signing process.

Then each signer (including Alice) must join as a participant.

Here's Alice doing so:
```
frost-client participant \
  --group "$GROUP_ID" \
  -s https://127.0.0.1:2744 \
  -c alice.toml
```

Here's Bob:
```
frost-client participant \
  --group "$GROUP_ID" \
  -s https://127.0.0.1:2744 \
  -c bob.toml
```

Each participant will see status updates and eventually be asked to sign:
```
Logging in...
Joining signing session...
Sending commitments to coordinator...
Waiting for coordinator to send signing package.................
Signing package received
Message to be signed (hex-encoded):
e24d7fad36af7e9e2fa2ecfaacc9e1cd3cadc19853ffc9bebf8787b19d02a305
Do you want to sign it? (y/n)
```
But here's a puzzle! How do they know they're signing the right thing,
as they can't backtrack a hash, which is what they're being asked to
sign!

The best practices methodology would be for Alice to send around the
PSBT prior to initiating the FROST signing session.

Ideally, each user could do a simple check with `bitcoin-cli`:
```
bitcoin-cli -rpcuser=$RPC_USER \
                            -rpcpassword=$RPC_PASS \
                            -regtest \
                            decodepsbt psbt=`cat frost-to-reg.psbt`
```

However, tests suggest that this doesn't work correctly at the time of
this writing. However, sites such as
[chainquery](https://chainquery.com/bitcoin-cli/decodepsbt) allow for
testing of the RPC in a more robust environment. A JSON of the entire
transaction should appear.

The users would want to consult the inputs and outputs to make sure
the right money is being sent to the right places. Afterward, they
could each run `sighash-helper` to verify the hash of the PSBT they
examined matched the hash they're signing. Then, they can give their
OK for signature with their FROST share.

(Yes, this is another place where the actual process would need to be
much improved to make it accessible for real-life usage. Developers
take note!)

## Step 9: Signing the PSBT using the FROST Group

You've got a signature, you've now just got to attach it to the PSBT
and send it out to the network.

First, you should grab the hex of the signature:
```
SIG_HEX=$(xxd -p sig.raw | tr -d '\n')
```
It should look something like this:
```
0842a58b3072175540b587b94763ef967db3c9ad48a7cecf6ff788595e049528638b08c1a7c07fbe025956e938bb92ce1e26bbcfacc78a68f49195381c27148a
```

Next, you need to attach the signature to the PSBT. This is the other
place where neither `bitcoin-cli` or `bdk-cli` had sufficient
functionality. That's because they both expect you to do signing with
their tool, where you instead signed with a FROST tool.

That's where the `psbt-sig-attach` helper comes into play. If you look
at the
[code](https://github.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/blob/main/04_1_Installing_Bitcoin_Tools.md#creating-the-psbt-sig-attach-helper-tool)
for the tool, you'll see that it attaches the signature to the 0th
UTXO (input) of your transaction and flags it with the `SIGHASH_ALL`
flag, meaning that the signature applies to all inputs and outputs.

Here's how it's used. You just pipe the PSBT file into the
`psbt-sig-attach` tool, and offer the signature as an argument.
```
SIGNED_PSBT=$(cat frost-to-reg.psbt | psbt-sig-attach "$SIG_HEX")
echo "$SIGNED_PSBT" > frost-to-reg.signed.psbt
```

You'll now have a PSBT that is somewhat longer because it includes a
signature. But, unlike the ECDSA signatures that were typically used
on Bitcoin, this Schnorr (FROST) signature doesn't grow in size with
the number of signers. This group signature, signed by both Alice and
Bob, is exactly the same size as a signature made by one person!  And
so the PSBT file is the same size as well! (That's one of the
advantages of FROST).

```
cHNidP8BAH0CAAAAAU+GpYAYGylD7Aq8iCEbUe8tDj6Ubk9drEryZb+IUR/xAQAAAAD9////AnHopDUAAAAAIlEgFs9vu6vPBwD8EBesRxbdwciiciVtZq09kN40zOY7kYoA4fUFAAAAABYAFBViDlLjJ0fOX2EzxHimbQ9MArg2ZgAAAAABASsAypo7AAAAACJRICE+aOPQcevOnEkRFNy9we527T9Ssw8FtfJ9Nvb983yFARNACEKlizByF1VAtYe5R2Pvln2zya1Ip87Pb/eIWV4ElShjiwjBp8B/vgJZVuk4u5LOHia7z6zHimj0kZU4HCcUiiEWUQbAvLkDugNYevvyx2my/O4shRisOD/whJq9a/Rrp+sFALsCtYwBFyBRBsC8uQO6A1h6+/LHabL87iyFGKw4P/CEmr1r9Gun6wABBSBRBsC8uQO6A1h6+/LHabL87iyFGKw4P/CEmr1r9Gun6wEGJQDAIiBRBsC8uQO6A1h6+/LHabL87iyFGKw4P/CEmr1r9Gun66whB1EGwLy5A7oDWHr78sdpsvzuLIUYrDg/8ISavWv0a6frJQFkkLPg4YWJCiHynbpXCdzJO/CCcnJbp5u4OG+iqwYCXbsCtYwAAA==
```

## Step 10: Finalizing and Sending the PSBT

You can now ask `bdk-cli` to close out the PSBT and prepare it for transmission:

```
FINAL_PSBT=$(bdk-cli \
  --network "$NETWORK" \
  --datadir "$FROST_DIR" \
  wallet \
    --client-type rpc \
    --url "$RPC_URL" \
    -a "$RPC_AUTH" \
    --database-type sqlite \
    --ext-descriptor "$FROST_EXT_DESC" \
    --int-descriptor "$FROST_INT_DESC" \
    finalize_psbt $(< frost-to-reg.signed.psbt) \
  | jq -r '.psbt')
```
You can then broadcast it as a transmission:
```
TXID=$(bdk-cli \
  --network "$NETWORK" \
  --datadir "$FROST_DIR" \
  wallet \
    --client-type rpc \
    --url "$RPC_URL" \
    -a "$RPC_AUTH" \
    --database-type sqlite \
    --ext-descriptor "$FROST_EXT_DESC" \
    --int-descriptor "$FROST_INT_DESC" \
    broadcast \
      --psbt "$FINAL_PSBT" | \
    jq -r '.txid')
```
Since you're on regtest, you'll then need to advance a block:
```
bitcoin-cli -rpcuser=$RPC_USER \
	    -rpcpassword=$RPC_PASS \
            -regtest \
            -generate 1
```
Afterward, you can verify your balances.

Here's the Bitcoin wallet:
```
bitcoin-cli -rpcuser=$RPC_USER -rpcpassword=$RPC_PASS -regtest getbalance

140.99999835
```
That's 89.99999835 + 50 (new coinbase award for new block) +1 (transfer).

You can similarly sync and check the balance of your FROST wallet with
`bdk-cli`.

```
bdk-cli \
  --network "$NETWORK" \
  --datadir "$FROST_DIR" \
  wallet \
    --client-type rpc \
    --database-type sqlite \
    --url "$RPC_URL" \
    -a "$RPC_AUTH" \
    --ext-descriptor "$FROST_EXT_DESC" \
    --int-descriptor "$FROST_INT_DESC" \
    sync

bdk-cli \
  --network "$NETWORK" \
  --datadir "$FROST_DIR" \
  wallet \
    --client-type rpc \
    --database-type sqlite \
    --url "$RPC_URL" \
    -a "$RPC_AUTH" \
    --ext-descriptor "$FROST_EXT_DESC" \
    --int-descriptor "$FROST_INT_DESC" \
    balance
```
The result is:
```
{
  "satoshi": {
    "confirmed": 899999857,
    "immature": 0,
    "trusted_pending": 0,
    "untrusted_pending": 0
  }
}
```
That's the 10 BTC minus the 1 BTC returned and the transaction fee.

Congrats, you've proven a use case for FROST signing by signing a
Bitcoin transaction! (Now get out there and design an easier way to
do so!)

## Summary: Signing PSBTs with FROST

Though it took ten steps here, the first six steps of this demo were
just setting things up so that you had a regular Bitcoin wallet and a
FROST Bitcoin wallet and the FROST Bitcoin wallet had funds.

Fundamentally, all that's required to sign a PSBT with FROST is:

1. Create an unsigned PSBT for the transaction.
2. Extract the `sighash` from the PSBT.
3. Use FROST Tools to sign the `sighash` for the transaction.
4. Insert the signature into the transaction.
5. Finalize & send the transaction.

## What's Next

That's it! We're considering expanding this tutorial in the future,
per our [TODO](TODO.md), as Blockchain Commons has technologies such
as [Gordian Clubs](https://developer.blockchaincommons.com/clubs/) and
[Gordian Envelope](https://developer.blockchaincommons.com/envelope/)
that provide other use cases for FROST, as well as tools such as
[Hubert](https://developer.blockchaincommons.com/hubert/) that can
support FROST.

But for the moment, you've got all the fundamentals of FROST, and
should be ready to begin your own design.

If you have questions or disagreements, please feel free to file
[Issues](https://github.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/issues)
and if you have corrections, particularly simple things like typos,
please file
[PRs](https://github.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/pulls).
