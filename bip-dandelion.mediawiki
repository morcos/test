<pre>
BIP: ?
Title: Dandelion: Privacy-Preserving Transaction Propagation
Author: Giulia Fanti <gfanti@andrew.cmu.edu>
        Andrew Miller <soc1024@illinois.edu>
        Surya Bakshi <sbakshi3@illinois.edu>
        Shaileshh Bojja Venkatakrishnan <bjjvnkt2@illinois.edu>
        Pramod Viswanath <pramodv@illinois.edu>
Comments-URI: ?
Status: Draft
Type:Standards Track
Created: 2017-06-09
License: PD
</pre>

== Abstract ==

Dandelion is a new transaction broadcasting mechanism that reduces the risk of eavesdroppers linking transactions to the source IP.

Dandelion transaction propagation proceeds in two phases: first the “stem” phase, and then “fluff” phase. During the stem phase, each node relays the transaction to a *single* peer. After a random number of hops along the stem, the transaction enters the fluff phase, which behaves just like ordinary flooding/diffusion. Even when an attacker can identify the location of the fluff phase, it is much more difficult to identify the source of the stem.

Illustration:
<pre>
                                                   ┌-> F ...
                                           ┌-> D --┤
                                           |       └-> G ...
  A --[stem]--> B --[stem]--> C --[fluff]--┤
                                           |       ┌-> H ...
                                           └-> E --┤
                                                   └-> I ...
</pre>

== Motivation ==

Bitcoin transaction propagation does not hide the source of a transaction very well, especially against a “supernode” eavesdropper that forms a large number of outgoing connections to reachable nodes on the network [1,2,3]. From the point of view of a supernode, the peer that relays the transaction *first* is the most likely to be the true source, or at least a close neighbor of the source. Various application-level behaviors of Bitcoin Core enable eavesdroppers to infer the peer-to-peer connection topology, which can further help identify the source [2,3].

The Dandelion protocol obscures the source by propagating each transaction along the stem (away from the source), before initiating the diffusion process at a safe distance. Dandelion was introduced in [4] and presented at ACM Sigmetrics 2017. This BIP discusses a more practical and robust variant of Dandelion called Dandelion++ (research article forthcoming [5]). For the sake of this document, we ignore the ++ distinction and just call the proposal Dandelion overall. 

Dandelion exhibits two key properties: first, its anonymity properties are near-optimal among propagation mechanisms that do not obfuscate transactions (e.g., using encryption). The intuition is that under Dandelion, transactions from different nodes generate statistically similar propagation patterns; hence, adversarial nodes will be unable to use those propagation patterns to reliably infer the source IP. The relevant analysis and proofs are included in [4]. Second, Dandelion does not significantly increase transaction propagation latency. Dandelion's latency overhead (compared to the status quo) is equal to the time spent in stem phase. Although [4] does not measure this latency explicitly, our simulations suggest that the average overhead will be on the order of seconds, and we are currently running experiments to empirically measure this latency distribution. 


== Specification ==

The Dandelion protocol is based on three mechanisms:

1. *Stem/fluff propagation.* Dandelion transactions begin in “stem mode,” during which each node relays the transaction to a single randomly-chosen peer. With some fixed probability, the transaction transitions to “fluff” mode, after which it is relayed according to ordinary Bitcoin flooding/diffusion.

2. *Mempool Embargo.* During the stem phase, each stem node (Alice) stores the transaction in an “embargoed” state. During the embargo period, Alice behaves as though she has not seen the transaction. That is, Alice will not include the embargoed transaction when responding to <code>MEMPOOL</code> requests, and will not respond to <code>GETDATA</code> requests from another node (Bob) unless Alice previously sent an <code>INV</code> to Bob. The embargo period ends as soon as Alice receives an <code>INV</code> advertising the transaction as being in fluff mode.

3. *Robust propagation.* Privacy enhancements should not put transactions at risk of not propagating. To protect against failures (either malicious or accidental) where a stem node fails to relay a transaction (thereby precluding the fluff phase), each node starts a random timer upon receiving a transaction in stem phase. If the node does not receive any INV messages for that transaction before the timer expires, then the embargo ends and the node diffuses the transaction normally.

Dandelion stem mode transactions are indicated by a new type of inventory item and a new transaction type.

