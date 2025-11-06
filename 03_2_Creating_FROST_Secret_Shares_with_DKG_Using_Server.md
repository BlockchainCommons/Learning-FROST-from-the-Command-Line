# 3.2: Creating FROST Secret Shares with DKG Using Server

The ZF FROST tools offers two ways to create DKG shares. The first
way, demoed in
[§3.1](03_1_Creating_FROST_Secret_Shares_with_DKG_Using_CLI.md) forces
users to hand-input all the communications being exchanged, which is a
pragmatic UI (and a nice bit of hands-on experience to show you how it
all works), but is prone to error.

Fortunately, the ZF FROST tools have a better answer: a server-based
methodology.

## Preparing the Server

This DKG demo manages communication through a server, `frostd`, which
must also have a certificate. Communication connections are
authenticated, and communication is encrypted, providing a level of
security that's appropriate for the use of the higher security profile
of DKG. But the most important element may be the improved UI: the
server takes care of things so that the user doesn't have to, largely
elminating the opportunity for errors.

### Installing the Server

`frostd` is the other major software available in the `frost-tools`
repo. You can install it by changing back to the `frost-tools`
directory that you cloned from GitHub and running `cargo install` for
`frostd`:

```
% cargo install --path frostd
```

:information_source: **NOTE:** If you no longer have your
`frost-tools` repo (or have lost track of it), refer to [§2.1:
Installing the FROST Tools](02_1_Installing_FROST_Tools.md) for the
installation instructions.

### Installing a Certificate

