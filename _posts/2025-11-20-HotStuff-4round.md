
# From Hidden Locks to Liveness Deadlocks: A Deep Dive into HotStuff’s Three-Chain Commitment

In the design of BFT consensus protocols, we often hear that HotStuff introduced the "Three-phase Commit" to solve the liveness problem. However, this term can be misleading. **The "hidden" aspect refers not only to the Leader's inability to see the lock but also to a systemic deadlock where nodes "want to unlock but dare not."**

This article dissects this classic Liveness issue and explains how HotStuff resolves it through simple yet profound rule adjustments.

## 1. Anatomy of a Deadlock: The 1 vs 2f Dilemma

To understand the essence of the problem, we must first reconstruct the extreme scenario that causes the protocol to stall. This typically occurs in two-phase or naive three-phase protocols where the total number of nodes is $N=3f+1$.

Consider a situation where, after a specific View, the system state splits as follows:
* **1 honest node** is locked on branch $B$ (it will only vote for $B$ or its successors);
* **2f honest nodes** are locked on branch $B'$ (they will only vote for $B'$ or its successors);
* **f Byzantine nodes** decide to act maliciously by remaining silent and refusing to vote for either side.

This constitutes a typical **Liveness Deadlock**. A quorum usually requires $2f+1$ votes. However, in this scenario:
* For branch $B$: Even if the single honest node and all Byzantine nodes (hypothetically) voted, the total is $1+f$, which is less than $2f+1$.
* For branch $B'$: Although the $2f$ honest nodes are the majority, due to the silence of the Byzantine nodes, they can never reach the $2f+1$ threshold.

Consequently, the system enters a stalemate: the single node stubbornly holds onto $B$, the $2f$ nodes stubbornly hold onto $B'$, and neither side can convince the other. This deadlock is the macroscopic manifestation of the **Hidden Lock Problem**.

## 2. Why is it Called a "Hidden Lock"?

The phenomenon is termed "Hidden Lock" because it involves two layers of "unknowns":

**Layer 1: Invisible to the Leader**
During a View Change, the Leader only needs to collect `NEW-VIEW` messages from a quorum of $N-f$ (i.e., $2f+1$) nodes to start a new round. In the scenario described above, if that single honest node locked on $B$ happens to be excluded from this quorum, the Leader will mistakenly believe there are no locks in the system. It will then proceed to propose $B'$, creating a conflict with the old lock.

**Layer 2: Unprovable to the Lock Holder**
For the node locked on $B$, it faces a dilemma. It knows it has locked $B$, but it cannot ascertain whether $B$ has been committed globally. Without proof that $B$ has been discarded or superseded, it dares not unlock to vote for $B'$, as doing so could lead to a Safety violation (i.e., a Double Commit).

This is the true lethality of the Hidden Lock: **The Leader cannot see the lock and thus cannot provide the evidence needed to make the locked node "give up"; the locked node, lacking this evidence, must stand its ground.** This information asymmetry results in the $1 \text{ vs } 2f$ deadlock.

## 3. HotStuff’s Solution: PreCommit Locks & PrepareQC

HotStuff’s core innovation relies not on complex cryptography, but on adjusting *when* to lock and *what* to transmit. It introduces a third phase and mandates that: **Nodes only lock during the second phase (PreCommit), but they propagate the certificate from the first phase (PrepareQC) during a View Change.**

This change cleverly exploits the intersection property of Quorums to force locks to "reveal themselves."

**1. Where there is a Lock, there are Witnesses**
If an honest node is locked on block $B$ (holding a `precommitQC`), it mathematically implies that in the previous round's Prepare phase, $2f+1$ nodes voted for $B$. Within these $2f+1$ nodes, even after subtracting $f$ potential Byzantine nodes, there are at least **$f+1$ honest nodes** that fully participated in the Prepare phase and hold the `prepareQC(B)`.

**2. The Honest Leader Must See the Traces**
When a new honest Leader collects $2f+1$ `NEW-VIEW` messages, by the Pigeonhole Principle, the source of these messages must intersect with the group of $f+1$ honest witnesses mentioned above. In other words, **the set of messages collected by the Leader must contain `prepareQC(B)`.**

This means that for an honest Leader, the lock is no longer hidden. The Leader will observe the highest `prepareQC` and propose based on it. The previously "stubborn" locked node will see that the new proposal extends the block it locked on and will vote for it. Other unlocked nodes, seeing that the proposal is justified by the highest QC, will also vote according to safety rules. System liveness is thus restored.

Why should a node follow and vote for the highest QC observed in the network? The reasoning is as follows: If any node has successfully committed a message, it implies that at least $f+1$ honest nodes must hold a lockedQC for it. These honest nodes would prevent the formation of any higher QC that does not extend the committed message. Therefore, the very existence of a higher QC that does not extend my own lockedQC serves as proof that my locked block was never committed.

## 4. Dealing with Malicious Leaders

Since the lock information is exposed, what happens if the next Leader is malicious and deliberately ignores the `prepareQC(B)` to propose a conflicting branch $B'$?

In this case, progress will indeed be temporarily stalled, but safety is preserved, and a permanent deadlock is avoided. The node locked on $B$ will refuse to vote for $B'$, preventing $B'$ from reaching consensus immediately. However, HotStuff does not rely on malicious leaders to make progress; it only guarantees that **the system will recover as soon as the next honest Leader takes charge.**

Even if, under the malicious Leader's influence, some nodes switch to $B'$ and form a new lock, the same logic applies: the new lock on $B'$ will result in at least $f+1$ honest nodes holding the `prepareQC` for $B'$. When the next honest Leader arrives, it will see that $B'$ has a higher View and propose based on $B'$. At this point, the old node locked on $B$ can unlock and switch to $B'$ based on the Liveness rule (`newView > lockedView`).

**In summary, HotStuff utilizes the "Three-phase + Propagate PrepareQC" mechanism to ensure that any effective lock leaves enough "witnesses" across the network. These witnesses guarantee that come what may, an honest Leader will always perceive the highest lock in the system via the View Change, thereby eliminating the risk of deadlocks caused by Hidden Locks.**