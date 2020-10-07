# Abstract

This a proposal for a new channel symmetric channel construction that uses the
key idea from a recent paper called "Generalized Bitcoin-Compatible Channels"[1] and refines it.

This is a revision of the [original proposal] posted to the lightning-dev mailing list.

# Background

As presently specified the two parties in a lightning channel are assigned different commitment transactions.
This _transaction asymmetry_ is logically necessary for the protocol to identify which party broadcasted a commitment transaction and potentially punish them if the other party provides proof it has been revoked (i.e. knows the revocation key).

It would be simpler if we could have a single set of transactions for any layer-2 protocol rahter than two transactions representing the same notional protocol state for each party.
Riard first considered this problem in [8] while trying to add a punishment mechanism to eltoo[9] style channel updates.
They proposed that you could identify the broadcasting party using _witness asymmetry_ i.e. both parties broadcast the same transaction but have different witnesses. 
Unfortunately, the solution proposed is rather convoluted.

More recently in [1], Aumayr et al. introduced a much more elegant witness asymmetric solution using adaptor signatures.
Instead of being assigned different transactions, the parties are assigned different adaptor signatures as witnesses for the _same_ transaction.
The adaptor signatures force the party broadcasting a commitment transaction to reveal a "publishing secret" to the other party.
The honest party can then use this publishing secret along with knowledge of the usual revocation secret to punish the malicious party for broadcasting an old commitment transaction.

## Improvements over Aymayr et al.[1]

Our protocol is a refinement of the Aymayr et al.'s proposal in the follwoing respects: 

### Timelocks

Aumayr et al. combine their idea with a "punish-then-split" mechanism similar to the original eltoo proposal.
Unfortunately, it therefore inherits the same issue with staggering time-locks.
BOLT 3 [4] describes why it avoids this problem as a design choice:

> The reason for the separate transaction stage for HTLC outputs is so that HTLCs can timeout or be fulfilled even though they are within the to_self_delay delay. Otherwise, the required minimum timeout on HTLCs is lengthened by this delay, causing longer timeouts for HTLCs traversing the network.

There is further background discussion to this choice in [2] and Towns proposes a solution for eltoo in [3].
In short, the problem is that it creates a risk for the party that needs to fulfill an HTLC with the secret in time because they must wait for the relative time-lock before completing it.
It is possible that this issue is not as severe as first thought and that the benefits of punish-then-split outweight the costs.
But to avoid dealing with this question we provide an alternate transaction structure that mimics the current lighthing timeout structure.

### Storage complexity

For each commitment transaction each party must communicate a publicaiton point, whose secret is revealed if they publish the commitment transaction.
However, in order to extract the secret this point must be recalled upon seeing the transaction witness.
Clearly this leads to O(n) storage complexity where n is the number of commitment transactions.

Here we overcome this limitation with *revocable signatures* which preserve the main qualities of the mechansim but allow the victim to punish the revoked trasaction with only static data. 

## Improvements over naive PTLC channels 

