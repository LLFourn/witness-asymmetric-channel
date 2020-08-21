_This is draft email to the lightning-dev mailing list. If all goes well maybe it can also be a place iterate on the design._

# Background

As presently specified, the two parties in a lightning channel are assigned different commitment transactions.
This is logically necessary for the protocol to identify which party broadcasted the commitment transaction and potentially punish them if the other party provides provides proof it has been revoked (i.e. knows the revocation key).

Wouldn't it be nice if we could identify the broadcasting party without assigning them different transactions?
Riard first considered this problem in [8] and proposed that you could identify the party using _witness asymmetry_ i.e. both parties broadcast the same transaction but have different witnesses. 
Unfortunately, the solution they propose is rather convoluted.

More recently in [1], Aumayr et al. introduced a much more elegant witness asymmetric solution using adaptor signatures.
Instead of being assigned different transactions, the parties are assigned different adaptor signatures as witnesses for the _same_ transaction.
The adaptor signatures force the party broadcasting a commitment transaction to reveal a "publishing secret" to the other party.
The honest party can then use this publishing secret along with knowledge of the usual revocation secret to punish the malicious party for broadcasting an old commitment transaction.

In the paper this idea is combined with "punish-then-split" mechanism similar to the original eltoo[8] proposal and therefore it unfortunately inherits the issue with staggering time-locks.
BOLT 3 [4] notes this problem as design motivation:

> The reason for the separate transaction stage for HTLC outputs is so that HTLCs can timeout or be fulfilled even though they are within the to_self_delay delay. Otherwise, the required minimum timeout on HTLCs is lengthened by this delay, causing longer timeouts for HTLCs traversing the network.

More in depth commentary by Towns can be found in [2] and they propose a solution to the eltoo proposal in [3].

In short, the problem is that it creates a risk for the party that needs to fulfill an HTLC with the secret in time.
The only known way of accounting for this risk is to increase the difference between the time-locks on each hop.
There could be situations where this trade off makes sense but it seems like it undesirable for a general purpose payment channel network.

# Proposal

I propose a channel protocol which steals the main idea from Aumayr et al. while retaining the time-lock precedence of BOLT 3 transactions.
That is, absolute time-locks are settled before relative time-locks to avoid having to account for relative time-locks when calculating absolute time-locks.

The guiding principal is to keep commitment transactions symmetric but to use the asymmetric knowledge of the parties to simulate the way lightning works right now.
The main benefits of this seem to be:

- It is more elegant as there are half the number of possible transactions. I expect this will follow through to reduced implementation complexity and maybe make it easier to explain as well.
- Every output can just be a 2-of-2 multi-sig (which could be a single key with MuSig or OP_CHECKMULTISIG with ECDSA etc). This makes the transactions a little smaller even when using 2-of-2 with ECDSA on balance (I think). The space saving comes from avoiding explicit revocation keys in the commitment transaction outputs.
- The design works today and easily transitions into a post-taproot world although there is still a bit to explore there [5].

## Commitment transaction outputs

The way the fund transaction is constructed and negotiated remains unchanged.
For each commitment transaction parties exchange two curve points each:

- A _publication_ point (denoted P) whose secret is revealed if that party broadcasts the commitment transaction.
- A _revocation_ point (denoted R) whose secret is explicitly revealed to the other party when revoking this commitment transaction.

I'll use the notation (Pa, Ra) and (Pb, Rb) to refer to the publication and revocation points of Alice and Bob respectively and 2-of-2(X,Y) to refer to a multi-signature output that supports adaptor signatures and requires the owners of X and Y to authorise it (e.g. `OP_CHECKMULTISIG` with ECDSA [6], or MuSig with Schnorr etc [7]).

The scriptPubkey of all outputs on a commitment transaction is 2-of-2(Pa + Ra, Pb + Rb). 
For a slight increase in fungibility you can imagine that we pseudorandomly randomize each output so that they don't all look identical to blockchain observers.

When signing a commitment transaction, the parties instead exchange adaptor signatures such that the party who publishes the commitment transaction must reveal their publication secret.
This means that if a party broadcasts a commitment transaction for which they have already revealed their revocation secret the other party will know both secrets and therefore both secret keys in 2-of-2(Pa + Ra, Pb + Rb).
They can use this to unilaterally spend from all the outputs on the commitment transaction to punish the malicious party.

