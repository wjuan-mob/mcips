- Feature Name: Fog-Compatible-Gift-Codes
- Start Date: Mar 21
- MCIP PR: [mobilecoinfoundation/mcips#0032](https://github.com/mobilecoinfoundation/mcips/pull/0032)
- Tracking Issue: https://github.com/mobilecoinfoundation/mobilecoin/issues/1735

# Summary
[summary]: #summary

We propose an alternate mechanism of function for the "gift codes" feature, which is compatible with
Mobile wallets using MobileCoin Fog.

The payload of the new mechanism is similar in size, and mobile apps are able to track
the state of the gift code using on-chain data. Additionally, this mechanism is in our opinion
simpler since it doesn't require creating a temporary account, and instead gives ownership of a `TxOut`
directly.

We propose that TxOut's which are sent as gift codes, should be paid first to the reserved
`GIFT_CODE_SUBADDRESS_INDEX`, which is equal to `u64::MAX - 2` (per MCIP #36).

# Motivation
[motivation]: #motivation

Full-service and mobilecoind support a "gift codes" feature. (Called "Transfer code" [in sources](https://github.com/mobilecoinfoundation/mobilecoin/blob/7077b418fab65e05a05e4fca1d272e3dddd2e392/api/proto/printable.proto#L28).)

The idea of a gift code is that you can give money to
someone without knowing their public address before-hand, creating a payload that can be handed to
anyone, which then funds their account. This could be used for gift cards for example.

The existing mechanism for gift codes is as follows:
* The person who creates a gift code, creates an entirely new set of account keys, and funds these account keys with an on-chain transaction.
* The gift code payload contains these account private keys.
* To receive a gift code, the recipient drains the funds from this temporary account into their main account.

This mechanism is not very robust in the sense that, if an "attacker" intercepts the gift code, then they can race the
recipient to drain the funds. Indeed, even the sender is capable of racing the recipient to take back the funds.
However, the design goal of the feature is that the sender of the gift code doesn't know the public address of the
recipient in advance, and wants whoever
obtains the payload to be able to use it, so these races are not considered a design flaw.
This feature has been used for example to implement the "red envelopes" feature in Mixin
messenger group chats -- one or more gift codes are transmitted in an encrypted group chat, and whoever clicks on it first
gets a festive reward.

While this functionality works when all the participants are using a desktop wallet like full-service, it turns out that this doesn't
work when the recipient is on a mobile device using a Fog account. The reason is that they are unable to perform a balance
check of the temporary account, since the outputs were not indexed by Fog.

This could be worked around, by making the person who creates the temporary account create it as a Fog account.
However, there could be several deployments of Fog, and this basically means that the creator of the gift code needs to know
what software the recipient is using, which is counter to the goal of the feature.

In later versions of transfer payloads, the `TxOut` public key was included.

```
/// Message encoding a private key and a UTXO, for the purpose of
/// giving someone access to an output. This would most likely be
/// used for gift cards.
message TransferPayload {
    /// [Deprecated] The root entropy, allowing the recipient to spend the money.
    /// This has been replaced by a BIP39 entropy.
    bytes root_entropy = 1 [deprecated=true];

    /// The public key of the UTXO to spend. This is an optimization, meaning
    /// the recipient does not need to scan the entire ledger.
    external.CompressedRistretto tx_out_public_key = 2;

    /// Any additional text explaining the gift
    string memo = 3;

    /// BIP39 entropy, allowing the recipient to spend the money.
    /// When deriving an AccountKey from this entropy, account_index is always 0.
    bytes bip39_entropy = 4;
}
```

It is possible to use the untrusted fog ledger endpoint to lookup `TxOut` by their public key,
but this is not private, and the fog operator can see which `TxOut` you looked up. This puts the
gift codes outside the normal threat-model for Fog transfers.

We propose to change the design to improve this
* Instead of sending the `tx_out_public_key`, we should send the global index of the tx out. Then, even a fog user
  can use this to recover the `TxOut` privately from fog-ledger. Also this results in a slightly smaller payload,
  and serves the purpose of uniquely identifying the `TxOut`.
  
Second, we propose an additional change in the design which improves some aspects of this.
* Instead of sending money to a temporary account, we try to just transfer ownership of a specific `TxOut`, by giving them
the private keys to that TxOut
  * To enable a recipient to see the unblinded amount of the `TxOut`, we just have to give them the `shared_secret`.
  * To enable a recipient to spend a `TxOut`, we just have to give them the `onetime_private_key`.
  
Given this data, a Fog user who receives the payload can ask Fog ledger for a merkle proof of membership for the `TxOut` of given index.
They can get the entire `TxOut` this way, match it against the shared secret and so unblind the amount commitment, and given the one-time
private key, they can spend it as normal using the transaction builder.

A major concern for mobile apps is the ability to track things across linked devices using the blockchain (and not a separate synchronization
mechanism). It is a concern for instance that one app may be trying to give away a `TxOut` in a gift code, while another selects it to be spent in a transaction.

To avoid that, we propose to standardize that `GIFT_CODE_SUBADDRESS_INDEX = u64::MAX - 2`, and that before building a gift code, the app should make a self-payment to this
subaddress with the correct amount for the gift code. Then the gift code is built against the `TxOut` that belongs to subaddress `u64::MAX - 2`.

The advantage of this is that unlike with the temporary address, the fog user is able to track the gift code.
* The `TxOut` still belongs to the sender's address, even if we gave the private keys away to someone (who may not have used it yet).
* We don't lose track of the in-flight gift code if the app is restarted -- all linked devices can learn about the in-flight gift code because of the use of subaddress 2.
  This was not the case with the temporary address.
* Because the `TxOut` still belongs to us, we can see if the gift code is actually used by the recipient, and if for some reason it is never used, we can
  reclaim the money. (This would still be possible with the previous design, but it's more complicated because we have to preserve the temporary address entropy off-chain somewhere.)
* The app can also more simply have recoverable transaction history around the use of gift codes, because they can represent in-bound `TxOut`'s to subaddress 2 as gift-code funding
  events.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A `TxOutGiftCode` is a payload which confers ownership of an individual `TxOut` in the blockchain,
and is a little less than 80 bytes in size.

* To enable someone to spend a TxOut, we just have to give them the `onetime_private_key` of the `TxOut`.
* To enable someone to see the unblinded amount of the TxOut, we just have to give them the `shared_secret` of the `TxOut`.
* To enable someone to find the TxOut in the ledger (even a Fog user), we just have to tell them the global index in the blockchain.

When a Fog wallet receives such a payload, they can first contact the Fog Ledger service with the global index, which returns to them a merkle proof of
membership, and a copy of the `TxOut` as it appears in the ledger. (Wallets that have a local `ledger_db` can just search the ledger directly.)

The wallet can then attempt to unmask the amount using the `shared_secret` of the `TxOut`. Assuming this succeeds, they know the value and blinding factor
of the `TxOut`. Given the one-time private key, they can compute the `KeyImage` and check if it is already spent. Finally, given all this data and a Merkle proof,
they can spend the `TxOut` as normal with the `TransactionBuilder`.

When building a `TxOutGiftCode`, the app should first make a self-payment with the desired amount of the gift code to `GIFT_CODE_SUBADDRESS_INDEX = u64::MAX - 2`, which is
now reserved for `TxOut`'s used with `TxOutGiftCode`. Apps should not select `TxOut`'s belonging to this subaddress for the purpose of Txo selection for other transactions,
and they may choose not to count these balances towards the account balance represented to the user -- these `TxOut` amounts represent the value of in-flight gift codes.

After a `TxOutGiftCode` is sent, the app can check if it is claimed (or "still pending") based on if this `TxOut` has been spent by someone. Until it is spent, the app can potentially
cancel the gift code by paying the `TxOut` back to themself on subaddress 0.

We additionally standardize Recoverable Transaction History memos, extending the scheme from [MCIP #4] to handle gift code transactions as well.


| Memo type bytes | Name                                              |
| -----------     | -----------                                       |
| 0x0201          | Gift code funding memo                            |
| 0x0202          | Gift code cancellation memo                       |
| 0x0002          | Gift code sender memo                       |

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A `TxOutGiftCode` conforms to the following protobuf schema, using types from [`external.proto`](https://github.com/mobilecoinfoundation/mobilecoin/blob/924abcff4937fced12cdd951aad598dbdbbfab05/api/proto/external.proto):

```
message TxOutGiftCode {
    // The global index of the TxOut which is gifted
    fixed64 global_index = 1;
    // The one-time private key which can be used to spend this TxOut
    RistrettoPrivate onetime_private_key = 2;
    // The shared secret which can be used to unblind the amount of this TxOut
    CompressedRistretto shared_secret = 3;
}
```

As described, a Fog wallet can look up a `TxOut` from its global index, and then
attempt to unblind the amount using the shared secret.

To add an input to the transaction builder, normally we construct an [`InputCredentials` object](https://github.com/mobilecoinfoundation/mobilecoin/blob/924abcff4937fced12cdd951aad598dbdbbfab05/transaction/std/src/input_credentials.rs#L11):

```
pub struct InputCredentials {
    /// A "ring" containing "mixins" and the one "real" TxOut to be spent.
    pub ring: Vec<TxOut>,

    /// Proof that each TxOut in `ring` is in the ledger.
    pub membership_proofs: Vec<TxOutMembershipProof>,

    /// Index in `ring` of the "real" output being spent.
    pub real_index: usize,

    /// Private key for the "real" output being spent.
    pub onetime_private_key: RistrettoPrivate,

    /// Public key of the transaction that created the "real" output being
    /// spent.
    pub real_output_public_key: RistrettoPublic,

    /// View private key for the address this input was sent to
    pub view_private_key: RistrettoPrivate,
}
```

All the clients are capable of sampling rings, obtaining membership proofs, and the
`TxOutGiftCode` contains the one-time private key.

However, the `InputCredentials` appears to require the view private key, which the recipient
of the `TxOutGiftCode` does not have.

Fortunately, it turns out that they don't really need it, it is only used in the `TransactionBuilder`
to construct the `shared_secret`:

https://github.com/mobilecoinfoundation/mobilecoin/blob/924abcff4937fced12cdd951aad598dbdbbfab05/transaction/std/src/transaction_builder.rs#L410

We propose either to
* Modify `InputCredentials` so that it includes the `shared_secret` instead of the `view_private_key`.
  Since clients usually compute the `shared_secret` as soon as they receive an input as a by-product of view-key matching,
  this may actually be more efficient and prevent computing elliptic curve operations again unnecessarily.
* Modify `InputCredentials` so that it holds either the `shared_secret` OR the `view_private_key`, for backwards
  compatibility.

Clients will then be able to go from a `TxOutGiftCode` to an `InputCredentials` very easily.

We could then make this schema a new option to `PrintableWrapper`.
For the older transfer payload, we can either deprecate it, or, we could introduce a `global_index` field which can replace the
`tx_out_public_key` and deprecate the use of the `tx_out_public_key` field.

When an app finds a TxOut belonging to the `GIFT_CODE_SUBADDRESS`, it can infer that this belongs to an in-flight gift code.
When this TxOut is spent, the app can infer that the gift code was redeemed, or canceled. The app may cancel a gift code by
paying it back to the primary subaddress.

## Recoverable transaction history and memos

Following MCIP #4, each transaction typically has an output from Alice to Bob, and a change output from Alice to Alice.
The memo on the change output is a note Alice writes to herself to keep track of her own activity. Alice validates these
memos by confirming that they belong to the change subaddress. The memo on the other output is a note from Alice to Bob,
typically identifying Alice.

When building a transaction which funds a gift code, the TxOut to `GIFT_CODE_SUBADDRESS` is called the "gift code output", and a change output is always written (even if value zero):
* `0x0201 Gift code funding memo` should be attached to the change output. This memo will point to the gift code output by way of a hash, and contains any note about the purpose of the gift code.
* `0x0002 Gift code sender memo` is attached to the gift code output.

When claiming a gift code, the gift code output is spent in a self-payment to the change subaddress. This transaction has a single output, the
change output, which has a memo of type `0x0002 Gift code sender memo`.
The app should attempt to validate the information in the memo from the gift code. For example, in a chat application, the app may be able
to validate who sent the gift code payload in a message. If the information is valid, then the app should copy the memo from the gift code TxOut into the
self-payment TxOut. If the app decides that the information cannot be validated, then it may use `0x0000 Unused` instead, to avoid committing
false information. Future apps may assume that any memos attached to TxOut's owned by the change subaddress are known to be valid.

When cancelling a gift code, the gift code output is paid back to self. This transaction has a single output, the change output, which goes to
the change subaddress and has a `0x0202 Gift code cancellation memo`. This memo contains the tx public key of the gift code output.

The overall strategy for an app implementing recoverable transaction history looks like this then:
* Any TxOut's on the default subaddress are in-bound transactions, which may have 0x01.. authenticated sender memos which can be validated.
* Any TxOut's on the change subaddress contain recoverable transaction history memos that we can trust
  * 0x0200 Destination memo indicates an outbound transfer, and includes the recipient and the amount
  * 0x0201 Gift code funding memo indicates that we funded a gift code, and indicates which TxOut is the gift code
  * 0x0202 Gift code cancellation memo indicates that we cancelled a gift code that we previously funded
  * 0x0002 Gift code sender memo indicates that a gift code was claimed by the recipient and was sent by the sender
  * 0x0000 Unused indicates that we claimed a gift code but we aren't sure who it came from

* Any TxOuts on the gift code subaddress indicate gift codes that we funded.
  * We can see the amount of the gift code using our view key
  * We can obtain notes about these TxOut's by finding the associated 0x0201 memos which would be in the same block
  * If such a TxOut is spent, then we can represent it as claimed, unless a matching 0x0202 memo appears, in which case we know we cancelled it.

## 0x0201 Gift code funding memo

| Byte range | Item |
| ---------- | ---- |
| 0 - 4      |  First 4 bytes of blake2b over `tx_out_public_key` of gift code tx out |
| 4 - 64     | "Note": A UTF-8 string with that is either exactly 60 bytes or null terminated if under 60 bytes |

How exactly the note is used may vary from application to application -- it could be a e.g. a phone number,
a user id, or user generated text.

It is correct for any application to display the "note" to the user as a human-readable string.

Some applications will have more ability to make sense of data like a phone number or a user id than others, but it is
correct for any app to do no more than display the note in the user's transaction history, to help them remember about
the gift code and/or if they should cancel it.

The rationale behind the 4-byte hash is:
* The gift code funding memo is always paired with a gift code tx out, which appears in the same block.
* Blocks are not that big, and particularly, users don't have that many TxOut's in a block.
* In a hash space with 4 billion possible items, the probability that there are two gift codes created by the same user in the
  same block with the same hash, is very small -- even if there are 10 gift codes funded in the same block, this probability would be
  less than one in ten million.
* Using only 4 bytes here reserves 60 bytes for the user's note

## 0x0202 Gift code cancellation memo

| Byte range | Item |
| ---------- | ---- |
| 0 - 8      | `global_index` of gift code tx out that was cancelled |
| 8 - 64     | Unused bytes |

## 0x0002 Gift code sender memo

| Byte range | Item |
| ---------- | ---- |
| 0 - 64     | "Note": A UTF-8 string with that is either exactly 64 bytes or null terminated if under 64 bytes |

In lieu of an address hash, the sender accounts for the source of the funds with a human-readable note.

# Drawbacks
[drawbacks]: #drawbacks

In our opinion, there is no reason why not to adopt at least the `global_index` change, since this is superior in every way to
sending the `tx_out_public_key`.

Using the `TxOutGiftCode` instead of the transfer-payload in mobile apps is superior because it enables linked devices to track
and potentially reclaim the gift code more finely, without relying on off-chain synchronized storage to track the in-flight transfer payloads.

One drawback is that this redesign did not achieve the goal of having only one on-chain event per gift code.
We had hoped that by eliminating the need for a temporary account, and just transferring a `TxOut` directly, we
would make it so that only the recipient needs to have an on-chain event. It turns out that an on-chain event
is still required in practice and probably unavoidable, at least with this approach.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There are a few minor technical variations we considered like, not revealing the shared secret, and revealing
the amount and blinding factor directly instead, but this is more bytes on the wire, and it may be useful for
the recipient to be able to read the memo anyways.

Concieveably, we could avoid the sender-side on-chain event, by using an [MCIP #31](https://github.com/mobilecoinfoundation/mcips/pull/0032)
"Signed Contingent Input". Then if you have a `TxOut` for 100 MOB, and you want to create a gift-code for 10 MOB, you could make the signed
contingent input and require that 90 MOB comes back to you as change, and give this to the counterparty, and then that would be the gift code.
However, this has a lot of drawbacks, one being that, it doesn't help linked devices synchronize this, and another being that, these inputs
are much bigger than the `TransferPayload` and the `TxOutGiftCode`, because they have to contain an entire Ring MLSAG.

# Prior art
[prior-art]: #prior-art

None that we are aware of.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None that we are aware of.

# Future possibilities
[future-possibilities]: #future-possibilities

In the future we may want to extend the gift code memos, e.g. to include payment request ids or something. But this set of memos seems to meet
the goals at end, including, creating memos for a peer, tracking, and cancelling them, and recovering this history, as well as creating gift
codes with text notes.
