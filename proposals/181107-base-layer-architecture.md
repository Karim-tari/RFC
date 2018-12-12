# Base Layer Architecture

## Metadata

This proposal is (tick applicable):

* [x] A new feature
* [ ] an extension to an existing feature
* [ ] an alternative approach to an existing feature or proposal

### Change log

* 2018-11-07: Submitted
* 2018-12-02: Refactor to proposal template
* 2018-12-04: Move detailed discussion to separate proposals

### Status

| Date       | Status    |
|:-----------|:----------|
| 2018-11-07 | Submitted |
| 2018-11-08 | Review    |

### Goals

1. Describe the main features of the base layer architecture

### Assumptions

1. The Tari base layer is based on the [MimbleWimble] protocol
2. Proof-of-work is obtained via merge-mining with Monero.

### Abstract

Major base layer modules:

* Transaction validation and propagation.
* Blockchain management, including block propagation, block validation, seeding and syncing.
* Merge mining
* APIs for clients (wallets, block explorers and peers)

Support modules (infrastructure):
* Inter-process messaging services.
* Peer-to-peer messaging service.
* [Data storage](#data-storage).
* [Cryptography](#cryptographic-module)

## Description

### Infrastructure layer

The infrastructure layer doesn't know anything about blockchains, transactions, or digital assets. This layer offers a
set of modules upon which the Tari infrastructure is built.

#### Inter-process communication (IPC)

IPC should be thread-safe, fast, and asynchronous. It should allow different Tari submodules to run in different
processes, or even different machines (if so desired) to maximise security.

[MessageBus](./181204-messagebus.md) is a proposal for a system that tries to achieve these goals.

[ØMQ](http://zguide.zeromq.org)
#### Data storage

The DataStorage module presents an abstracted API for persisting data (presumably to disk). This allows Tari to swap 
out the persistence implementation without affecting modules that depend on it.
 
[LMDB](http://www.lmdb.tech/doc/) is a very lightweight key-value store. It is incredibly fast and memory-efficient 
and is used as the persistent storage layer in Monero and Grin, among others. LMDB's success in these projects make 
it a natural choice as the default persistence solution for Tari.

[LMDB Datastore](./181204-lmdb-datastore.md) is a proposal for implementing the datastore API using LMDB.

Domain modules that will make use of the Datastore are:
* Block storage
* Unspent transaction output storage
* Peer information (full node)
* Peer information (digital asset network)
* Wallet data (keys, transactions and output commitments)
* Digital Asset data (keys, metadata, host information)

#### Cryptographic module

The Tari crypto module provides an abstraction layer for:

* Deterministic key generation
* Message signing
* Signature verification
* Signature aggregation (MuSig)
* Multisig operations
* Commitments (Pedersen, ELGamal etc)
* Threshold cryptography
* Range Proofs
* Bulletproofs
* Symmetric encryption
* Zero-knowledge contingent swaps
* Scriptless script support

The crypto API is presented as a collection of traits so that the backend (Curve25519) can be swapped out without 
impacting any modules making use of the module. 

Tari uses Curve25519 for EC cryptography. The [dalek libraries](https://github.com/dalek-cryptography/curve25519-dalek) 
are a native Rust implementation. 


### Domain layer modules

The Domain layer houses the Tari "business logic". All protocol-related concepts and procedures are defined and 
implemented here. The domain layer makes use of modules on the infrastructure layer to achieve its goals, but the 
infrastructure layer knows nothing about anything implemented here.

This entails that any and all terms defined in the [Glossary] will have a software implementation here, and only here.

#### Base layer nodes

[Base node]s perform the following:

##### Listen for new [Transaction] messages by subscribing to the `MessageBroker` as a `TransactionMessageSubscriber`

* When a new message is received, check whether the `Transaction` is valid by 
  * verifying all signatures in the `Transaction`
  * checking the `Transaction` accounting (specifically that the excess is  valid public key)
  * verifying all the range proofs in the `Transaction`
* If a new `Transaction` is valid,
  * Mark it as verified (is this how we want to handle this???)
  * Push the Verified Transaction onto the internal `MessageBus`

##### Listen for new [Block] messages by subscribing to the `MessageBroker` as a `BlockMessageSubscriber`
  
* When a new block is received, check whether the block is valid by
  * Verifying the `BlockHeader`
  * Verifying the `ProofOfWork`
  * Verifying the block weight (proxy for block size)
  * Verifying every `Transaction` in the block
  * Verifying that the block is being added to the chain tip 
* If the block is not being added to the chain tip, but is otherwise valid, check for chain reorgs
* If the block is valid, 
  * add it to the chain tip
  * transmit the block to its peers

##### Listen for new peer connection requests

When a new connection request is received, the node will check whether the peer has been blacklisted.
If not, it will allow the connection and add the peer to the `Peers` list.

##### Tell peers about new transactions

##### Tell peers about new blocks

##### Seeding and synchronizing

When base nodes start up, they need to synchronize the blockchain with their peers. Nodes need to have the 
following functionality to achieve this:
* Connect to peer
* Request peer list from peer
* Request most recent chain state (total accumulated work, block height etc)
* Request blocks from peer
* Request `Transaction`s from peer
* Send requested blocks to peer


#### Mining nodes

Mining nodes perform the following:

##### Compile new [Block] by subscribing to the `MessageBroker` as a `VerifiedTransactionSubscriber`

* Construct `BlockHeader` 

* Construct Coinbase transaction

* Add verified transactions to the block using a `TransactionSelectionPolicy` -- allowing miners to swap out their transaction selection strategy if they wish.

* Perform cut-through between transactions where transactions would be cancelled out

* Push the newly constructed block [Block] onto the internal `MessageBus`

#### What else?

ToDo

#### The Mempool

The mempool is a database of all unconfirmed, VALID, transactions. The mempool is used in the following situations:
* For mining nodes to select transactions to build new blocks
* To use as a cross-checking reference for checking whether a new transaction is valid

The mempool adds transactions to the pool by listening for `NewVerifiedTransaction` messages on the internal 
`MessageBus`.

The mempool also listens for `NewVerifiedBlock` messages.

The mempool responds to requests for transaction lists according to a `BlockSelectionStrategy`, usually, something 
that maximises fees for a certain block size, but mining nodes are free to select the strategy they desire.

The mempool must respond to requests for whether a given UTXO is being used as an input in the mempool.

The mempool removes transactions from the mempool when:
* A `NewVerifiedBlock` messages arrives. All transactions listen in the block are removed from the mempool.
* Memory constraints are reached, specified in a `MempoolPolicy`. The oldest and/or transactions with lowest fees will
 be dropped according to the `MemPoolPolicy` in action.

#### Peer Broadcasting

A `PeerBroadcaster` listens for specific messages on the internal `MessageBus` and then relays them to peers using a 
`PeerBroadcastStrategy` (e.g. gossip or flood). Separate `PeerBroadcaster` instances will propagate transactions 
and blocks. 

_TODO_: Should syncing and seeding happen on dedicated ports, since they're not really as event driven? Or utilize the 
same infrastructure? 
 
 
#### Transactions

Transactions are what allows value transfer to occur in the Tari network. A full `Transaction` consists of

* A `TransactionType` (Coinbase, Regular, Checkpoint, Collateral)
* An array of zero or more `TransactionInput`s
* An array of zero or more `TransactionOutput`s. Transaction outputs comprise
  * A `Commitment` which blinds the value of the output (MW protocol suggests Pedersen commitments -- is it 
  worthwhile abstracting this to allow other commitment types?)
  * A `RangeProof` proving that the commitment value > 0
  * A bitmask of `OutputFeatureFlags` 
* A `TransactionKernel`, which contains: 
  * Signatures for every output `BlindingFactor`
  * A bitmask of `TransactionFeatures` for the `Transaction`
  * The transaction fee in nanoTari
  * The maturation schedule (determining when outputs can be spent)
  * The `TransactionExcess`, which must be a valid public key for the accounting to hold
  * A signature using the excess private key, proving knowledge that the excess value is known 
  
_TODO_: Grin includes these fields too. Figure out why they are necessary
  * The max lock_height of all *inputs* to this transaction

_Random Thought_: Is it better to have a single `Transaction` struct with a `is_valid` field (true, false. None), or  
different types: `ValidatedTransaction`, `UnvalidatedTransaction` & `InvalidTransaction`? The latter form can leverage 
Rust's static typing to prevent, _at compile time_ any bugs that would let an invalid transaction into a block, for example. It's 
pretty cool, but I wonder if it's worth the effort / results in too much repetition of code?

#### Blocks

In a decentralised proof-of-work ledgers, transactions cannot be reliably and unambiguously ordered by timestamp. 
Instead, Nakamoto consensus defines transaction ordering by block sequence. Miners collect transactions into blocks 
and publish them to the network to be included into the blockchain. Tari's base layer is based on the MimbleWimble
 protocol; thus Tari blocks contain the following information:
 
* A `BlockHeader` which contains various metadata about the `Transaction`s in the block, as well as a link to the 
previous block in the chain.
* The `BlockBody`, which contains all the `Transactions` in the block.

The `BlockHeader` contains the following:
* The block version number
* The block height
* The `BlockHash` of the previous block
* A `BlockTimestamp` for this block
* A `MerkleRoot` for the transaction commitments in the `BlockBody`
* A `MerkleRoot` for the range proofs in the `BlockBody`
* A `MerkleRoot` of all transaction kernels in the `BlockBody`
* The `ProofOfWork` for the block, including
    * The `MoneroBlockHeader` for the block - because we're merge mining

_TODO_: Grin includes these fields too. Figure out why they are necessary:	
  * Total accumulated sum of kernel offsets since genesis block.
  *	Total accumulated sum of kernel commitments since genesis block. Should always equal the UTXO commitment sum 
  minus supply.
  * Total size of the output MMR after applying this block
  * Total size of the kernel MMR after applying this block

_Note_: The Grin implementation of MW splits transaction excesses into two values to prevent reconstructing of 
individual transactions of blocks by analyzing the inputs and outputs. We should implement this feature to improve 
privacy. 

#### TransactionMessageSubscriber

The `TransactionMessageSubscriber` is a simple service that connects to the `MessageBroker` and filters all Transaction 
messages originating from the network. It deserialises the message into a `Transaction` object before handing it over
 to the connected client (typically a `BaseNode` instance). 
 
#### BlockMessageSubscriber

Similarly, the `BlockMessageSubscriber` is a simple service that connects to the `MessageBroker` and filters 
all Block messages originating from the network. It deserialises the message into a `Block` object before handing it 
over to the connected client.

  
# Disclaimer

This document is subject to the [disclaimer](../DISCLAIMER.md).

[Glossary]: /Glossary.md "Glossary"
[MimbleWimble]: /Glossary.md#mimblewimble
[Base node]: /Glossary.md#base-node
[transaction]: /Glossary.md#transaction
[block]: /Glossary.md#block
