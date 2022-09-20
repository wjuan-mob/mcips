- Feature Name: Reserved Subaddresses
- Start Date: (2022-04-10)
- MCIP PR: [mobilecoinfoundation/mcips#36](https://github.com/mobilecoinfoundation/mcips/pull/36)
- Tracking Issue: https://github.com/mobilecoinfoundation/mobilecoin/issues/1787

[summary]: #summary

Subaddress indices are unsigned 64-bit integers.

The indices with high bit set are reserved for "special purposes", and should
not be assigned to individual contacts by desktop wallets that are using subaddresses this way.

`u64::MAX` will be reserved for an "invalid / none" value.

[mobilecoinfoundation/mcips#4](https://github.com/mobilecoinfoundation/mcips/pull/4) will be updated,
changing the change subaddress from `1` to `u64::MAX - 1`.

Subsequent reserved indices will count down from there.

# Motivation
[motivation]: #motivation

In developing new features, we have discovered that sometimes, we want to reserve particular subaddress
indices for specific purposes. This is particularly helpful for mobile wallets which are designed to
be used with "linked devices" that share your private keys. By using the subaddress index to represent
information about a `TxOut`, they can ensure that other linked devices will all see this information,
without needing to create another synchronization mechnaism.

A good example of this is the "change subaddress" introduced in
[mobilecoinfoundation/mcips#4](https://github.com/mobilecoinfoundation/mcips/pull/4).

However, this is at odds with how the desktop wallets use subaddresses. The desktop wallets
assign different subaddresses to different contacts, counting up from 1.

To avoid collisions and promote compatibility, we want to ensure that the change subaddress and other
special subaddresses will never be used for contacts by desktop wallets. This will help desktop wallets
to be able to make as much sense of this information as possible if a user imports their mobile wallet keys
into a desktop wallet.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Subaddress indices with the high bit set are reserved for special purposes.

These will be assigned numbers counting down from `u64::MAX`. This avoids collisions
with subaddresses already assigned by the desktop wallet.

| Subaddress index | Purpose         |
| ---------------- | --------------- |
| `u64::MAX`       | Invalid / None      |
| `u64::MAX - 1`   | Change subaddress [MCIP #4](https://github.com/mobilecoinfoundation/mcips/pull/4)      |
| `u64::MAX - 2`   | Gift code subaddress [MCIP #32](https://github.com/mobilecoinfoundation/mcips/pull/32) |

# Drawbacks
[drawbacks]: #drawbacks

This is reserving a lot of subaddresses. However, if later it turns out we need to
give most of them back, that should be fine.

It turns out that the full-service desktop wallet is already sending change to a change subaddress,
but only in certain configurations, and it is using subaddress index 1 for this.
This proposal will require full-service to start using `u64::MAX - 1` after block version 1,
and can have backwards compatibility support for subaddress index 1 in blocks before that version.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative would have been to use change subaddress index 1 for all clients, and not use `u64::MAX - 1`.

However, this has a major drawback for anyone using `mobilecoind` today, which is that they have been using
subaddress index 1 with contacts. So they have already handed that subaddress out to their contacts, and they
have no practical ability to change it now that they've given it away. If they are an exchange, customers may
still deposit money to that address.

Additionally, the change subaddress index plays a critical role in RTH memo validation in MCIP #4.
It is regarded as a secret -- anyone who knows your change subaddress is able to send you destination memos
that your clients will treat as valid. If you are an exchange and someone knows your change subaddress, then
they can potentially forge bad destination memos and defraud you.

Therefore, it is safer to use `u64::MAX - 1`, a subaddress index that has never been used by any client before,
as the change subaddress index, and to migrate full-service to use this before RTH memos exist in the wild.

*Specifically, full-service, and any either client, should NEVER treat a destination memos as valid if it comes
on the subaddress of index 1.*

Since this means that we cannot standardize on `1` as the change subaddress index, `u64::MAX - 1` is the best
remaining option.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None at this time.

# Future possibilities
[future-possibilities]: #future-possibilities

None at this time.
