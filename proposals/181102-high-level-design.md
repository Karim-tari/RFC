# High level design

## Base layer responsibilities

The Tari Base Layer has the following jobs:

* Record every [Tari coin]() transfer in history in an immutable public ledger, or [blockchain](./181105-terms
.md#blockchain).
* Allow users to prove ownership of Tari coins.
* Reward miners for securing the network by issuing new coins in [Coinbase Transaction](./181105-terms
.md#coinbase-transaction)s
* Validate all Tari coin [transactions](181105-terms.md#transactions).
* Provide a set of higher-order transaction formats to facilitate:
  * Multi-signature transactions
  * Atomic swaps
  * Recording Digital asset state checkpoint summaries
  * Registering of 2nd-layer nodes and their bonded collateral
  * Act as arbitrator in second-layer disputes
  * Zero-knowledge contingent payments
 
The Tari Base layer will be based on the MimbleWimble protocol and implemented as a proof-of-work-based blockchain. 
The proof of work will be performed via merge mining with Monero. Arguments for this design are presented [in the 
overview](./181029-overview.md).

## Monero merge mining

Proof of work (PoW) plays three key functions in blockchains:
* To act as a random Oracle
* Make it very expensive to rewrite the transaction history

[Merge mining](https://tari-labs.github.io/tari-university/merged-mining/merged-mining-scene/MergedMiningIntroduction.html)
 allows a blockchain to bootstrap hash rate from an established blockchain.
 
### Standalone miners
 
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
  
### Pool mining

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

## Full node software

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

_TODO_: Difference between pruned nodes and archival nodes, particularly WRT to syncing


# Disclaimer

This document is subject to the [disclaimer](../DISCLAIMER.md).
    
    