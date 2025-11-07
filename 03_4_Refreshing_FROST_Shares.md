# 3.4: Resharing Shares

One of the notable capabilities of FROST is the ability to refresh (or
repair) shares. This functionality is not available for demo in the ZF
FROST command-line tools, but developers should nonetheless be aware
of the capabilities,

## Resharing Options

Because of the unique advantages of Schnorr (and its aggregateable
signatures), FROST's signature shares can be recalculated in a variety
of ways without changing the public key for a group.

This can generally be described as **repairing** or **refreshing** the
FROST shares. These processes require at least a threshold of
participants (and sometimes all participants) and do not reveal any
secrets to those participants (other than the ones they'd normally
receive).

* **Repair.** Restore lost shares.
* **Refresh.** Update shares.

These features allow the FROST group to be changed in a number of ways:

* **Enroll.** A new member can be added to a FROST group.
* **Disenroll.** An old member can be excluded from a FROST group.
* **Change Threshold.** A threshold can be increased or decreased.

In other words, a FROST quorum can be changed in any way after it's
created!

These features are still fairly cutting edge. They have been deployed
in some libraries and some apps, but should be considered experimental
until fully security reviewed.

## Security Ramifications

FROST refreshes do _not_ revoke previous signing shares! In a trusted
system, the threshold of signers who are refreshing the shares will
afterward delete their old shares, so that the older set of shares is
no longer usable. But, this is not a requirement!

For example, Alice and Bob might update their 2-of-3 threshold shares,
disenrolling Eve and enrolling Carol. They should at this point delete
their old signing shares. In this case, Eve still has her signing
share, and it's still valid (they always remain valid!), but Alice and
Bobs' old shares are now gone, so it's impossible for Eve to achieve
threshold. But if Bob acts in bad faith and does not delete his share,
he can now conspire with Eve, because both of their old shares, prior
to the refresh, remain usable.

In short: the threshold of signers resharing for a FROST group must
act in good faith or else the prior shares will remain usable.

## Refreshing Shares Programmatically

Though resharing is not currently available through the ZF FROST
command-line tools, you can refresh a FROST group with the ZF FROST
API using [`compute_rereshing_shares()` and
`refresh_share()`](https://frost.zfnd.org/tutorial/refreshing-shares.html).

## Summary: Refreshing Shares

One of the advantages of FROST is that you can reshare after the fact,
refreshesing, repairing, enrolling, disenrolling, or changing the
threshold of your group. This creates strong resilience (because you
can update a group if a share or even a member goes missing) and also
creates strong flexibility (because you can change a group over time).

## What's Next

[Chapter 4: Using FROST with Bitcoin](04_0_FROST_and_Bitcoin.md)
demonstrates how to use FROST signing for a real-world application:
verifying Bitcoin transactions.
