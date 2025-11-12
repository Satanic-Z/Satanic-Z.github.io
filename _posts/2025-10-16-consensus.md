---
layout: post
title: "A Roadmap for Understanding Blockchain Consensus: Key Assumptions and Modules"
date: 2025-10-16 00:00 +0800
---


In this post, I'll outline some of the core assumptions and essential modules in the field of **consensus**, as it’s a great place to start.

### A. Core Assumptions

#### A1: Network Assumptions

1.  **Partial Synchrony:** This is the path from PBFT → three-chain HotStuff (as described in the original paper) → two-chain HotStuff (seen in protocols like Jolteon and Aptos). The primary evolution here is solving the **commit interruption problem**.
    * **Explanation:** Pipelined HotStuff requires a block to receive Quorum Certificates (QCs) from three consecutive views to be committed; otherwise, security is compromised. However, if a QC is not produced in one of the intermediate views, the proposed block is wasted. A significant body of work has focused on solving this specific issue.

2.  **Asynchrony:** This category includes protocols based on Asynchronous Common Subset (ACS), Multi-Valued Byzantine Agreement (MVBA), Binary Byzantine Agreement (BBA), and various DAG-based consensus mechanisms. To get started, you can look up some of the foundational papers. For example, for ACS, read the HoneyBadger paper. For MVBA, check out VABA and Dumbo-MVBA.
    * Asynchronous protocols universally rely on a **random coin** to circumvent the FLP impossibility theorem. Understanding the importance of this common coin and how to use it is crucial for designing and analyzing asynchronous consensus. Furthermore, understanding the proof method for FLP impossibility will help you grasp subsequent proofs in the literature.

3.  **Synchrony:** Synchronous consensus is not my area of expertise. I recommend checking out the work of Ling Ren and the excellent content on the Decentralized Thoughts website.

#### A2: Adversary Assumptions

The adversary model can be divided into static, mildly adaptive, and strongly adaptive corruption. For a deeper dive, I recommend the paper *"Adaptively Secure Broadcast, Revisited"*, as I haven't seen an earlier discussion on this topic.

1.  **Static Corruption:** As the name implies, the adversary chooses which nodes to corrupt at the beginning of the protocol and never changes them.
2.  **Mildly Adaptive Corruption:** The adversary can change which nodes are corrupted during execution, but they *cannot* corrupt an honest node *after* it has sent a message.
3.  **Strongly Adaptive Corruption:** The adversary has the power to corrupt a node *after* it has already sent a message.

I'll admit my understanding of this area is still developing, and I don't have a crystal-clear intuition yet. 

#### A3: Cryptography Assumptions

Why do we use signatures? What is the role of a hash function? For these topics, I highly recommend the work of Sisi Duan and Xuechao Wang, two incredibly gifted researchers who are currently pushing the frontiers of uncertified consensus protocols.

### B. Important Modules & Concepts

#### B1: Mempool

The mempool decouples block data dissemination from the critical path of the consensus protocol. This greatly enhances the stability of consensus and has become a standard component in modern experimental setups. The core idea is for consensus messages to contain only the *digests* (hashes) of blocks. Nodes then fetch the full block data after receiving the proposal.

A key problem the mempool must solve is the **data availability problem**: a malicious node could include a block's digest in a proposal without ever broadcasting the actual block data. The current standard solution is Narwhal, but its "pull-based" request model feels, in my personal opinion, less elegant than the approach in DispersedLedger, where the concept and role of the mempool are also more clearly defined.

#### B2: Sleepy Model Consensus

This is not an area I have researched deeply, but I recall the general progression. The "Sleepy Model" was proposed by Elaine Shi et al. and has since evolved into its own subfield. You can refer to Ling Ren's work for more details.

*(Correction: It has been pointed out that modern consensus assumptions can be broadly categorized into three types: the Sleepy model, the Permissionless model, and the classic BFT model.)*
- The sleepy model relies on the assumption that the node number $n$ can vary over time, with the only requirement being that at any given moment, the number of honest nodes exceeds the number of malicious ones.
- The permissionless model assumes that nodes can join and leave the network at will, and does not assume a fixed set of participants or an explicit number of $n$.
- The classic BFT model assumes a fixed set of $n$ nodes, with up to $t$ of them being malicious.

#### B3: Complexity

Understanding consensus complexity requires first knowing two key concepts: **message complexity** and **bit/communication complexity**.

* **Message Complexity:** Refers to the total number of messages that must be sent across the network to commit a single decision. It is the most fundamental theoretical metric for evaluating a consensus protocol's complexity.
* **Bit Complexity:** Refers to the total number of bits that must be sent across the network. This metric is often used to gauge the real-world performance of a protocol. Lowering bit complexity usually involves micro-optimizations like amortization, but its lower bound is fundamentally constrained by the message complexity.

Focusing on message complexity, the **Dolev-Reischuk lower bound** proved that any deterministic consensus protocol requires quadratic ($O(n^2)$) message complexity, even under a static adversary. This bound has been widely confirmed. For example, the normal case for PBFT (without cryptographic signatures) is $O(n^2)$, while its view-change is $O(n^4)$. HotStuff, by broadcasting only a single QC during a view-change (assuming cryptography is used), achieves $O(n^2)$ message complexity even with *t* consecutive failures, thus matching the theoretical lower bound.

Many public chain consensus protocols claim to have sub-quadratic, or even sub-linear, complexity. However, these are often non-deterministic and therefore lack true, finality.

Asynchronous consensus protocols typically have the highest complexity. Intuitively, to ensure liveness, every node must be prepared for a certain level of broadcasting, leading to at least quadratic complexity. Combined with other factors, their message complexity is generally not ideal.

The ability to analyze a protocol's complexity is a crucial skill. It allows you to quickly assess whether your own ideas have theoretical merit. If your idea has the same complexity as the state-of-the-art (SOTA) without offering any improvements in its assumptions (e.g., they use signatures and you don't) and provides no real-world performance benefit, then it's probably best to discard the idea and move on.

---
*That's all I can think of for now. My knowledge is limited, so this guide is by no means complete. I'll add more as I learn.*