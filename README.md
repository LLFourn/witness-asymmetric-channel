# Abstract

This a proposal for a new channel symmetric channel construction that uses the
key idea from a recent paper called "Generalized Bitcoin-Compatible Channels"[[1]] and refines it.

This is a significant refinement of the [original proposal] posted to the lightning-dev mailing list.

# Background

As presently specified the two parties in a lightning channel are assigned different commitment transactions.
This _transaction asymmetry_ is logically necessary for the protocol to identify which party broadcasted a commitment transaction and potentially punish them if the other party provides proof it has been revoked (i.e. knows the revocation key).

It would be simpler if we could have a unified set of transactions for a state rather than two sets of transactions representing the same off-chain state for each party.
Riard first considered this problem in [[8]] while trying to add a punishment mechanism to eltoo[[9]] style channel updates.
They proposed that you could identify the broadcasting party using _witness asymmetry_ i.e. both parties broadcast the same transaction but have different witnesses. 
Unfortunately, the solution proposed is rather convoluted.

More recently in [[1]], Aumayr et al. introduced a much more elegant witness asymmetric solution using adaptor signatures.
Instead of being assigned different transactions, the parties are assigned different adaptor signatures as witnesses for the _same_ transaction.
The adaptor signatures force the party broadcasting a commitment transaction to reveal a "publishing secret" to the other party.
The honest party can then use this publishing secret along with knowledge of the usual revocation secret to punish the malicious party for broadcasting an old commitment transaction.

## Improvements over Aymayr et al.[[1]]

Our protocol is a refinement of the Aymayr et al.'s proposal in the following respects: 

### Time-locks

Aumayr et al. combine their idea with a "punish-then-split" mechanism similar to the original eltoo proposal.
Unfortunately, it therefore inherits the same issue with staggering time-locks.
BOLT 3 [[4]] describes why it avoids this problem as a design choice:

> The reason for the separate transaction stage for HTLC outputs is so that HTLCs can timeout or be fulfilled even though they are within the to_self_delay delay. Otherwise, the required minimum timeout on HTLCs is lengthened by this delay, causing longer timeouts for HTLCs traversing the network.

There is further background discussion to this choice in [[2]] and Towns proposes a solution for eltoo in [[3]].
In short, the problem is that it creates a risk for the party that needs to fulfill an HTLC with the secret in time because they must wait for the relative time-lock before completing it.
It is possible that this issue is not as severe as first thought and that the benefits of punish-then-split outweigh the costs.
But to avoid dealing with this question we provide an alternate transaction structure that mimics the current lightning timeout structure.

### Storage complexity

For each commitment transaction each party must communicate a publication point, whose secret is revealed if they publish the commitment transaction.
However, in order to extract the secret this point must be recalled upon seeing the transaction witness.
Clearly this leads to O(n) storage complexity where n is the number of commitment transactions.

We overcome this limitation with *revocable signatures* which preserve the main qualities of the mechanism but allow the victim to punish the revoked transaction with only static data. 

## Improvements over naive PTLC channels 

A useful benchmark for our protocol is the current lightning network except with the naive addition of PTLCs with asymmetric PTLC-success and PTLC-timeout transactions.
This means that adding a PTLC requires creating two new commitment transactions for each party each with their own PTLC output and pre-signed transactions spending from them.
An additional relative time-locked claim transaction needs to be signed in the case that the the PTLC transaction spends towards the party who owns the commitment transaction (to give the other party time to punish).
This leads to a total of 6 signatures exchanged.

Here is what we believe are the improvements our proposal offers over this:

- **Conceptual simpler**: Internally there are half the number of possible transactions and all outputs are a 2-of-2.
- **Less PTLC communication**: Since the commitment transactions and PTLC outputs are shared by both parties there is no need to sign two sets of PTLC transactions per PTLC -- only 4 signatures need to be sent. 
- **Eager punishment**: as soon as a revoked transaction is broadcast the punishment can be applied to the channel without having to wait for the revoked transaction to be confirmed.
- **Future compatible**: Little fundamentally changes when moving between ECDSA, non-aggregated Schnorr and aggregated Schnorr (i.e. MuSig). 

