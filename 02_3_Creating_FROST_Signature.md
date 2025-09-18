# 2.3: Creating a FROST Signature

With their signing shares in hand, members are now ready to sign documents.

## Choosing What to Sign

Obviously, signers need something to sign. In [Chapter
3](03_0_FROST_and_Bitcoin.md), this will be a Bitcoin
transaction. Signing transfers of digital assets are a common use case
for FROST signatures.

However, you can also sign _anything_ that requires collaborative
authorization of the secret share holders. That might frequently
include the validation of decisions made by a board of directors or
other organization:

```
% cat > board-meeting-250917.txt
Date: 9/17/25
Present: 0100000000000000000000000000000000000000000000000000000000000000, 0200000000000000000000000000000000000000000000000000000000000000

Decisions:
* Allocate $10 to Jam in Central Park Party
* All Members Must Contribute $5 to Group Kitty by 9/24/25
```

## Signing with a Coordinator

FROST signing usually requires a coordinator. In the case of the ZF
FROST Tools, that's the `coordinator` app. It also needs a way for the
group members to connect with the `coordinator`. In the ZF FROST
Tools, that's done with the `participant` app. Both should have
been installed when you ran the `cargo install`, as they're part of
the `frost-client` package.

The `coordinator` app may be run with the following flags as options:

| Flag | Description | Default | Options |
| ----- | --------- | --------- | --- |
| -C | Ciphersuite | ed25519 | redpallas |
| --cli | Interactive | <yes> |
| -i | IP Address | 0.0.0.0 |
| -m | Message | <filename> |
| -n | Participants | 
| -P | Public Key Filename | public-key-package.json |
| -p | Port | 443 |
| -r | Randomizers | <random> |
| -S | Signers |
| -s | Signature Output File | 

> :warning: **WARNING:** In our testing, the
`participant`/`coordinator` connection was not always reliable. This
seems to be even more the case when the `coordinator` is run with the
HTTP options (allowing the setting of a non-localhost IP address). As
a result, all examples are run on localhost, even though that's not
the best real-world option. Even with that setup, the `participant`
program sometimes did not deliver commitments when initially run, in
which case `participant` was shut down (^C) and restarted.

The following command will start `coordinator` on `localhost:443`. It
will require two signatures to sign `board-meeting-250917.txt` and
place the signature in `board-meeting-250917.sig`:

```
% coordinator -m board-meeting-250917.txt -s board-meeting-250917.sig -n 2
```

### Round 1: Commitments

Each of the members of the FROST group who will be signing can then
start the `participant` app using the `-k` flag to designate their
`key-package`:
```
% participant -k key-package-1.json
Reading key package from key-package-1.json
Connected to server at [1.R.0] 127.0.0.1:443
```

This will connect them to the server and send a commitment, which is
round 1 of the FROST signing protocol (and the round that can be
pre-computed, but is not here).

When the commitment is correctly sent, the following will show up on
the `coordinator`:
```
Client connected
Received: {"IdentifiedCommitments":{"identifier":"0100000000000000000000000000000000000000000000000000000000000000","commitments":{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"hiding":"0ea60b77708bb6157f206e107db8cd85a0fbb2fa02e5b56ad178b6349895d383","binding":"21b2994fa1e3034f65c6dc167369f69a9fc41e93cc7f70a5b417fccb82faabc7"}}}

```

> :warning: **WARNING:** If the `participant` did not correct send the
commitment to the `coordinator`, you'll see the "Client Connected"
message, but not "Received". In this caes, you should break out (^C)
and restart `participant` for that user.

```
Client connected
Client disconnected
```
Additional members may connect until you've met your threshold:
```
% participant -k key-package-2.json

Received: {"IdentifiedCommitments":{"identifier":"0200000000000000000000000000000000000000000000000000000000000000","commitments":{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"hiding":"d9525e2bef65a4ed45437ecefeefa7b5054199474e2089748bfbf921c150e9d0","binding":"106a9587b07faec2cca0c2f060644ef0f8f8b8d4f08e6c5afdc414cdfe44286a"}}}

```

### Round 2: Signing

At this point, the `coordinator` will have received all the
commtiments and will initiate round 2 of the FROST protocol by sending
the message and commitments to the members, each of whom can choose to
sign:

