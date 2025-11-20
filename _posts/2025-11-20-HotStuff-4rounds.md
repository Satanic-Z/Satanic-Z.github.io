

# PBFT’s Quadratic View-Change, Why HotStuff’s 3 Rounds Aren't Enough, and What 4 Rounds Actually Solve

This article discusses a very simplified world: the most basic version of HotStuff. We will not consider pipelining or various engineering optimizations. Instead, we will focus solely on a few core concepts: **lockQC**, **commitQC**, and **highQC**, as well as the differences in safety and liveness between PBFT's quadratic view-change, **2-chain + 1QC**, and **3-chain + 1QC**.



### First, Let’s Clarify the Model We Are Discussing

We assume a standard $n>3f$ model, with at most $f$ Byzantine nodes. In the protocol, nodes vote on a block, and gathering $n-f$ votes forms a Quorum Certificate (QC).

Every node maintains a local **lockQC**, indicating which block it is currently locked on. Once locked on block $B$, according to protocol constraints, the node will not vote for any block conflicting with $B$ unless a specific "unlock condition" is met. On the other hand, there is the **commitQC**, representing that a block has met the commit conditions and can be committed locally. During a view-change, the new leader selects a **highQC** to decide which block to extend in the new view.

Within this basic framework, we will first look at **2-chain + 1QC** (what you refer to as the "three-round version of HotStuff"), then look at how PBFT's `new-view` supplements information, and finally examine how **3-chain + 1QC** (the "four-round version of HotStuff") resolves the same pitfall without relying on PBFT-style `new-view` messages.

---

### 2-chain + 1QC: Two Critical Scenarios

In the **2-chain + 1QC** model, roughly speaking, you can commit $B$ when you see a QC for $B$ and a QC for $child(B)$. The local lockQC updates when it sees a QC for a certain round. Under this model, there are two scenarios we cannot bypass.

The first scenario is: a node is already locked on $B$, and at this moment, a new `lockQC(B')` appears in the network.
The second scenario is: a node is already locked on $B$, but the leader proposes a new block $B'$ that does not extend $B$, but rather extends $B$'s parent (i.e., a sibling branch of $B$).

Let’s look at them individually.

#### Scenario 1: Existing lock(B), yet a new lock(B') appears

Let's assume a node is currently locked on block $B$, meaning its local `lockQC = B`. Meanwhile, a new `lockQC(B')` appears in the network, where $B'$ conflicts with $B$.

Here lies a simple but crucial mathematical fact: If a `commitQC` for $B$ exists, then at least $f+1$ honest nodes must have locked their votes for the round corresponding to $B$. Based on the locking rules, these $f+1$ honest nodes will not vote for a conflicting block like $B'$ subsequently. Therefore, for $B'$ to form its own `lockQC(B')`, it needs $n-f$ votes. However, those $f+1$ honest votes will never be given to it, leaving at most $2f$ votes available. Thus, it is impossible for $B'$ to gather $n-f$ votes, and `lockQC(B')` cannot exist.

Reasoning in reverse: if a new `lockQC(B')` does indeed exist, it implies that a `commitQC` for $B$ does **not** exist. In other words, the very fact of seeing "there is a new lockQC for $B'$" proves that $B$ has not been committed. Consequently, the node locked on $B$ can safely switch to $B'$. From a safety perspective, whether it continues to stick with $B$ or switches to $B'$ at this moment, neither choice will lead to a double commit.

The problem lies in **liveness**. Although logically one *can* switch to $B'$, in actual execution, a malicious node can manipulate who the `highQC` is—for example, by keeping `highQC = B` for several subsequent rounds. While nodes locked on $B$ theoretically *could* switch camps, if the leader consistently uses $B$ as the justification, the system might remain stuck on the $B$ line for a long time. We cannot rely on this "switch if you see lockQC(B')" rule to truly guarantee liveness.

#### Scenario 2: Existing lock(B), leader proposes new block B' extending parent(B)

The real trouble is Scenario 2. Here, a node is already locked on $B$ with a local `lockQC(B)`. A new leader proposes block $B'$, which does not extend $B$, but extends $B$'s parent (a sibling block). $B$ and $B'$ are conflicting, but $B'$ relies on the QC of $parent(B)$.

Faced with this $B'$, the node locked on $B$ falls into a dilemma.

On one hand, this node does not know if there is a `precommitQC` or `commitQC` for $B$ anywhere in the network. From its perspective, messages might still be in transit; others might have voted for $B$ again, or even formed a `commitQC` for $B$ that it hasn't seen yet. If it votes for $B'$ now, a double commit might occur in some execution path: $B$ is being committed elsewhere, while this node participates in committing $B'$.

On the other hand, if it resolutely refuses to vote for $B'$, it is possible that no one can truly commit $B$. Suppose the leader repeatedly proposes this kind of sibling block extending $parent(B)$, and the nodes locked on $B$ consistently refuse to cooperate. The remaining nodes (at most $2f$ plus Byzantine nodes) may never gather $n-f$ votes. The system could stall indefinitely due to this lock, losing liveness.

Therefore, in the **2-chain + 1QC** model, regarding the scenario "Existing lock(B), leader proposes B' extending parent(B)," the node locked on $B$ simply does not have enough local information to decide whether to vote or refuse: voting risks a double commit, while refusing risks a loss of liveness. **This is the core dilemma of 2-chain + 1QC.**

---

### PBFT: What exactly are the n-f messages in new-view solving?

PBFT's `new-view` mechanism exists precisely to solve this "insufficient local information" problem. The key lies in those $n-f$ messages within the `new-view`.