Note there is no problem with both parties broadcasting the same commitment transaction at the same time with their different signatures.
Both parties seeing the other's commitment transaction over the Bitcoin network actually acts as a kind of out of band optimization because it means they both learn each other's publication secrets and can therefore settle all the outputs without having to wait for relative time-locks (regardless of which signature makes it into the blockchain).

## Balance outputs

For a balance output going to Alice the output is 2-of-2(Pb, Ra) and the converse for Bob's balance output i.e. 2-of-2(Pa,Rb).
Each balance output has a pre-signed relative time-locked claim transaction spending it to an address owned by Alice.

## PTLCs

PTLCs are added directly to commitment transactions as outputs in the form described above.
Each PTLC has two transactions spending from it: PTLC-success and PTLC-timeout.
Broadcasting the PTLC-success transaction reveals the PTLC secret in the usual way using an adaptor signatures.
The PTLC-timeout transaction has an absolute time-lock on it through the nlocktime transaction value.

For a PTLC output being offered by Alice to Bob both transactions have a single output with the following scriptPubkeys: 

- PTLC-success: 2-of-2(Pa, Rb)
- PTLC-timeout: 2-of-2(Pb, Ra)

Both transactions have a pre-signed "claim" transaction that spends the funds to an address owned by the desired party after a relative time-lock (i.e. it has a sequence number set).
Alternatively, the PTLC-success/timeout can be constructed like HTLC-success/timeout transactions in lightning today where the destination party can spend unilaterally after the relative time-lock enforced with OP_CSV.
I find this slightly unappealing because it means using an output that is not just a 2-of-2.

Only the destination party needs to have the signature (or adaptor signature) on the PTLC-success or PTLC-timeout transactions.
So if it's a PTLC offered by Alice to Bob then Bob will have an adaptor signature on PTLC-success and Alice will have a ordinary signature on the PTLC-timeout transaction.
There is no theoretical security issue with both parties having the signatures it just unnecessary makes making writing state machines easier if Alice can rule out a PTLC-success transaction being broadcast unless she broadcasted it herself.

# Scenarios

To demonstrate the logical implications of the key combinations used in the above 2-of-2s let's work through some examples.
I'll use the label "BOB OWNS" to indicate that Bob knows both keys for the 2-of-2 and Alice has no pending time-locked transactions that spend from the output.
Likewise I'll use "BOB MUST SPEND" to indicate that Bob knows both keys but there *is* a pending time-locked transaction that Alice may use in the future.

In these examples Alice is the one broadcasting the commitment transaction but it could just as easily be Bob (remember it is the same transaction), in which case the converse of each situation applies.

## Balance output

Here is what happens for Balance outputs.

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
Bob can unilaterally spend the output because he knows all keys in 2-of-2(Pa + Ra, Pb + Rb) in this case.
If he misses out, he gets another shot after the PTLC-success and PTLC-timeout transactions are confirmed.

This is very similar to lightning today but here is what happens in the two scenarios for completeness:

Alice broadcasts a revoked commitment transaction which has a PTLC towards Bob (reveals Pa and has already revealed Ra):
1. PTLC-timeout confirmed: BOB MUST SPEND. The output is 2-of-2(Pa,Rb) and Alice revealed Pa when she broadcasted the commitment transaction but Alice has a pending claim transaction to spend the funds. 
2. Revocation tx confirmed: Bob takes all the funds.

Alice broadcasts a revoked commitment transaction which has a PTLC towards herself (reveals Pa and has already revealed Ra):
1. PTLC-success  confirmed: BOB MUST SPEND. The output is 2-of-2(Pb,Ra) and Alice revealed Ra when she revoked the commitment transaction but Alice has a pending claim transaction to spend the funds.
2. Revocation tx confirmed: Bob takes all the funds.

# Misc Remarks

- It looks straightforward to apply anchor outputs to this scheme 
- Each "claim" transaction can be `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` to allow combining it into a larger transaction.

Please let me know if something is not clear or underdeveloped or you think I have missed something. 
Thanks in advance for any ideas you have that could improve this scheme. 

[1]: https://eprint.iacr.org/2020/476.pdf 
[2]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-January/002448.html 
[3]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2015-November/000343.html
[4]: https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md
[5]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-December/002375.html 
[6]: https://joinmarket.me/blog/blog/schnorrless-scriptless-scripts/
[7]: https://joinmarket.me/blog/blog/flipping-the-scriptless-script-on-schnorr/_ 
[8]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-July/002064.html