These improvements are admittedly minor.
However, we believe this idea is worth exploring as an intermediate upgrade path due to its simplification of the transaction structure.
Complexity is the number one enemy of security and also stifles the implementation of new features. 
The reduction in communication is also more pronounced in other protocols like *Discreet Log Contracts*[[10]] where many thousands of signatures need to be exchanged to add a protocol output to the commitment transaction.

## Revocable Signatures

A helpful dichotomy that distinguishes our approach is between revokable *transactions* (current lightning approach) and revokable *signatures* (our approach).
Presently, in lightning channels a transaction is revoked by the party owning that transaction revealing its associated revocation secret.
If the revoked commitment transaction is confirmed then all of its outputs may be taken by the other party by using their knowledge of the revocation secret.

In our witness asymmetric design a party will revoke their *signature* on a transaction rather than the transaction itself.
Revoking a signature happens in the same way as a transaction: you reveal a revocation secret.
Our goal is that when you broadcast a revoked signature **you reveal your static private key**.
Knowledge of your static private key is enough to claim all the funds from the funding output all commitment transaction outputs and any PTLC-success/timeout outputs.
This has an interesting implication: just by seeing the revoked signature attached to an old commitment transaction in your mempool, you can now claim the funding output immediately without waiting for the revoked transaction to confirm.

To fully emulate the coin access structure of current lightning channels for any given state (e.g. if the other party broadcasted the commitment transaction you can take your balance right away) we must extend our notion of revocable signatures.
Each revocable signature needs to be able to be *anticipated*. 
Note that typical Schnorr signatures have this property: given a message `m` a public key `X` and a nonce `R` we can *anticipate* the signature as `S = R + H(R || X || m) * X`.
This allows us to use the revelation of `s` to reflect a state in the protocol.
This anticipated signature essentially replaces the role of the "publishing secret" from [[1]] since we use the revelation of the underlying signature to identify which party published the commitment transaction.
The main difference is that anticipated signatures do not require storing a "publishing point" per commitment transaction to enact the punishment.

### Revocable signatures with non-aggregated 2-of-2

Revocable signatures for non-aggregated multi-signatures can be implemented using single signer adaptor signatures.
This applies to `OP_CHECKMULTISIG` with Bitcoin as it is today or Tapscript Schnorr 2-of-2.
Let `Ra` be Alice's revocation key and `A` and `B` be Alice Bob's static public keys respectively.
Bob can create a revocable signature for Alice by giving her an adaptor signature under `B` encrypted by `Ra + A` (i.e. Alice will reveal the discrete log of `Ra + A` if she decrypts and broadcasts the signature).
For Alice to revoke her signature to Bob she reveals the discrete log of `Ra` to Bob.
Now if Alice were to decrypt and broadcast the signature Bob can extract the discrete logarithm of `A` from it (i.e. learn her static secret key since he knows the discrete logarithm of `Ra` he can subtract it from `Ra + A`).

Although ECDSA has adaptor signatures [[6]] they are difficult to anticipate because of the modular inversion done on the secret nonce.
Instead we can just anticipate the revelation of `Ra + A` (in the above example) to know when Alice has broadcasted her revocable (but not necessarily revoked) signature.

### Revocable signatures with key aggregated 2-of-2

Making a key aggregated signature revocable is rather straightforward -- the revocation key is simple the nonce you used to create the signature.
Let (`a`, `A`), (`b`, `B`) and (`ra`, `Ra`), (`rb`, `Rb`) be the static key-pairs and nonce pairs for a particular two-party signing protocol execution of Alice and Bob respectively.
We omit the protocol to actually produce these signatures for now.

Imagine that Alice receives `s = ra + rb + H(A + B || Ra + Rb || m)` as the joint signature from the signing protocol (and keeps it secret from Bob).
Clearly Alice can verifiably revoke the signature by revealing `ra` to Bob since if Bob were to see the signature afterwards Bob could extract `a` as

`a = s - (ra - rb)/H(Ra + Rb || A + B || m) - b`

# Proposal

Given either of the above revocable signature schemes we propose a new witness asymmetric channel construction.

## Notation

We describe the protocol with respect to two parties Alice and Bob.
`2-of-2(A,B)` refers to a multi-signature output that requires the owners of `A` (Alice) and `B` (Bob) to authorize it.
The multi-signature scheme needs to be revocable as described above.

