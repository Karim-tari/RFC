# Base Layer Architecture

## Summary

The Tari base layer is based on the [MimbleWimble] protocol and is merge-mined with Monero.

Major base layer modules:

* Transaction validation and propagation.
* Blockchain management, including block propagation, block validation, seeding and syncing.
* Merge mining
* APIs for clients (wallets, block explorers and peers)

Support modules (infrastructure):
* A general peer-to-peer messaging service ([MessageBus](#messagebus)).
* [Data storage](#data-storage).
* [Cryptography](#cryptographic-module)

## Infrastructure layer

The infrastructure layer doesn't know anything about blockchains, transactions, or digital assets. This layer offers 
a set of modules upon which we build the Tari infrastructure.

### MessageBus

[ØMQ]((http://zguide.zeromq.org)) is a mature distributed communications framework. It's 
very lightweight, is very fast and has excellent documentation. ØMQ forms the basis for peer-to-peer and inter-process
 messaging in Tari.
 
Tari follows a loosely coupled, pub-sub message-based approach to message passing. 

Messages in Tari are serialized into the binary [MessagePack](https://msgpack.org/index.html) format before being put
 onto the wire. 
 
Messages come from various sources - peers, other processes on the same machine, or clients accessing the APIs. The 
data in the messages is invisible to MessageBus. All it sees is a string of bytes (that can be deserialized by serde 
and the MessagePack library).

The MessageBus architecture follows a fan-in, fan-out pattern. An arbitrary number of MessagePublishers connect to a 
single MessageBroker object that passes messages on to an arbitrary number of MessageSubscribers. MessagePublishers 
are implementation-specific classes that, e.g. collect messages from peer nodes over I2P, or HTTP, or from a mining 
pool over Stratum. The MessagePublishers MAY format messages if desired, but MUST deliver messages to the 
MessageBroker in MessagePack format.

Messages SHOULD have the MessageType field as the first field in its serialization.

The MessageBroker MUST NOT modify the message.

MessageSubscribers connects to the MessageBroker and MUST specify a filter for messages it is interested in. This is 
achieved by calling the `setFilter` method every MessageBroker MUST implement, which sets the message filter. The 
filter is based on the MessageType field which SHOULD be present as the first field in every message.

MessageSubscribers MUST subscribe to either one message type, or all messages.

The MessageBroker MUST send every message to every MessageSubscriber that has a filter on for that type of message. 

The basic structure looks as follows:

```text
         Stratum                  Peer Nodes                REST API
            +                         +                        +
            |                         |                        |
            |                         |                        |
  +---------+---------+     +---------+---------+     +--------+----------+
  |  MessagePublisher |     |  MessagePublisher |     |  MessagePublisher |
  +-------------------+     +-------------------+     +-------------------+
  |       PUSH        |     |        PUSH       |     |       PUSH        |
  +---------+---------+     +---------+---------+     +---------+---------+
            |                         |                         |
            |                         |                         |
            |                         |                         |
            |                         |                         |
            |                         |                         |
            |                         |                         |
            |               +---------v--------+                |
            +--------------->       PULL       <----------------+
                            +------------------+
                            |   MessageBroker  |
                            +------------------+
                            |       PUB        |
           +----------------+--------+---------+---------------+
           |                         |                         |
           |                         |                         |
           |                         |                         |
           |                         |                         |
+----------v--------+      +---------v---------+     +---------v---------+
|        SUB        |      |        SUB        |     |        SUB        |
+-------------------+      +-------------------+     +-------------------+
| MessageSubscriber |      | MessageSubscriber |     | MessageSubscriber |
+-------------------+      +-------------------+     +-------------------+
```

The MessageBus is constructed in a very general way, so that the same infrastructure can be used for both base and 
second layer messaging.

The MessageBus is locked into using ØMQ, but Messages implement the `Serializable` trait from serde, so that the binary
 protocol can be swapped out if desired.

The MessageBus can be used for both internal (inter-thread or in-process) messaging using `ipc` or `inproc` 
transports, as well as external messaging using `tcp`. 

The MessageBus can also be augmented with 0MQ pipelines (using PUSH, PULL, ROUTER and DEALER sockets), which are 
analogous to node.js stream pipes.

### Data storage

The DataStorage module presents an abstracted API for persisting data (presumably to disk). This allows Tari to swap 
out the persistence implementation without affecting modules that depend on it.
 
[LMDB](http://www.lmdb.tech/doc/) is a very lightweight key-value store. It is incredibly fast and memory-efficient 
and is used as the persistent storage layer in Monero and Grin, among others. LMDB's success in these projects make 
it a natural choice as the default persistence solution for Tari.

_TODO_:
* Define the DataStorage API

### Cryptographic module

The Tari crypto module provides an abstraction layer for:

* Deterministic key generation
* Message signing
* Signature verification
* Signature aggregation (MuSig)
* Multisig operations
* Commitments (Pederson)
* Threshold cryptography
* Range Proofs
* Bulletproofs
* Symmetric encryption

The crypto API is presented as a collection of traits so that the backend (Curve25519) can be swapped out without 
impacting any modules making use of the module. 

Tari uses Curve25519 for EC cryptography. The [dalek libraries](https://github.com/dalek-cryptography/curve25519-dalek) 
are a native Rust implementation. 


## Domain layer modules

The Domain layer houses the Tari "business logic". All protocol-related concepts and procedures are defined and 
implemented here. The domain layer makes use of modules on the infrastructure layer to achieve its goals, but the 
infrastructure layer knows nothing about anything implemented here.

This entails that any and all terms defined in the [Glossary] will have a software implementation here, and only here.

### Base layer nodes

[Base node]s perform the following:

#### Listen for new [Transaction] messages by subscribing to the `MessageBroker` as a `TransactionMessageSubscriber`

* When a new message is received, check whether the `Transaction` is valid by 
  * verifying all signatures in the `Transaction`
  * checking the `Transaction` accounting (specifically that the excess is  valid public key)
  * verifying all the range proofs in the `Transaction`
* If a new `Transaction` is valid,
  * Mark it as verified (is this how we want to handle this???)
  * Push the Verified Transaction onto the internal `MessageBus`

#### Listen for new [Block] messages by subscribing to the `MessageBroker` as a `BlockMessageSubscriber`
  
* When a new block is received, check whether the block is valid by
  * Verifying the `BlockHeader`
  * Verifying the `ProofOfWork`
  * Verifying every `Transaction` in the block
  * Verifying that the block is being added to the chain tip 
* If the block is not being added to the chain tip, but is otherwise valid, check for chain reorgs
* If the block is valid, 
  * add it to the chain tip
  * transmit the block to its peers

#### Listen for new peer connection requests

When a new connection request is received, the node will check whether the peer has been blacklisted.
If not, it will allow the connection and add the peer to the `Peers` list.

#### Tell peers about new transactions

#### Tell peers about new blocks

#### Seeding and synchronizing

When base nodes start up, they need to synchronize the blockchain with their peers. Nodes need to have the 
following functionality to achieve this:
* Connect to peer
* Request peer list from peer
* Request most recent chain state (total accumulated work, block height etc)
* Request blocks from peer
* Request `Transaction`s from peer
* Send requested blocks to peer

### The Mempool 

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
 
 

### Peer Broadcasting

A `PeerBroadcaster` listens for specific messages on the internal `MessageBus` and then relays them to peers using a 
`PeerBroadcastStrategy` (e.g. gossip or flood). Separate `PeerBroadcaster` instances will propagate transactions 
and blocks. 

_TODO_: Should syncing and seeding happen on dedicated ports, since they're not really as event driven? Or utilize the 
same infrastructure? 
 
 
### Transactions

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

### Blocks

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

### TransactionMessageSubscriber

The `TransactionMessageSubscriber` is a simple service that connects to the `MessageBroker` and filters all Transaction 
messages originating from the network. It deserialises the message into a `Transaction` object before handing it over
 to the connected client (typically a `BaseNode` instance). 
 
### BlockMessageSubscriber

Similarly, the `BlockMessageSubscriber` is a simple service that connects to the `MessageBroker` and filters 
all Block messages originating from the network. It deserialises the message into a `Block` object before handing it 
over to the connected client.

  
# Disclaimer

This document is subject to the [disclaimer](../DISCLAIMER.md).

[Glossary]: ./181105-terms.md "Glossary"
[MimbleWimble]: ./181105-terms.md#mimblewimble
[Base node]: ./181105-terms.md#base-node
[transaction]: ./181105-terms.md#transaction
[block]: ./181105-terms.md#block