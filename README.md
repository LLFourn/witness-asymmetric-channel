# Abstract

This a proposal for a new channel symmetric channel construction that uses the
key idea from a recent paper called "Generalized Bitcoin-Compatible Channels"[1]
and tries to practically apply it to lightning.

# Background

As presently specified the two parties in a lightning channel are assigned different commitment transactions.
This _transaction asymmetry_ is logically necessary for the protocol to identify which party broadcasted a commitment transaction and potentially punish them if the other party provides proof it has been revoked (i.e. knows the revocation key).

Wouldn't it be nice if we could identify the broadcasting party without assigning them different transactions?
Riard first considered this problem in [8] while trying to add a punishment mechanism to eltoo[9] style channel updates.
They proposed that you could identify the broadcasting party using _witness asymmetry_ i.e. both parties broadcast the same transaction but have different witnesses. 
Unfortunately, the solution proposed is rather convoluted.

More recently in [1], Aumayr et al. introduced a much more elegant witness asymmetric solution using adaptor signatures.
Instead of being assigned different transactions, the parties are assigned different adaptor signatures as witnesses for the _same_ transaction.
The adaptor signatures force the party broadcasting a commitment transaction to reveal a "publishing secret" to the other party.
The honest party can then use this publishing secret along with knowledge of the usual revocation secret to punish the malicious party for broadcasting an old commitment transaction.

Aumayr et al. combine this idea with a "punish-then-split" mechanism similar to the original eltoo proposal.
Unfortunately, it therefore inherits the same issue with staggering time-locks.
BOLT 3 [4] describes why it avoids this problem as a design choice:

> The reason for the separate transaction stage for HTLC outputs is so that HTLCs can timeout or be fulfilled even though they are within the to_self_delay delay. Otherwise, the required minimum timeout on HTLCs is lengthened by this delay, causing longer timeouts for HTLCs traversing the network.

There is further background discussion to this choice in [2] and Towns proposes a solution for eltoo in [3].

In short, the problem is that it creates a risk for the party that needs to fulfill an HTLC with the secret in time.
The only known way of accounting for this risk is to increase the difference between the time-locks on each hop.
There could be situations where this trade off makes sense but it seems undesirable for a general purpose payment channel network.

# Proposal

I propose a channel protocol which steals the main idea from Aumayr et al. while retaining the time-lock precedence of BOLT 3 transactions.
That is, absolute time-locks are settled before relative time-locks to avoid having to account for relative time-locks when calculating absolute time-locks.

The guiding principle is to keep commitment transactions symmetric but to use the asymmetric knowledge of the parties to simulate the way lightning works right now.
The main benefits of this seem to be:

- It is more elegant as there are half the number of possible transactions. I expect this will follow through to reduced implementation complexity and maybe make it easier to explain as well.
- Every output can just be a 2-of-2 multi-sig (which could be a single key with MuSig or OP_CHECKMULTISIG with ECDSA etc).
  This makes the transactions a little smaller even when using 2-of-2 with ECDSA on balance, The space saving comes from avoiding explicit revocation keys in the commitment transaction outputs and using PTLCs rather than HTLCs.
- The design works today and easily transitions into a post-taproot world although there is still a bit to explore there [5].

## Notation

I describe the protocol with respect to two parties Alice and Bob.
2-of-2(Xa,Xb) refers to a multi-signature output that supports adaptor signatures and requires the owners of Xa (Alice) and Xb (Bob) to authorize it (e.g. `OP_CHECKMULTISIG` with ECDSA [6], or MuSig with Schnorr etc [7]).

## The Fund transaction

The structure of channel funding does not change from the current BOLT spec.
The Fund transaction spends from the funding party's (or funding _parties_) outputs to an output of 2-of-2(Fa,Fb) where Fa and Fb are public keys owned by Alice and Bob respectively. 
These keys are generated and exchanged once per channel.

## Commitment transactions