In practice `2-of-2(A,B)` should be interpret as some deterministic randomization of `A,B` so that all `2-of-2(A,B)` outputs do not look the same however we omit this for clarity.

We refer to the anticipated signature of Alice and Bob's revocable signature on the commitment transaction as `publicationA_i` and `publicationB_i` respectively.
These are revealed when Alice or Bob broadcasts their revocable (but not necessarily revoked) signature.

## The Fund transaction

The structure of channel funding does not change much from the current BOLT spec.
Before creating the Fund transaction the two parties exchange their static public keys they will use for the channel (`A` and `B`).
The Fund transaction spends from the funding party's (or funding *parties*') outputs to a `2-of-2(A,B)`.

## Commitment transactions

A commitment transaction represents a state of the channel.
They each spend from the `2-of-2(A,B)` on the Fund transaction. 
A commitment transaction has two balance outputs and zero or more PTLC outputs.
Before signing the ith commitment transaction the parties must exchange revocation public keys `Ra_i` and `Rb_i`.

Each commitment transaction is signed by both parties so that each party has a single (but different) revocable signature on it e.g. Alice has a revocable signature spending on the commitment transaction with `Ra_i` as the revocation key.

## Balance outputs

The two balance outputs assign some proportion of the funds exclusively to each party.
Each one has a scriptPubkey of:

- Alice: `2-of-2(A, publicationB_i) `
- Bob: `2-of-2(publicationA_i, B)`

Each Balance output has a relative time locked claim transaction spending it to an address owned by the deserving party.
Note that if Alice publishes a valid state she reveals `publicationA_i` and Bob can take his funds immediately.
As soon as Bob sees a revoked signature attached to an old commitment transaction published by Alice he learns the secrets for both `publicationA_i` and `A` so can immediately take both his and Alice's balance outputs.

## PTLC outputs

PTLC outputs are added directly to the commitment transaction.
Each PTLC has two pre-signed transactions spending from it, PTLC-success and PTLC-timeout, which both spend to a single destination address.
Broadcasting the PTLC-success transaction reveals the PTLC secret in the usual way using an adaptor signatures.
The PTLC-timeout transaction has an absolute time-lock on it through the nlocktime transaction value.
Only the party which stands to gain from the PTLC transactions receives a signature on it i.e. if Alice is offering a PTLC to Bob then Alice will have a signature on the PTLC-timeout transaction and Bob will have a signature on the PTLC-success transaction.

A PTLC output scriptPubkey is a simple combination of the static keys: `2-of-2(A,B)`.

For a PTLC output being offered by Alice to Bob the outputs of the PTLC transactions are:

- PTLC-success: `2-of-2(publicationA_i, B)`
- PTLC-timeout: `2-of-2(A, publicationB_i)`

Note the following scenarios:

- Alice confirms a valid commitment transaction and the PTLC-success transaction confirms first: Bob immediately claims the funds since he knows both `publicationA_i` and `B`.
- Alice confirms a valid commitment transaction and the PTLC-timeout transaction confirms first: Alice waits for the relative time-lock of her claim transaction and then broadcasts that. 
- Alice confirms a revoked commitment transaction: Bob can take the PTLC output right away since it's a `2-of-2(A,B)` and he can the secret for `A` from the revoked signature attached to the revoked transaction.
- Alice confirms a revoked commitment transaction and the PTLC-timeout transaction: Bob can take the PTLC-timeout output since he can extract `A` from the revoked signature attached to the revoked transaction.

The implications of switching the direction of the PTLC are straightforward.


[original proposal]: ./original.md
[1]: https://eprint.iacr.org/2020/476.pdf 
[2]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2015-November/000339.html 
[3]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-January/002448.html  
[4]: https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md
[5]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-December/002375.html 
[6]: https://joinmarket.me/blog/blog/schnorrless-scriptless-scripts/
[7]: https://joinmarket.me/blog/blog/flipping-the-scriptless-script-on-schnorr/_ 
[8]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-July/002064.html
[9]: https://blockstream.com/eltoo.pdf
[10]: https://github.com/discreetlogcontracts/dlcspecs