In PBFT, during a view-change, every node sends a `VIEW-CHANGE` message, packaging its locally seen prepared/locked state (which you can understand as the pre-commit layer) to the new leader. After collecting $n-f$ `VIEW-CHANGE` messages, the new leader selects a **highQC** from them—the "highest" chain in the global view.

For a node locally locked on `lockQC(B)`, these $n-f$ messages help confirm a critical fact: **In this "n-f node view" of the entire network, does a commitQC for B exist, and what is the highest QC exactly?**

* If the `highQC` aggregated by the `new-view` is $B$, it means that in the global perspective of this view-change, $B$ is still the "highest" chain most likely to be committed. It is safe for the node to continue voting for $B$ because the `new-view` has agreed to proceed on $B$'s branch.
* If the `highQC` aggregated by the `new-view` is $B'$ or $B$'s parent, the situation is reversed. This implies the view of these $n-f$ nodes tells you: there is no `commitQC` for $B$ in the network; no one has seen evidence sufficient to commit $B$. Therefore, the node can confidently vote for $B'$ and transfer its lock from $B$ to $B'$ without the risk of double commitment.

In other words, the $n-f$ messages in PBFT's `new-view` essentially help a locally locked node answer one question: **Is there a commitQC for B anywhere in the network?** A single node cannot see this from its local view, but aggregating $n-f$ `VIEW-CHANGE` messages allows the new leader to select a `highQC` that effectively represents a "safe direction confirmed by the network." Thus, for the aforementioned $B'$ that "extends parent(B)," PBFT uses the `highQC` selection to let nodes know clearly whether they should continue voting for $B$ or safely switch to $B'$.

---

### 3-chain + 1QC: How does HotStuff’s "Four-Round" Approach Handle the Same Issue?

Now let's return to HotStuff, looking only at the simplest version without pipelining. Consider **3-chain + 1QC**, which you refer to as the "four-round version of HotStuff."

We continue to focus on that troublesome Scenario 2: a node is locked on $B$, and the leader proposes a new block $B'$ extending $parent(B)$ instead of $B$. In 2-chain + 1QC, the node struggled with whether to vote for $B'$. In **3-chain + 1QC**, the situation changes.

Under the **3-chain + 1QC** rules, the `safeNode` predicate naturally utilizes the relationship of view numbers. For a node already locked on $B$, as long as the new block $B'$'s `justify.view` does not exceed its current lock's view, and $B'$ does not extend $B$, it will **not** vote for $B'$. Regarding the $B'$ that extends $parent(B)$, its justification comes from $parent(B)$, so the view number is necessarily no greater than $B$'s. Consequently, the node will simply not vote for $B'$.

This means that for Scenario 2, under the 3-chain + 1QC framework, the question of "whether to vote for B'" is blocked from the start by the view number comparison. It is no longer a problem that requires local guessing.

From here, 3-chain + 1QC allows for two paths of development:

1.  If `lockQC(B')` is never formed, it means this branch cannot truly be locked, and nodes locked on $B$ will never vote for it. Eventually, when an honest leader proposes a new block extending $B$, everyone can continue to advance on $B$'s branch. This corresponds to: "B' didn't lock, so we wait for a correct leader to extend B."
2.  If `lockQC(B')` *is* indeed formed at some point, this process itself carries information. Since nodes locked on $B$ would not vote for $B'$, the formation of `lockQC(B')` implies that other nodes (including potential malicious ones) voted for it. Furthermore, the existence of `lockQC(B')` implies that **no node has committed B**. Once there is a `commitQC` for $B$, at least $f+1$ honest nodes are locked tight on $B$ and will not vote for $B'$; thus, by simple counting, `lockQC(B')` could not form.

Therefore, the event "`lockQC(B')` is generated" itself proves the fact: **There is no commitQC for B in the network.**

Combined with the fact that $B'$'s view number is greater than $B$'s view number, this satisfies a very natural unlock condition for the node locked on $B$: *A lockQC with a higher view has appeared, corresponding to a different branch B', and B has not been committed.* In this case, the node can safely transfer its lock from $B$ to $B'$.

So, in the **3-chain + 1QC** model, Scenario 2 is split into two stages:
* **Before** `lockQC(B')` appears, the view number naturally prevents nodes locked on $B$ from voting for $B'$.
* **After** `lockQC(B')` appears, the existence of $B'$ itself proves $B$ was not committed, and since $B'$ has a higher view, nodes locked on $B$ are allowed to safely switch.

The entire process achieves a similar information filtering effect implicitly through view numbers and locking rules, without needing to explicitly collect $n-f$ `new-view` messages like PBFT.

---

### Summary: The Relationship Between PBFT, 2-chain, and 3-chain

In summary, we can link these points together with a relatively concise statement:

In both 2-chain and 3-chain cases, as long as `lockQC(B')` already exists, nodes locked on $B$ can safely switch to $B'$ because the existence of `lockQC(B')` proves there is no `commitQC` for $B$.

The difference lies in the moment **before** `lockQC(B')` is generated:
* In the **2-chain** case, the node genuinely does not know if it should switch to $B'$. It cannot judge if $B$ has met or is about to meet the commit conditions globally, falling into the dilemma of "voting risks double commit, not voting risks liveness."
* **PBFT** explicitly solves this incomplete information problem using the $n-f$ messages in `new-view`. The choice of `highQC` tells the node "which chain the network should follow now."
* **3-chain + 1QC (HotStuff)** embeds the same problem into the protocol structure via view numbers and lock rules: it directly prohibits nodes locked on $B$ from voting for $B'$ while $B'$ is not yet locked; once $B'$ truly locks and has a higher view, that fact itself proves $B$ was not committed, thereby allowing the node locked on $B$ to safely turn to $B'$.