All commitment transactions spend from the 2-of-2 on the Fund transaction and each represents a "state" of the channel.
A commitment transaction has two balance outputs and zero or more PTLC outputs.

Before creating a commitment transaction the parties must exchange two curve points each:

- A _publication_ point (denoted P) whose secret is revealed if that party broadcasts the commitment transaction.
- A _revocation_ point (denoted R) whose secret is explicitly revealed to the other party when revoking this commitment transaction.

Let (Pa, Ra) and (Pb, Rb) be the publication and revocation points of Alice and Bob respectively.
The scriptPubkey of all outputs on a commitment transaction is 2-of-2(Pa + Ra, Pb + Rb). 
To make it slightly harder to distinguish these outputs from other uses of 2-of-2 you can imagine that we pseudorandomly randomize each output so that they are not identical.

When signing a commitment transaction, the parties instead exchange adaptor signatures such that the party who publishes the commitment transaction must reveal their publication secret.
This means that if a malicious party broadcasts a commitment transaction for which they have already revealed their revocation secret the other party will know both secrets and therefore both secret keys in 2-of-2(Pa + Ra, Pb + Rb).
They can use this to unilaterally spend from all the outputs on the commitment transaction to punish the malicious party.

As a point of interest there is no problem with both parties broadcasting the same commitment transaction at the same time with their different signatures.
In theory, both parties seeing the other's commitment transaction over the Bitcoin network could act as a kind of out of band optimization because it means they both learn each other's publication secrets and can therefore settle all the outputs without having to wait for relative time-locks (regardless of which signature makes it into the blockchain).

## Balance outputs

A balance output going to Alice has a scriptPubkey of 2-of-2(Pb, Ra) and the converse for Bob's balance output i.e. 2-of-2(Pa,Rb).
Each balance output has a pre-signed relative time-locked claim transaction spending it to an address owned by the deserving party.

## PTLCs

PTLC outputs are added directly to commitment transactions in the form described above.
Each PTLC has two pre-signed transactions spending from it: PTLC-success and PTLC-timeout.
Broadcasting the PTLC-success transaction reveals the PTLC secret in the usual way using an adaptor signatures.
The PTLC-timeout transaction has an absolute time-lock on it through the nlocktime transaction value.

For a PTLC output being offered by Alice to Bob both transactions have a single output with the following scriptPubkeys: 

- PTLC-success: 2-of-2(Pa, Rb)
- PTLC-timeout: 2-of-2(Pb, Ra)

Both transactions have a pre-signed "claim" transaction that spends the funds to an address owned by the desired party after a relative time-lock (i.e. it has a sequence number set).
Alternatively, the PTLC-success/timeout can be constructed like HTLC-success/timeout transactions in lightning today where the destination party can spend unilaterally after the relative time-lock enforced with OP_CSV.
I find this slightly unappealing because it means using an output that is not just a 2-of-2.

Only the destination party needs to have the signature (or adaptor signature) on the PTLC-success or PTLC-timeout transactions.
So if it's a PTLC offered by Alice to Bob then Bob will have an adaptor signature on PTLC-success and Alice will have an ordinary signature on the PTLC-timeout transaction.
There is no theoretical security issue with both parties having the signatures it just makes writing state machines easier if Alice can rule out a PTLC-success transaction being broadcast unless she broadcasted it herself.

# Scenarios

To demonstrate the logical implications of the key combinations used in the above 2-of-2s I'll somewhat tediously go through how each output is resolved.
I'll use the label "BOB OWNS" to indicate that Bob knows both keys for the 2-of-2 and Alice has no pending time-locked transactions that spend from the output.
Likewise I'll use "BOB MUST SPEND" to indicate that Bob knows both keys but there *is* a pending time-locked transaction that Alice may use in the future.

In these examples Alice is the one broadcasting the commitment transaction but it could just as easily be Bob (remember it is the same transaction), in which case the converse of each situation applies.

