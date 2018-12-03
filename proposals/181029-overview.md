# Architectural overview

## Metadata

This proposal is (tick applicable):

* [x] A new feature
* [ ] an extension to an existing feature
* [ ] an alternative approach to an existing feature or proposal

### Change log

* 2018-12-02: Reformat for template
* 2018-10-29: Initial submission

### Status

| Date       | Status    |
|:-----------|:----------|
| 2018-10-29 | Submitted |
| 2018-10-30 | Review    |

### Goals

The aim of this proposal is to
1. Provide a very high-level perspective for the moving parts involved in the Tari protocol.
2. Present some trade-offs involved in the major technology decisions for the Tari protocol and posit some tentative solutions.

### Assumptions

1. We wish to optimise the Tari digital assets experience for speed, and the Tari coin experience for security.

### Abstract

A two-layer approach for Tari is proposed.

The base layer deals with Tari coin transactions. It governed by a proof-of-work blockchain that is merged-mined with Monero.
The base layer is highly secure, decentralised and relatively slow.

The digital assets network runs on a second layer. It is built for liveness, speed and scalability at the expense of decentralisation.


## Description

### Some objectives to keep in mind

The Tari protocol is all about *native digital assets*. So the architecture must support the particular requirements of
a digital assets platform.

### The Tari base coin
The Tari base coin is a cryptocurrency that acts as the "lubricant" for the network's machinery. It is quite possible that 
the user requirements/expectations for the Tari coin and the assets themselves are different, precipitating different
strategies for accommodating those needs.

#### Base coin requirements

Specifically, since the Tari coin plays the role of money in many ways, the following are MUST HAVE features of the base coin:

* Security
* Decentralised
* Censorship resistance
* Privacy

In addition, the base coin has the following NICE TO HAVE features, but not at the expense of the four primary
features:

* Speed
* Scalability

#### Digital assets

The Tari network will be used to create and manage digital assets. These digital assets will define and contain
asset tokens. For example, in a ticketing context, an single event asset will contain many ticket tokens. The 
ticket tokens will have state, such as its current owner and whether it has been redeemed or not. 

The Tari network must manage and execute _asset instructions_, including asset creation, token transfers and state changes. 

The critical MUST HAVE requirements for digital assets and their tokens are:

* Security
* Speed (perhaps in the order of 1,000 TPS)
* Scalability (think millions of in-game items + metadata)

The following features would be highly desirable and would add to the Tari protocol's value proposition:

* Opt-in privacy (depending on the needs of the asset issuer)
* Configurable decentralisation (depending on the needs of the asset issuer). For example, a given issuer might want 
it such that only their node(s) be allowed to execute asset instructions (a permissioned system), whereas others might
prefer completely decentralised management of their assets (permissionless).

#### Two layers

These broad requirements are in some ways mutually exclusive. Consider the distributed system trilemma of wanting a 
network to be

* Fast
* Cheap
* Secure