```
Sending SigningPackage to participants...
Waiting for participants to send their SignatureShares...
```
Each group member will see something like the following:
```
Received: {"SigningPackageAndRandomizer":{"signing_package":{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"signing_commitments":{"0100000000000000000000000000000000000000000000000000000000000000":{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"hiding":"0ea60b77708bb6157f206e107db8cd85a0fbb2fa02e5b56ad178b6349895d383","binding":"21b2994fa1e3034f65c6dc167369f69a9fc41e93cc7f70a5b417fccb82faabc7"},"0200000000000000000000000000000000000000000000000000000000000000":{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"hiding":"d9525e2bef65a4ed45437ecefeefa7b5054199474e2089748bfbf921c150e9d0","binding":"106a9587b07faec2cca0c2f060644ef0f8f8b8d4f08e6c5afdc414cdfe44286a"}},"message":"446174653a20392f31372f32350a50726573656e743a20303130303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030302c20303230303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030300a0a4465636973696f6e733a0a2a20416c6c6f636174652024313020746f204a616d20696e2043656e7472616c205061726b2050617274790a2a20416c6c204d656d62657273204d75737420436f6e7472696275746520243520746f2047726f7570204b6974747920627920392f32342f32350a"},"randomizer":null}}
Message to be signed (hex-encoded):
446174653a20392f31372f32350a50726573656e743a20303130303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030302c20303230303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030300a0a4465636973696f6e733a0a2a20416c6c6f636174652024313020746f204a616d20696e2043656e7472616c205061726b2050617274790a2a20416c6c204d656d62657273204d75737420436f6e7472696275746520243520746f2047726f7570204b6974747920627920392f32342f32350a
```

### Checking Your Message

But that message (displayed in "Message to be signed (hex-encoded):")
doesn't look anything like the text file of your minutes! That's
because it's been converted to hex, which is typical for digital
signing.

You can (and should!) check the message before you sign it. This is
done with `xxd -r -p`, which will convert hex to ASCII.

```
% echo "446174653a20392f31372f32350a50726573656e743a20303130303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030302c20303230303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030300a0a4465636973696f6e733a0a2a20416c6c6f636174652024313020746f204a616d20696e2043656e7472616c205061726b2050617274790a2a20416c6c204d656d62657273204d75737420436f6e7472696275746520243520746f2047726f7570204b6974747920627920392f32342f32350a" | xxd -r -p

Date: 9/17/25
Present: 0100000000000000000000000000000000000000000000000000000000000000, 0200000000000000000000000000000000000000000000000000000000000000

Decisions:
* Allocate $10 to Jam in Central Park Party
* All Members Must Contribute $5 to Group Kitty by 9/24/25
```

Sure enough, that's the original message. Once each of the
participating signers has verified the message, they can sign.

### Finalizing the Signature

When the participants agree to sign, they send their signing shares to
the `coordinator`. When the `coordinator` has received enough, it
will aggregate a signature for the message file:
```
Waiting for participants to send their SignatureShares...
Received: {"SignatureShare":{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"share":"6682858a180be6f0527a6733d33c5c957a9c2e6ff91345bfbf15f53ff4d8d608"}}
Received: {"SignatureShare":{"header":{"version":0,"ciphersuite":"FROST-ED25519-SHA512-v1"},"share":"fb5f07e9722cebf1c517a4c5263937a1998056f4ad79bf373825667571938107"}}
Client disconnected
Client disconnected
Raw signature written to board-meeting-250917.sig
```

## Ensuring the Division

Though the above examples demonstrate signing occurring on a single
host, the signing shares that make up a threshold should be divided by
this point. One of the major points of using FROST (but generally, of
using any multisignature or threshold signature algorithm) is to
protect your authorization by dividing it up.

In this example, the two group members could be signing on different
machines, which means that their actual key is never rquired to exist
in a single place!

By dividing authorization in this way, you ensure there is no Single
Point of Failure (SPOF), which could lead to the loss of the
authorization, and no Single Point of Compromise (SPOC), which could
lead to the theft of the authorization.

## Summary: Creating a FROST Signature

Creating a FROST signature is a simple application of the protocol
described in [§1.2: The FROST Signature
Process](01_2_FROST_Signature_Process.md). A coordinator organizes
things; each participant sends commitments; the coordinator resends
the commitments and the material to be signed; the participants use
their signing shares; and the coordinator aggregates them to creat the
final signature.

## What's Next

For the next step after "Signing with FROST", see [§2.4:
Checking a FROST Signature](02_4_Checking_FROST_Signature.md).