## Balance output

Here is what happens for balance outputs.

Alice broadcasts a non-revoked commitment transaction (reveals Pa):
1. Balance output for Alice: Alice waits for the relative time-lock and then broadcasts her claim transaction.
2. Balance output for Bob: BOB OWNS. The output is 2-of-2(Pa,Rb) and Alice has revealed Pa.

Alice broadcasts a revoked commitment transaction (reveals Pa and has already revealed Ra):
1. Balance output for Alice: BOB MUST SPEND. The output is 2-of-2(Ra,Pb) so Bob knows both keys but he needs to spend it before Alice's relative time-locked claim transaction.
2. Balance output for Bob: BOB OWNS. The output is 2-of-2(Pa,Rb) and Alice has revealed Pa.

## PTLC output

In each of the following scenarios Alice broadcasts the commitment transaction and therefore reveals Pa.
I describe what happens when the PTLC-success transaction makes it in instead of the PTLC-timeout and vise versa.

Alice broadcasts a non-revoked commitment transaction which has a PTLC offered to Bob (reveals Pa):
1. PTLC-success confirmed: BOB OWNS. The output is 2-of-2(Pa,Rb) and Alice revealed Pa when she broadcasted the commitment transaction.
2. PTLC-timeout confirmed: Alice waits until the relative time lock expires and then broadcasts the claim transaction.

Alice broadcasts a non-revoked commitment transaction which has a PTLC offered to herself (reveals Pa):
1. PTLC-success confirmed: Alice waits for the relative time-lock to expire and then broadcasts the claim transaction.
2. PTLC-timeout confirmed: BOB OWNS. The output is 2-of-2(Pa,Rb) and Alice revealed Pa when she broadcasted the commitment transaction.

When Alice broadcasts a revoked commitment with a PTLC either Alice manages to also confirm the PTLC-success or PTLC-timeout transactions (depending on the direction of the PTLC) or Bob revokes the PTLC output in a transaction of his choosing.
Bob can unilaterally spend the PTLC output because he knows all keys in 2-of-2(Pa + Ra, Pb + Rb).
If he misses out, he gets another shot after the PTLC-success and PTLC-timeout transactions are confirmed.
This is very similar to lightning today but here is what happens in the two scenarios for completeness:

Alice broadcasts a revoked commitment transaction which has a PTLC towards Bob (reveals Pa and has already revealed Ra):
1. PTLC-timeout confirmed: BOB MUST SPEND. The output is 2-of-2(Pb,Ra) and Alice revealed Ra when she revoked the commitment transaction but Alice has a pending claim transaction to spend the funds. 
2. Revocation tx confirmed: Bob takes all the funds.

Alice broadcasts a revoked commitment transaction which has a PTLC towards herself (reveals Pa and has already revealed Ra):
1. PTLC-success confirmed: BOB MUST SPEND. The output is 2-of-2(Pb,Ra) and Alice revealed Ra when she revoked the commitment transaction but Alice has a pending claim transaction to spend the funds.
2. Revocation tx confirmed: Bob takes all the funds.

# Misc Remarks

- It looks straightforward to apply anchor outputs to this scheme 
- Each "claim" transaction can be `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` to allow combining it into a larger transaction.

[1]: https://eprint.iacr.org/2020/476.pdf 
[2]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2015-November/000339.html 
[3]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-January/002448.html  
[4]: https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md
[5]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-December/002375.html 
[6]: https://joinmarket.me/blog/blog/schnorrless-scriptless-scripts/
[7]: https://joinmarket.me/blog/blog/flipping-the-scriptless-script-on-schnorr/_ 
[8]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-July/002064.html
[9]: https://blockstream.com/eltoo.pdf


Please let me know if something is not clear or underdeveloped or you think I have missed something. 
Thanks to those who have offered their comments so far and thanks in advance for any ideas you have that could improve this scheme. 