but only being able to pick at most two [[1]](https://en.wikipedia.org/wiki/CAP_theorem). 

This suggests partitioning the Tari network into two layers:

* A base layer that provides a public ledger of base coin transactions, secured by proof of work (to maximise Security), and
* A second layer that manages digital asset state that is very fast and cheap, but which can fall back on the base layer's security when required.
  
### The Base Layer

The only system that we _know_ currently works as a base layer capable of delivering at least the first three must-have 
properties above (security, decentralisation, censorship resistance) is a coin based on a proof-of-work blockchain.

Proof of X, where X is stake, spacetime, waffle, etc. is either totally unproven at best, or a house of cards at worst.

Looking at the _nice-to-have_ properties, one can go a long way towards achieving a scalable blockchain by

* Keeping as much data off the blockchain to begin with
* Employing novel "compression" methods such as cut-through
* Using a space-efficient transaction protocol

In terms of privacy, there are several options to take:

* Ring signatures & CT, ala Monero
* ZK-Snarks, ala ZCash
* Confidential Transactions, ala MimbleWimble

Weighing all these considerations up, it seems reasonable that MimbleWimble offers the best "bang for buck" in terms
of the desired features of the base layer. To quote @fluffypony, "MW is the most sound, scalable 'base layer' protocol we know"

### Proof of work

There are a few options for the proof of work mechanism for Tari:

* Implement an existing PoW mechanism. This is a bad idea, because a nascent cryptocurrency that uses a non-unique
  mining algorithm is incredibly vulnerable to a 51% attack from miners from other currencies using the same algorithm.
  Bitcoin Gold, Verge have already experienced this, and it's a [matter of time](https://www.crypto51.app/) before 
  it happens to Bitcoin Cash. 
* Implement a unique PoW algorithm. This is a risky approach and comes close to breaking the #1 rule of 
  cryptocurrency design: Never roll your own crypto.
* Implement a hybrid PoW algorithm. This strategy has it's own set of problems _[citation needed]_.
* [Merge mining](https://tari-labs.github.io/tari-university/merged-mining/merged-mining-scene/MergedMiningIntroduction.html).
  This approach is not without its own risks but offers the best trade-offs in terms of bootstrapping the
  network and could offer high levels of hash rate from day one if the parent blockchain miners opt in and high levels
  of 51% attack resistance if the resulting hash rate is sufficiently distributed among mining pools.

Given Tari's relationship with Monero, a merge-mined strategy with Monero makes the most sense.
  
### The second layer

They key properties of the digital assets network are speed and scalability, without compromising on security. Here, we
can definitely sacrifice decentralisation, for example. In fact, in many ways it's desirable, since the vast majority
of assets (and their issuers) don't need or want _the entire network_ to validate every state change in their asset
contracts.

Once this trade-off is accepted, the goals of speed and scalability that we set out above look far more achievable.

Digital assets necessarily have _state_. Therefore the digital assets layer must have a means of synchronising and agreeing
on state where asset state management is handled by multiple servers, (a.k.a. reaching consensus).

There are multiple options we can consider for the second-layer architecture, but most of them require some sort of 
overlay network coupled to a consensus algorithm.

Overlay network options:
* Each node maintains a list of peer nodes, but hold very little other information about them (e.g. Bitcoin); Node
capabilities are obtained by querying peers.
* Each node tracks pockets of the network, and perhaps some metadata (e.g. DHT's like Kademlia)
* Each node tries to maintain a full list of peers, along with information about which digital assets (DAs) each node is authorised to
 process instructions for. Queries about a specific DA can be routed directly to the node(s) that are tracking it. 
 (Lightning builds a full topology of the channel network)

Consensus options:
* A full second-layer blockchain. This is probably overkill and unlikely to achieve the speed and cost targets we would 
like, particularly if it's a proof-of-work blockchain.
* State Channel network. Ideas like Plasma etc., are interesting, but are incredibly complex. Complexity increases the
  system's attack surface, and makes it more prone to failure and bugs. One feels that there are simpler and more elegant solutions
  to this problem.
* "Permissioned" DAs. In this model, DA issuers must nominate (trusted) nodes to run their DAs for them. Employing
multiple nodes provides DoS attack resistance rather than Byzantine fault tolerance. Bonded
contracts between issuer and nodes on the base layer can provide additional incentives to maintain uptime and honesty
 (though the incentive structure must be carefully thought out to prevent co-ordinated attacks by rogue asset issuers
  colluding with malicious nodes)
* Directed Acyclic Graphs. DAG implementations, like Spectre, PARSEC, HoneyBadgerBFT etc. are an interesting 
possibility. DAGs have the potential to provide reasonable speed and scalability whilst still offering true BFT 
tolerance and thus promising a truly permissionless second layer. 

_Other tools in our belt / possible features_:
* 2nd layer nodes use public/private keys to identify themselves.
* 2nd layer nodes post a bonded contract that they stand to lose if they are shown to be bad actors.
* 2nd layer nodes can be compensated for this risk and the cost of operating a node by earning (tiny) fees for every
digital asset instruction (creation, modification) they execute.
* Asset issuers can choose to nominate the nodes that get to execute their DA instructions (authorised nodes). This 
corresponds to a permissioned system. These nodes would be trusted; the issuers would run these nodes themselves 
or know who does.
* Asset issuers could also issue assets in "permissionless" mode, presumably with a set of criteria for eligibility 
(e.g. minimum size of bonded contract). A suitable consensus algorithm will need to be in place to allow this mode to
 operate successfully.
* Asset issuers could also post a bonded contract to the 2nd layer nodes with value equal to the estimated life-time management costs of the DA
* We have a (non)central authority at our disposal: the base layer! The base layer could potentially be used as a 
registrar for 2nd layer nodes, notary for bonded contracts and contract state and arbiter in case of disputes.

### Summary

Table 1 summarises the defining characteristics of the Tari network layers:

|                                      | Base Layer | Second layer           |
|--------------------------------------|------------|------------------------|
| Speed                                | Slow       | Fast                   |
| Scalability                          | Moderate   | Very high              |
| Security                             | High       | Mod (High w/ fallback) |
| Decentralisation                     | High       | Low - Med              |
| Processes Tari coin tx               | Yes        | No                     |
| Processes digital asset instructions | Only checkpoints | Yes              |

## Topics for further discussion

* Configurable privacy in digital Assets. What are some use cases for private DAs in a public network?
* Which configuration of network overlay & consensus algorithm will be the simplest, and still work?
* Start to think about how the 2nd layer and base layer interact.
* Long-lived vs short-lived digital assets? How does this influence the incentive / funding model?


# Disclaimer

This document is subject to the [disclaimer](../DISCLAIMER.md).