The [`mkcert` app](https://github.com/FiloSottile/mkcert) allows you
to create test certificates on your machine. Look at the README in
that repo for full installation instructions, but most simply you can:
`brew install mkcert` (Mac); `sudo apt install libnss3-tools` (Linux);
or `choco install mkcert` (Windows). If you use a different package
manager, there are probably instructions on what to do at the
[`mkcert` repo](https://github.com/FiloSottile/mkcert).

After installing the `mkcert` software, you can create a testing
certificate for your `localhost` machine with the following commands:

```
% mkcert -install
```

On a Mac, this will require you to enter your password for `sudo` and
to update your system settings. Other systems will require slightly
different verifications.

You may also get some status reports including some errors like this,
which are fine because you're not creating this certificate for
Firefox:
```
Warning: "certutil" is not available, so the CA can't be automatically installed in Firefox! ⚠️
```

You can now create a certificate for your local machine:

```
% mkcert localhost 0.0.0.0 127.0.0.1 ::1

Created a new certificate valid for the following names 📜
 - "localhost"
 - "0.0.0.0"
 - "127.0.0.1"
 - "::1"
```
You'll also be told where your certificates are, which should be something like::
```
The certificate is at "./localhost+3.pem" and the key at "./localhost+3-key.pem" ✅
```

(The precise filename will vary if you create for a different machine
name and/or with a different number of aliases.)

### Starting the Server

You can now start the `frostd` server with that certificate:
```
% frostd  --tls-cert localhost+3.pem --tls-key localhost+3-key.pem
```
It will run in the foreground and report out messages to you:
```
2025-09-19T00:55:10.092611Z  INFO frostd: server running
2025-09-19T00:55:10.093142Z  INFO frostd: starting HTTPS server at 0.0.0.0:2744
```

## Preparing the Users

Now, the users need to be able to talk to each other _securely_ via
that server.

### Create User Credentials

To communicate sercurely, each user will need to create credentials,
which can be done with the `frost-client` utility:

```
% frost-client init -c alice.toml
% frost-client init -c bob.toml
% frost-client init -c eve.toml
```
Usually, each user would do this on their own machine, but again for
simplicity we're doing everything on a single machine.

Here's what the `alice.toml`
[configuration](https://en.wikipedia.org/wiki/TOML) files looks like:

```
version = 0

[communication_key]
privkey = "04692aef6190b43d9524d719e0a7f45689ea3265a36fe34e69c625a9b8ac2967"
pubkey = "00d639b42342ec83031c0cfb0462cb759832fb4d7b7c5942edd3c067b465406f"

[contact]

[group]
```

It contains a communication key, which is the last crucial element of
the client-server setup, because it allows the members who will form
the FROST group to authenticate each other and to encrypt
communications, using the keypair.

### Installing tomlq

In order to easily read those config files from the command line,
you'll need a tool called `tq`, which is a Rust Tool that can be
installed from the [tomlq repo](https://github.com/cryptaliagy/tomlq).

```
% git clone https://github.com/cryptaliagy/tomlq.git
%  cd tomlq
%  cargo install --path .
```

### Extracting the Contact Info

Now you can use those TOML files, `tq`, and the `frost-client` to
record the public key and the ZF FROST contact string for each user:

```
ALICE_PUBKEY=$(tq -f alice.toml -r 'communication_key.pubkey')
ALICE_CONTACT=$(frost-client export --name "Alice" -c alice.toml 2>&1 | tail -1)
BOB_PUBKEY=$(tq -f bob.toml -r 'communication_key.pubkey')
BOB_CONTACT=$(frost-client export --name "Bob" -c bob.toml 2>&1 | tail -1)
EVE_PUBKEY=$(tq -f eve.toml -r 'communication_key.pubkey') 
EVE_CONTACT=$(frost-client export --name "Eve" -c eve.toml 2>&1 | tail -1)
```

The `PUBKEY` just contains the public key in the TOML file, while the
contact string encodes that to allow it to be passed to another `frost-client`.
Here's what they look like.
```
% echo $ALICE_PUBKEY
00d639b42342ec83031c0cfb0462cb759832fb4d7b7c5942edd3c067b465406f
% echo $ALICE_CONTACT
zffrost1qyqq2stvd93k2gqq6cumgg6zajpsx8qvlvzx9jm4nqe0kntm03v59mwncpnmge2qdu8ya0ws
```

### Exchanging the Contact Information

At this point, in the real world, each user would have collected their
own `PUBKEY` and `CONTACT`. Now, they would need to send these two
long strings of data to each of the other users who will be
participating in the FROST group. They could send it over a secure
messaging system such as Signal, but it'd be better to meet in person
so that you can verify that the information belongs to a known person.

(In this example, we're simply assuming the strings we've stored in
variables have gotten to the other users.)

### Importing the Contact Information

The `frost-client` will directly digest the contact info.

Each user needs to collect the information from each of the other
users. So Alice would `import` the following into her `toml` file.:

```
frost-client import -c alice.toml $BOB_CONTACT
frost-client import -c alice.toml $EVE_CONTACT
```

These commands will show the following results:
```
Imported this contact:
Name: Bob
Public Key: 2f232f8a6f69d7dac39cbadff58fc753066bed7ec7d2401acb838e1b033c2305
```
And:
```
Imported this contact:
Name: Eve
Public Key: f2f2f9344e203c1279e6997d8f9030d7727d36a2ba053a1ec3b1f3dae266a26d
```
Afterward, `alice.toml` should be updated accordingly:
```
version = 0

[communication_key]
privkey = "04692aef6190b43d9524d719e0a7f45689ea3265a36fe34e69c625a9b8ac2967"
pubkey = "00d639b42342ec83031c0cfb0462cb759832fb4d7b7c5942edd3c067b465406f"

[contact.Bob]
name = "Bob"
pubkey = "2f232f8a6f69d7dac39cbadff58fc753066bed7ec7d2401acb838e1b033c2305"

[contact.Eve]
name = "Eve"
pubkey = "f2f2f9344e203c1279e6997d8f9030d7727d36a2ba053a1ec3b1f3dae266a26d"

[group]
```

As shown, the ZF FROST contact info has been converted back to a
public key. As long as Alice trusted the method of distribution for
the contact info (and the source of that information) they now will be
able to trust information from Bob or from Eve that is signed with the
corresponding private key.

Bob does the same:

```
frost-client import -c bob.toml $ALICE_CONTACT
frost-client import -c bob.toml $EVE_CONTACT
```

As does Eve:
```
frost-client import -c eve.toml $ALICE_CONTACT
frost-client import -c eve.toml $BOB_CONTACT
```

Besides importing the `CONTACT` into their `toml` file, each user
would also set a `PUBKEY` variable for each other user, as doing so
will make it easier to contact the server, momentarily.

And now we've got a server and enough information to get started
creating the DKG!

## Creating DKG Shares

You're now ready to create shares using ZF FROST's server-based DKG
method.


You should already have `frostd` running. Each user will now run
`frost-client` on their own terminal to connect to that server. The
following command-line options are used when running `frost-client`

| Flag | Description | Default | Options |
| ----- | --------- | --------- | --- |
| \-d | Description | | |
| \-s | Server URL | | |
| \-S | Public Keys | | |
| \-t | Threshold | | |
| \-C | Cipher Suite | ed25519 | |
| \-c | TOML File | | |

### Connecting to the Server

The following commands will initiate the DKG Protocol. They should be
used by the _first_ user to connect.

Alice runs:
```
% frost-client dkg -d "DKG: Alice, Bob, Eve" \
-s 127.0.0.1:2744 \
-S $BOB_PUBKEY,$EVE_PUBKEY \
-t 2 \
-c alice.toml
```

Note that:

* The `-d` description is just a label put into the individual's TOML file.
* The `-s` server name and port number will have appeared when the server started, but it should usually be `127.0.0.1:2744`.
* The `-S` public keys are why we saved those keys in `$PUBKEY` variables, but you could just as easily have entered them by hand. The first user to initiate the `dkg` connection enters the keys of the other two. They won't need to do so at all (and in fact, doing so caused the DKG to fail in our tests, which clearly shouldn't be the case if all the keys match).
* The `-t` is the threshold, in this case 2 out of the three pubkeys (the user's own and the ones entered as `-S`). If the number of pubkeys and/or the threshold don't match, things won't work! Everyone will just wait around, because there aren't enough participants to create the requested DKG.
* The `-c` is the config file for each individual user. This one is Alice's.

Bob's command will then look like this:
```
% frost-client dkg -d "DKG: Alice, Bob, Eve" \
-s 127.0.0.1:2744 \
-t 2 \
-c bob.toml
```
Eve's will look like this:
```
% frost-client dkg -d "DKG: Alice, Bob, Eve" \
-s 127.0.0.1:2744 \
-t 2 \
-c eve.toml
```
Note that both omit the `-S` flag; not doing so might cause the DKG to hang.

Each user should see the following:
```
Logging in...
Creating DKG session...
Getting session info...
Waiting for other participants to send their Round 1 Packages............
```

Things will pause here while everyone connects ... and could get stuck
here if there's a problem such as a discrepency between thresholds or
public keys. (This is also where things got stuck when we tried to use
the `-S` flag for non-initiating users.)

But hopefully, everyone soon sees the following:
```
Waiting for other participants to send their broadcasted Round 1 Packages.....
Waiting for other participants to send their Round 2 Packages....
Group created; information written to alice.toml
```
That's right, it's all done automatically from the initial connection,
as long as each user set up their initial comunication keys correctly!

Here's a look at Alice's TOML after the process:
```
version = 0

[communication_key]
privkey = "cb3d54011ad8c05d714ff2bb3b0466917723624ba89fd28b8df7c0a1b56b2e1a"
pubkey = "492dd8fced4ec58490800ec6216ca3fff0ed7148fad20cd7a9da52e62d6ddf26"

[contact.Bob]
name = "Bob"
pubkey = "6ba29fa51f76f6de6e99ce98767e56e9716fddb39c27bdf48d8d5458e22c8f77"

[contact.Eve]
name = "Eve"
pubkey = "dcffe972745b7f9a9d44f422c4b60de3ffc18d913bbe22d17627f864ffd34f15"

[group.7de1487fe264ab9ae292fcbce0e5dc1bb5524c810beae7119de74c99aa313e6c]
description = "DKG: Alice, Bob, Eve"
ciphersuite = "FROST-ED25519-SHA512-v1"
public_key_package = "00b169f0da03577250dcbe4c4589fb88caa29634135be4a78b49f626d890a354cf4354859f050d69dc7af725101636926792b6ecc41a7e24dbe5ad92bc88e61619bfb433cb84b8a48843f96a8f5d9c0b19e24c8caea907a91b526c6a60da8797453d4b2c240df46c0a8e1dab98621d1dbbccb5f091b30cf469f77c035181b80580bb595a4ad382e52235a65875816d0fa17e771db3711a7c4c8afba36f5febcf1ef68e48af0fa46286abe1e5e1e8b4740752420da1eedf69b83a34e3b4f60bc7b8b1f9a02c757de1487fe264ab9ae292fcbce0e5dc1bb5524c810beae7119de74c99aa313e6c"
key_package = "00b169f0da577250dcbe4c4589fb88caa29634135be4a78b49f626d890a354cf4354859f05ddb0f4708376b99de06d4a086c6c6371d7992102201ec39e6426555fa46abe040d69dc7af725101636926792b6ecc41a7e24dbe5ad92bc88e61619bfb433cb847de1487fe264ab9ae292fcbce0e5dc1bb5524c810beae7119de74c99aa313e6c02"
server_url = "127.0.0.1:2744"

[group.7de1487fe264ab9ae292fcbce0e5dc1bb5524c810beae7119de74c99aa313e6c.participant.577250dcbe4c4589fb88caa29634135be4a78b49f626d890a354cf4354859f05]
identifier = "577250dcbe4c4589fb88caa29634135be4a78b49f626d890a354cf4354859f05"
pubkey = "492dd8fced4ec58490800ec6216ca3fff0ed7148fad20cd7a9da52e62d6ddf26"

[group.7de1487fe264ab9ae292fcbce0e5dc1bb5524c810beae7119de74c99aa313e6c.participant.82e52235a65875816d0fa17e771db3711a7c4c8afba36f5febcf1ef68e48af0f]
identifier = "82e52235a65875816d0fa17e771db3711a7c4c8afba36f5febcf1ef68e48af0f"
pubkey = "dcffe972745b7f9a9d44f422c4b60de3ffc18d913bbe22d17627f864ffd34f15"

[group.7de1487fe264ab9ae292fcbce0e5dc1bb5524c810beae7119de74c99aa313e6c.participant.b8a48843f96a8f5d9c0b19e24c8caea907a91b526c6a60da8797453d4b2c240d]
identifier = "b8a48843f96a8f5d9c0b19e24c8caea907a91b526c6a60da8797453d4b2c240d"
pubkey = "6ba29fa51f76f6de6e99ce98767e56e9716fddb39c27bdf48d8d5458e22c8f77"
```
It now contains the FROST group of Alice, Bob, and Eve, along with
information on each participant.

This is very much the sort of UI you want for DKG creation:
automated.

1. The users have private and public keys that are automatically generated.
2. The DKG ceremony happens through a server that uses those keys to secure communication.
3. All the user has to do is intiate a connection.
4. Both rounds of communication happen in the background, then the DKG key shares are automatically stored.

## Signing with DKG Shares Generated by the Server

In order to sign with your new DKG shares, you must still be running
`frostd`, and then you must run `frost-client coordinator` to connect
to the server and manage the signing session.

`frost-client coordinator` can be run with the following flags:

| Flag | Description | Default | Options |
| ----- | --------- | --------- | --- |
| -c | Credentials | credentials.toml |
| -g | Group ID | <#> |
| -m | Message | <filename> |
| -o | Signature Output File |
| -r | Randomizers |
| -s | Server URL | <url> |
| -S | Signers | <public keys> |

:warning: **WARNING:** Note that `frost-client coordinator` is a
different program from the `coordinator` that you previously ran. That
was a standalone server, this is one that is run through `frostd`. As
a result, some of the command-line options are different (and one even
reuses the same argument for a different purpose, so be careful!).
 
Everyone will need to know the Group ID of this FROST group to
communicate. This appears throughout the credentials files in lines
like this:
```
[group.f6b2f30a4f3a666110478eca120a9a7d19740390fff8bb2250eb274177e6dd22.participant.2cb07edbc49dbcac5aa169c6f0c417e3a170431db15265ffc90e9f161b87c108]
```

Alice can capture it as follows:
```
% GROUP_ID=$(grep -oE '^\[group\.[0-9a-f]+' alice.toml | head -1 | sed 's/^\[group\.//')
```

Alice can now start up the `coordinator` as follows.
```
frost-client coordinator --group $GROUP_ID -s 127.0.0.1:2744 -S "$ALICE_PUBKEY,$BOB_PUBKEY" -m board-meeting-250917.txt -o board-meeting-250917.sig -c alice.toml
```
She is supplying her credentials file, so that the `frost-client
coordinator` can look up the group, and she is picking the two people
who will sign: herself and Bob. Eve signing this transaction will not
result in it being valid!

Alice should see these results:
```
Reading message from board-meeting-250917.txt...
Logging in...
Creating signing session...
Waiting for participants to send their commitments......
```

Now each of Alice and Bob need to run `frost-client participant` to do
the actual signing. Yes, this does mean that Alice is running two
programs, `frost-client coordinator` and the `frost-client
participant`!

This is a simple command that just requires the Group ID, the server URL, and the credentials file:
```
GROUP_ID=$(grep -oE '^\[group\.[0-9a-f]+' alice.toml | head -1 | sed 's/^\[group\.//')
frost-client participant -g $GROUP_ID -s 127.0.1:2744 -c alice.toml
```

Bob does the same:
```
GROUP_ID=$(grep -oE '^\[group\.[0-9a-f]+' bob.toml | head -1 | sed 's/^\[group\.//')
frost-client participant -g $GROUP_ID -s 127.0.0.1:2744 -c bob.toml
```

:warning: **WARNING:** Again `participant` and `frost-client
participant` are different programs.

They should each see:
```
Logging in...
Joining signing session...
Sending commitments to coordinator...
Waiting for coordinator to send signing package...........
Message to be signed (hex-encoded):
446174653a20392f31372f32350a50726573656e743a20303130303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030302c20303230303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030303030300a0a4465636973696f6e733a0a2a20416c6c6f636174652024313020746f204a616d20696e2043656e7472616c205061726b2050617274790a2a20416c6c204d656d62657273204d75737420436f6e7472696275746520243520746f2047726f7570204b6974747920627920392f32342f32350a
Do you want to sign it? (y/n)
```

Much as when [signing after
TDG](https://github.com/BlockchainCommons/Learning-FROST-from-the-Command-Line/blob/main/02_3_Creating_FROST_Signature.md#checking-your-message),
that hex-encoding should be checked so that you know what you're
signing. That can be done with `echo "<hex>" | xxd -r -p`.

After they sign, the participants should see:
```
Sending signature share to coordinator...
Done
```
Then the coordinator should see:
```
Raw signature written to board-meeting-250917.sig
```
That's it. Due to the ease of use of the `frost-client` and `frostd`,
the signing is done already!

## Verifying with DKG Shares Generated by the Server

You can use the `frost-verify` program that you downloaded in
[§2.4](02_4_Checking_FROST_Signature.md) to check your new
signature. You just need to account for the fact that your public key
package is now stored in your credentials file (e.g., `alice.toml`)
rather than a separate public key package file. To support this, you
use the `-C` flag and list your credentials file rather than the `-P`
flag that you used previously.

```
% frost-verify verify -m board-meeting-250917.txt -s board-meeting-250917.sig -C alice.toml
✅ Signature verification: PASSED
```

## Accessing the Public Key Package Programmatically

If you look at your credentials file, you'll see that the
`public_key_package` is recorded by the ZF FROST server in hex, not an
easy-to-read JSON file, as was the case when you used the previous CLI
tools:

``` public_key_package =
"00b169f0da032cb07edbc49dbcac5aa169c6f0c417e3a170431db15265ffc90e9f161b87c10893b643f9c6aeb52b094407bc3be61faa261d304ec6db6337ad05811f4d6aa482aab71ca92218f843a1f673f1428737de78512df0efc9e1108e3a8624cd05b90c3415e1c341ad3ca3de173b4e274b2cb2551a3f2ffef8824892d8fabdfcf786f637087bae60ed73e128a3d6c383491e31403bddf40ef33b4531b2e1cb1a76310db7c3534af91f5e02880f8b38d8a11d77345d2219481292fc5a5ee03d8edd0a6bf6b2f30a4f3a666110478eca120a9a7d19740390fff8bb2250eb274177e6dd22"
``` This is a hex encoding of a Rust vector. If you decide that you
want to work with this file programmatically, you can decode it using
the [Serde framework](https://docs.rs/serde/latest/serde/) for Rust,
but that goes beyond the scope of this tutorial.

## Summary: Creating FROST Secret Shares with DKG Using Server

Creating shares with DKG, but using the ZF FROST server, demonstrates
how easy FROST can be (once you get the server set up!). It's a lesson
in accessibility for FROST developers.

## What's Next

To review the share creation methods demonstrated in §2.2, §3.1, and
§3.2 as well as the signing methods demonstrated in §2.3, §3.1, and
§3.3, continue with [§3.3: ZF FROST in a
Nutshell](03_3_ZF_FROST_in_a_Nutshell.md).

Or if you're comfortable with that already, jump to another FROST
advanced feature in [§3.4: Refreshing FROST
Shares](03_4_Refreshing_FROST_Shares.md).