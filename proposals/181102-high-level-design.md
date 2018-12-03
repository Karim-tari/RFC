# High level design

## Metadata

This proposal is (tick applicable):

* [x] A new feature
* [ ] an extension to an existing feature
* [ ] an alternative approach to an existing feature or proposal

### Change log

* 2018-12-03: Refactor to use proposal template
* 2018-11-02: Initial submission

### Status

| Date       | Status    |
|:-----------|:----------|
| 2018-11-02 | Submitted |
| 2018-11-03 | Review    |


### Goals

Identifies the key responsibilities of the Tari base layer, including its interaction with the digital assets network.

### Assumptions

1. The Tari base layer operates as a proof-of-work blockchain cryptocurrency, merge mined with Monero.
2. The base layer protocol is based on Mimblewimble.
3. A Tari digital assets layer exists as a second-layer operating on top of the base layer.

### Abstract

This proposal provides a list of functional requirements for the Tari base layer including mining, transaction and block
validation and digital asset interactions.

## Description

The Tari Base Layer has the following jobs:

* Record every [Tari coin](181105-terms.md#tari-coin) transfer in history in an immutable public ledger, or
  [blockchain](./181105-terms.md#blockchain).
* Allow users to prove ownership of Tari coins.
* Reward miners for securing the network by issuing new coins in [Coinbase Transaction](./181105-terms.md#coinbase-transaction)s
* Validate all Tari coin [transactions](./181105-terms.md#transaction).
* Provide a set of higher-order transaction formats to facilitate:
  * Multi-signature transactions
  * Atomic swaps
  * Recording Digital asset state checkpoint summaries
  * Registering of 2nd-layer nodes and their bonded collateral
  * Act as source of truth in second-layer disputes
  * Zero-knowledge contingent payments
 
The Tari Base layer will be based on the MimbleWimble protocol and implemented as a proof-of-work-based blockchain. 
The proof of work will be performed via merge mining with Monero. Arguments for this design are presented [in the 
overview](./181029-overview.md).

### Monero merge mining

Proof of work (PoW) plays two key functions in blockchains:
* To act as a random Oracle
* Make it very expensive to rewrite the transaction history

[Merge mining](https://tari-labs.github.io/tari-university/merged-mining/merged-mining-scene/MergedMiningIntroduction.html)
 allows a blockchain to bootstrap hash rate from an established blockchain.
 
#### Standalone miners
 
Monero-only miners _do not need to know about_ Tari. However, a Tari-enabled Monero merge mining implementation 
requires the following:

* A full Monero mining implementation
  * Monero block selection module
  * Monero mempool module
  * Monero block propagation module
* Modifications to the Monero Coinbase transaction.
* The Tari mining implementation
  * The Tari block selection code
  * Tari block propagation
  * Tari mempool module
  
#### Pool mining

The majority of Monero miners opt to use a mining pool rather than run a standalone mining operation. 

Providing support for mining pool operators is something we should strongly consider. There are 2 main open-source 
mining pool source codes, including

* https://github.com/zone117x/node-cryptonote-pool
* https://github.com/Snipa22/nodejs-pool

Our contributions would comprise 
* modifying Monero block templates, 
* interfacing with the Tari full node.
* communicate with Tari standalone mining module to build Taro block templates
* modify stratum server communications for handling Tari reward shares
* Write the stratum API implementation for clients
* _TODO: What else?_

### Full node software

A Tari full node performs the following critical jobs in maintaining the integrity of the Tari base layer.

* It's responsible for the propagation of Tari transactions to the rest of the network
* It validates every transaction received and silently drops invalid transactions, while propagating valid ones.
* It's responsible for propagating new valid blocks to the rest of the base layer network
* It validates every new block received, silently dropping invalid ones and propagating valid blocks to the rest of 
the network.
* It provides a historical blockchain data to help peers sync up on blockchain history
* Exposes an API to clients (esp SPV clients and wallets) for 
  * Block queries
  * Kernel data queries
  * New transaction requests
  * 2nd layer merkle root checkpoint queries
  * Registered 2nd layer nodes queries
  * Persistent transaction data state queries

_TODO_: Difference between pruned nodes and archival nodes, particularly WRT to syncing


# Disclaimer

This document is subject to the [disclaimer](../DISCLAIMER.md).
    
    