New Dandelion transaction inventory type:
<pre>
MSG_DANDELION_TX = 5
MSG_WITNESS_DANDELION_TX = MSG_DANDELION_TX | MSG_WITNESS_FLAG
</pre>

Dandelion transaction message type:
<pre>
NetMsgType::DANDELIONTX;
</pre>

After receiving a Dandelion transaction, the node flips a biased coin to determine whether to propagate it in “stem mode”, or to switch to “fluff mode.” The bias is controlled by a parameter exposed to the command line, initially 90% chance of staying in stem mode (meaning the expected stem length would be 10 hops).
<pre>
-dandelion=<n>
Configure Dandelion (privacy-preserving transaction propagation) stem 
probability percent (default: 90, max 100, 0 to disable)
</pre>

We have evaluated a spectrum of possible schemes for selecting which peers receive relayed stem transactions. We call the set of possible relays the “stem set”. On one extreme, a node could relay every stem transaction to the same peer (i.e., the stem set has size 1). On the other extreme, a node could randomly choose a new relay from its existing peers for every transaction. The tradeoffs between these options affect how much “precision” an attacker gets, and depend on how easily the attacker can infer the connection topology of the Dandelion-relay overlay. Our compromise is to maintain a stable, randomly-chosen stem set of *two* peers, and to select one randomly each time we relay a transaction. The stem set is chosen from among the outgoing (or whitelisted) connections, which prevents an adversary from easily inserting itself into the stem graph. Each node periodically re-randomizes its stem set every 10 minutes.

Service bits:
Support for Dandelion is indicated by a temporary experimental service bit.
<pre>
NODE_DANDELION = (1 << 24)
</pre>
We imagine that in the future, the service will be discovered in-band by sending “MSG_DANDELION_TX” after the hand-shake.


== Considerations ==
The main implementation challenges are: (1) identifying a satisfactory tradeoff between Dandelion’s privacy guarantees and its latency/overhead, and (2) ensuring that privacy cannot be degraded through abuse of existing mechanisms. In particular, the implementation should prevent an attacker from identifying stem nodes without interfering too much with the various existing mechanisms for efficient and DoS-resistant propagation.

* The privacy afforded by Dandelion depends on 3 parameters: the stem probability, the number of outbound peers that can serve as dandelion relays (i.e., the “stem set” size), and the time between re-randomizations of the stem set. These parameters define a tradeoff between privacy and broadcast latency/processing overhead. Lowering the stem probability harms privacy but helps reduce latency by shortening the mean stem length; based on theory, simulations, and experiments, we have chosen a default of 90%. Lowering the stem set size (from a default size of 2 to 1) makes dandelion’s privacy guarantees fragile to adversaries that can learn the stem set of each node; here we choose robustness over optimal privacy. Reducing the time between each node’s re-randomization of its stem set reduces the chance of an adversary learning the stem sets for each node, at the expense of increased overhead. These tradeoffs are outlined more precisely in a forthcoming article [5].
* When receiving a Dandelion inventory item, the request skips the usual “mapAlreadyAskedFor” priority queue, which adds a 2-minute timer before asking the next node. Otherwise, an eavesdropper would be able to probe whether a node has received stem transaction <code>tx</code> by sending <code>INV(tx)</code> and checking whether or not <code>GETDATA(tx)</code> is received in response.
* When receiving or asking for a Dandelion stem transaction, we avoid placing that transaction in <code>filterInventoryKnown</code>. This way, transactions can also travel back “up” the stem in the fluff phase.
* Like ordinary transactions, Dandelion transactions are only relayed after being successfully accepted to mempool. This ensures that nodes will never be punished for relaying Dandelion transactions, and that existing replace-by-fee and fee-filter behavior is preserved.
* If an orphan transaction is received in Dandelion mode, it is added to <code>mapOrphanTransactions</code>, and also marked as stem-mode. If the transaction is later accepted to mempool, then it is relayed as a Dandelion transaction (either stem mode or fluff mode, depending on a coin flip).
* If a node receives a child transaction that depends on one or more currently-embargoed Dandelion transactions, then the transaction is also relayed in stem mode, and the embargo timer is set to the maximum of the embargo times of its parents. This helps ensure that parent transactions enter fluff mode before child transactions.
* Transaction propagation latency should be minimally affected by opting-in to this privacy feature; in particular, a transaction should never be prevented from propagating at all because of Dandelion. The random timer guarantees that the embargo mechanism is temporary, and every transaction is relayed according to the ordinary diffusion mechanism after some maximum (random) delay on the order of 30-60 seconds.
* Despite the best effort of the embargo mechanism, it is difficult to ensure that an attacker cannot find some other indirect way to probe whether a node has participated in the stem phase for a transaction. For example, an attacker seeing <code>tx1</code> could create an invalid transaction <code>tx2</code> that purports to spend an output from <code>tx1</code>, and send it to victim node Alice. If Alice has placed <code>tx1</code> in mempool, then Alice will reject the attacker and disconnect it; if Alice has not seen <code>tx1</code>, then <code>tx2</code> would instead go to the orphan cache. Fortunately, this example at least requires the attacker to burn one of its outgoing connections. Here we’re favoring an imperfect solution over something more complicated to implement.


