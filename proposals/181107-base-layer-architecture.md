# Base Layer Architecture

## Summary

The Tari base layer is based on the [MimbleWimble] protocol and is merge-mined with Monero.

Major base layer modules:

* Transaction validation and propagation.
* Blockchain management, including block propagation, block validation, seeding and syncing.
* Merge mining
* APIs for clients (wallets, block explorers and peers)

Support modules:
* Cryptography
* A general peer-to-peer messaging service. The interface is abstracted to the point that it 
is reasonably easy to switch out different backends, whether it's IPv4, Tor, I2P etc. The same communication protocol
 should be employed for the second layer.
* Data storage.

## Infrastructure layer

The infrastructure layer doesn't know anything about blockchains, transactions, or digital assets. This layer offers 
a set of modules upon which we build the Tari infrastructure.

### Peer-to-peer messaging

[ØMQ]((http://zguide.zeromq.org)) is a mature distributed communications framework. It's 
very lightweight, is very fast and has excellent documentation. ØMQ forms the basis for peer-to-peer and inter-process
 messaging in Tari.
 
Tari follows a loosely coupled, pub-sub message-based approach to message passing. 

[Message producers] create messages and put them onto the message bus. Any service or module that is interested in 
those messages can subscribe to receive them. With ØMQ, it's relatively straightforward to
 allow publishers and subscribers to be on the same machine in different threads, or across the world.   

Messages in Tari are serialized into the binary [MessagePack](https://msgpack.org/index.html) format
### Data storage

[LMBD](http://www.lmdb.tech/doc/) is a very lightweight key-value store. It is incredibly fast and memory-efficient 
and is used as the persistent storage layer in Monero and Grin, among others.

### Cryptographic services

Tari uses Curve25519 for EC cryptography. The [dalek libraries](https://github.com/dalek-cryptography/curve25519-dalek) 
are a native Rust implementation.  

## Domain layer modules

### Transaction Listener Service

Transactions are received from the [TransactionListener] service, which makes use of  

* [Introduction to MimbleWimble](intro.md)
* Cryptographic Primitives
  * Pedersen Commitments
  * Aggregate (Schnorr) Signatures
    * Bulletproofs
* Block and Transaction Format
  * Transaction
    * Input, output
    * Kernel
  * Block
    * Header
    * Body
    * Compact Block
* Chain State and Merkle Mountain Range
  * Motivation
  * [Merkle Mountain Range](mmr.md)
  * [State and Storage](state.md)
  * [Fast Sync](fast-sync.md)
  * Merkle Proofs
* Proof of Work
  * Cuckoo Cycle
  * Difficulty Algorithm
* Wire protocol
  * Seeding and Sync
  * Propagation
  * Low-level Messages
* Dandelion & Aggregation
* Building Transactions
* Important Parameters
  * Fees and Transaction Weight
  * Reward and Block Weight