A useful benchmark for our protocol is the current lightning network except with a naive addition of PTLCs with asymmetric PTLC-success and PTLC-timeout transactions (i.e. adding a PTLC to state adds four new transactions to the tree -- two for each participant's commitment transaction).
Here is what we beleive are the improvements our proposal offers over this:

- **Conceptuall simpler**: Internally there are half the number of possible transactions and all outputs are a 2-of-2.
- **Half the PTLC communication**: Since the commitment transactions and PTLC outputs are shared by both parties there is no need to sign two sets of PTLC transactions per PTLC 
- **Eager pubishment**: as soon as a revoked transaction is broadcast the punishment can be applied to the channel wihtout having to wait for the revoked transaction to be cofirmed.
- **Future compatible**: Little fundmentally changes when moving between ECDSA, non-aggregaed Schnorr and aggregated Schnorr (i.e. MuSig). 

These improvements are admittedly minor in most respects.
However, we believe this idea is worth exploring to see how much of an impact the simpler structure offers.
Complexity is the number one enemy of security and also stifles the implementation of new features. 
The halving of communication is also material for other off-chain transformations of protocols where thousands of signatures may need to be exchanged per commitment transaction.

## Revocable Signatures

A helpful dichotomy that distingishes our appraoch is between revokable *transactions* and revokable *signatures*.
Presently, in the lightning network, a transaction is revoked by the party owning that transaction revealing its associated revocation secret.
If the revoked commitment transaction is confirmed then all of its outputs may be taken by the other party with knowledge of the revocation secret.

In our witness asymmetric design a party will revoke their signature on a transaction rather than the transaction itself.
Revoking a signature happens in the same way as a transaction: you reveal a revocation secret.
Our goal is that when you broadcast a revoked signature **you reveal your static private key**.
Knowledge of your static private key is enough to claim all the funds from the funding output all commitment transaction outputs and any PTLC-success/timeout outputs.
This has an interesting implication: just by seeing the revoked siganture attached to an old commitment transaction in your mempool, you can now claim the funding output immediately without waiting for the revoked transaction to confirm.

To fully emulate the coin access structure of current lighthing for any given state (e.g. if the other party braodcasted the commitment transaction you can take your balance right away) we must extend our notion of revocable signatures.
Each revocable signature needs to be able to be *anticipated*. 
Note that typical Schnorr signatures have this property: given a message `m` a public key `X` and a nonce `R` we can *anticipate* the signature as `S = R - H(R || X || m) * X`.
This allows us to use the revelation of `s` to reflect a state in the protocool.
In our case, we use the revelation of the anticipated signature to indicate that a certain commitment transaction has been published by a certain party and refer to them as "publication points" just as in [1].
Note that in [1] the publication is an idepdenent point that is communicated for each commitment transaction which must be stored for the life of the channel.
Here the publication point is derived from the public revocation key and other data which can be deterministically reproduced allowing us to have O(1) storage complexity.

### Revocable signatures with non-aggregated 2-of-2

Revocable signatures for non-aggregated multi-signatures can be implemented using single signer adaptor signatures  we use adaptor signatures.
This applies to `OP_CHECKMULTISIG` with Bitcoin as it is today or Tapscript schnorr 2-of-2.
Let `Ra` be Alice's revocation key and `A` and `B` be Alice Bob's static public keys respectively.
For Bob to create a revocable signature for Alice by giving her an adatpor signature under `B` encrypted by `Ra + A` (i.e. Alice will reveal the discrete log of `Ra + A` if she decrypts and broadcasts the signature).
For Alice to revoke her signature to Bob she reveals the discrete log of `Ra` to Bob.
Now if Alice were to decrypt and broadcast the signature Bob can extract the discrete logarithm of `A` from it (i.e. learn her static secret key).

The associated publication point is simply the encryption key for the adaptor signature i.e. `publication_A = Ra + A`.

### Revocable signatures with aggregated 2-of-2

Making a key aggregated signature revocable is rather straightforward -- the revocation key is simple the nonce you used to create the signature.
Each party needs a *different* signature on the same commitment transaction so both parties end up with a single revocable signature that only they know.
To acheive this, the two parties execute the signing protocol twice in parallel.
For Alice and Bob respectively let,

- (`ra_i`,`rb_i`) be their revocation secret keys with `Ra_i = ra_i * G` etc.
- (`ra_i'`, `rb_i'`) be their auxialliary deterministically produced nonces with `Ra' = ra' *G` etc 
- (`a`,`b`) be their static secret keys with `A = a * G` etc and their joint public key is `A + B`.

The two parties receive signatures of the following form where `m` is the transaction BIP341 transaction digest.

- Alice: `sa = ra_i + rb_i' + H(Ra_i + Rb_i' || A + B || m)(a + b)`
- Bob: `sa = ra_i' + rb_i + H(Ra_i' + Rb_i || A + B || m)(a + b)`

We omit the protocol to actually produce these signatures for now.
Importantly, we note that if (for example) Alice brodcasts `sa` after revealing `ra_i` to Bob, Bob can determinstically re-generate `rb_i'` and `ra_i` and use it to extract `a` as follows:

`a = sa - (ra_i - rb_i')/H(Ra_i + Rb_i || A + B || m) - b`.

The publication point the signatures is simply the image of `sa` and `sb` i.e. `publication_A = Ra_i + Rb_i' - H(Ra_i + Rb_i' || A + B || m) * (A + B)`.

# Proposal

Given either of the above revocable signature schemes we propose a new witness asymmetric channel construction.

## Notation

We describe the protocol with respect to two parties Alice and Bob.
`2-of-2(A,B)` refers to a multi-signature output that requires the owners of `A` (Alice) and `B` (Bob) to authorize it (e.g. `OP_CHECKMULTISIG` with ECDSA [6], or MuSig with Schnorr etc [7]).

In practice `2-of-2(A,B)` should be interpretd as some deterministic randomization of `A,B` so that all `2-of-2(A,B)` outputs do not look the same.

## The Fund transaction

The structure of channel funding does not change much from the current BOLT spec.
Before creating the Fund transaction the two parties exchange their static public keys they will use for the channel (`A` and `B`).
The Fund transaction spends from the funding party's (or funding _parties_) outputs to an output of `2-of-2(A,B)`.

## Commitment transacitons

A commitment transaction represents a state of the channel.
They each spend from the `2-of-2(A,B)` on the Fund transaction and represent conflicting "states" of the channel where only the most recent one is valid.
A commitment transaction has two balance outputs and zero or more PTLC outputs.
Before signing the ith commitment transaction the parties must exchange revocation public keys `Ra_i` and `Rb_i`.

Each commitment transaction is signed by both parties so that each party has a single (but differnet) revocable signature on it e.g. Alice has a revocable signature spending on the commitment transaction with `Ra_i` as the recocation key.

## Balance outputs

The two balance outputs assign some proportion of the funds exclusively to each party.
Each one has a scriptPubkey of:

- Alice: `2-of-2(A, publication_B) `
- Bob: `2-of-2(publication_A, B)`

Each Balance output has a relative time locked claim transactoin spending it to an address owned by the deserving party.
Note that if Alice publishes a non-revoked state she reveals `publication_A` and Bob can take his funds immediately.
As soon as Bob sees a revoked signature attached to an old commitment transaction published by Alice he learns the secrets for both `publication_A` and `A` so can immediately take both his and Alice's balance outputs.

## PTLC outputs

PTLC outputs are added directly to the commitment transaciton.
Each PTLC has two pre-signed transactions spending from it: PTLC-success and PTLC-timeout.
Broadcasting the PTLC-success transaction reveals the PTLC secret in the usual way using an adaptor signatures.
The PTLC-timeout transaction has an absolute time-lock on it through the nlocktime transaction value.
Only the party which stands to gain from the PTLC transactions receives a signature on it i.e. if Alice is offering a PTLC to Bob then Alice will have a signature on the PTLC-timeout transaction and Bob will have a signature on the PTLC-success transaction.

A PTLC output scriptPubkey is a simple combination of the static keys: `2-of-2(A,B)`.

For a PTLC output being offered by Alice to Bob the outputs of the PTLC transactions are:

- PTLC-success: `2-of-2(publication_A, B)`
- PTLC-timeout: `2-of-2(A, publication_B)`

Note the following scenarios:

- Alice confirms a valid commitment transaction and the PTLC-success transaction confirms first: Bob immediately claims the funds since he knows both `publication_A` and `B`.
- Alice confirms a valid commitment transaction and the PTLC-timeout transaction confirms first: Alice waits for the relative timelock of her claim transaction and then broadcasts that. 
- Alice confirms a revoked commitment transaction: Bob can take the PTLC output right away since it's a `2-of-2(A,B)` and he can the secret for `A` from the revoked signature attached to the revoked transaction.
- Alice confirms a revoked commitment transaction and the PTLC-timeout transaction: Bob can take the PTLC-timeout output since he can extract `A` from the revoked signature attached to the revoked transaction.

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