== Backward compatibility ==

Dandelion is intended for gradual deployment and adoption, with privacy gains that increase monotonically with the fraction of adopting nodes. Theoretical analysis shows that at any adoption level, the privacy of Dandelion nodes will be better than the status quo, and non-Dandelion nodes will have privacy no worse than the status quo [5]. To achieve these results, we rely on two implementation decisions:

1. *Fluff mode by default.* Nodes without Dandelion support will continue to relay transactions with independent exponential delays. Hence, if a Dandelion node extends the stem to a non-Dandelion node, it is as if the transaction automatically enters the fluff phase. Thus, the fewer nodes that support Dandelion, the shorter the average stem length.

2. *Avoidance of self-reporting.* Nodes do not consider the Dandelion service bit when choosing which nodes to connect to or which nodes to relay to. Otherwise, during the initial gradual adoption period, this could give preference to attackers that signal Dandelion support.


== Implementation ==

A reference implementation of Dandelion is available at https://github.com/gfanti/bitcoin/tree/dandelion

This implementation includes a modification to the wallet software that relays newly-created transactions as Dandelion transactions.

An incomplete testing harness, `p2p-dandelion.py`, is also included.


== Analysis ==

This BIP references some theoretical analysis that is work in progress, e.g., regarding the anonymity guarantees of partially-deployed Dandelion. The following plot shows theoretical results on the expected recall (probability of linking a transaction to an IP address) as a function of the fraction of corrupt nodes. 

[[File:dandelion-analysis.png|40px|Link=]]

Here we are assuming an adversarial model where a constant fraction of colluding nodes passively observe all transactions. These "spy" nodes try to infer the source of each transaction using observed timestamps, knowledge of the graph, and knowledge of which node delivered each transaction to each spy node. ‘Version-checking’ means that a node preferentially adds dandelion-compatible peers to its stem set. ‘No-version-checking’ means that each Dandelion node chooses its stem set without regards for dandelion-compatibility (as recommended in this BIP). The plot shows theoretical upper and lower bounds on the expected recall for *Dandelion nodes* that use each strategy, so the true expected recall for version-checking (resp. no-version-checking) is somewhere in the blue (resp. red) region. Non-Dandelion nodes will experience no change in their expected recall. These results suggest that no-version-checking is a better strategy, which also outperforms the simulated status quo (diffusion), even when deployment levels are low. For the simulated performance of diffusion, we used the suboptimal 'first-spy' estimator, where for each transaction, the adversary implicates the first honest node to deliver that transaction to any spy node. This gives a lower bound on the adversary's deanonymization power with respect to diffusion.

== Acknowledgements ==

Gregory Maxwell provided us with invaluable feedback and suggestions that guided our implementation approach.

== References ==

1. An Analysis of Anonymity in Bitcoin Using P2P Network Traffic http://fc14.ifca.ai/papers/fc14_submission_71.pdf

2. Deanonymisation of clients in Bitcoin P2P network https://arxiv.org/abs/1405.7418

3. Discovering Bitcoin’s Public Topology and Influential Nodes https://cs.umd.edu/projects/coinscope/coinscope.pdf

4. (Sigmetrics 2017) Dandelion: Redesigning the Bitcoin Network for Anonymity https://arxiv.org/abs/1701.04439

5. Dandelion++: TBA


== Copyright ==

This document is placed in the public domain.
