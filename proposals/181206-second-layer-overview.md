# Digital Assets Network overview

## Metadata

This proposal is (tick applicable):

* [x] A new feature
* [ ] an extension to an existing feature (with link)
* [ ] an alternative approach to an existing feature or proposal (link)

### Change log

### Status

| Date       | Status    |
|:-----------|:----------|
| 2018-11-01 | Submitted |

### Goals

The goal of this proposal is to describe the key features of the Tari second layer, a.k.a the Digital Assets Network (DAN)

### Assumptions

* The proposals from [the high level overview](./181029-overview.md) are accepted; specifically that
* The [digital asset network] should be fast, relatively cheap, and highly scalable.

### Abstract


* [Digital asset]s are managed by special nodes called [Validator node]s (VNs). VNs manage digital asset state and ensure
  that the rules of the asset contracts are enforced.
* VNs form a peer-to-peer communication network that together defines the Tari [Digital Asset Network] (DAN)
* VNs register themselves on the base layer and commit collateral to prevent Sybil attacks.
* Scalability is achieved by sacrificing decentralisation. Not *all* VNs manage *every* asset. Assets are managed by
  subsets of the DAN, called VN committees. These committees reach consensus on DA state amongst themselves.
* VNs earn fees for their efforts.
* Digital asset contracts are not Turing complete, but are instantiated using Digital Asset templates that are defined
  in the DAN protocol code.

## Description

Nodes on the 2nd layer are called [Validator Node]s. They validate instructions concerning Tari [digital asset]s.

[Validator node]s form a peer-to-peer network, called the Tari [Digital Asset Network] (DAN, or TDAN).

## Responsibilities of Validator Nodes

### Registration

VNs register themselves on the [Base Layer] using a special [transaction] type. The registration [transaction] type
requires spending of a certain minimum amount of [Tari coin], the (`RegistrationCollateral`), that has a time-lock on the
output for a minimum amount of time (`RegistrationPeriod`) as well as some metadata, such as the VNs public key. The
details of this transaction are not confidential and are publicly auditable.

VNs may spend this Tari back to themselves.

Requiring nodes to register themselves serves two purposes:
* Makes VN Sybil attacks expensive
* Provides an authoritative "central-but-not-centralised" registry of validator nodes from the base layer.

### Manage digital assets

* VNs are expected to manage the state of digital assets on behalf of digital asset issuers. They receive fees as reward
for doing this.
* Digital assets consist of an initial state plus a set of state transition rules. These rules are set by the Tari
  protocol, but will usually provide parameters that must be specified by the asset issuer.
* It is the VNs responsibility to ensure that every state change in a digital asset conforms to the contract's rules.
* VNs accept digital asset [Instructions] from clients and peers. [Instructions] allow for creating, updating, and
  expiring digital assets on the DAN.
* VNs provide additional collateral when accepting an offer to manage an asset, which is stored in a multi-signature
  contract on the base layer. This collateral can be taken from the VN if it is proven that the VN engaged in
  malicious behaviour.
* VNs participate in fraud proof validations in the event of consensus disputes (which could result in the malicious VN's
  collateral being slashed).
* VNs periodically post checkpoint summaries to the [base layer] for each asset that they are managing.
* Digital asset metadata (e.g. large images) are managed by VNs, but whether the data is considered part of the state
  (and thus checkpointed) or out of state depends on the type of digital asset contract employed.

### Peer-to-Peer Communication

* VNs must maintain a list of peers, and which assets each peer is managing.
* VNs must relay [instructions] to interested peers.
* VNs must respond to requests for information about digital assets that they manage on the DAN.
* VNs and clients can advertise public keys to facilitate P2P communication encryption


### Digital Assets

* Digital asset contracts are *not* Turing complete, but are selected from a set of `DigitalAssetTemplate`s that govern
  the behaviour of each contract type. e.g. there could be a Single-Use Token template for simple ticketing systems; a
  Coupon template for loyalty programmes and so on.
* The template system is intended to be highly flexible and additional templates can be added to the protocol periodically.
* Asset issuers can register TLDIs (top-level digital issuer) names on the base chain to help disambiguate similar
  contracts and improve the signal-to-noise ratio from scam- or copy-cat contracts.

[base layer]: ../Glossary.md#base-layer
[digital asset]: ../Glossary.md#digital-asset
[validator node]: ../Glossary.md#validator-node
[digital asset network]: ../Glossary.md#digital-asset-network
[transaction]: ../Glossary.md#transaction
[tari coin]: ../Glossary.md#tari-coin
[instructions]: ../Glossary.md#instructions