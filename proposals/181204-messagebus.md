# The Message Bus service

## Metadata

This proposal is (tick applicable):

* [ ] A new feature
* [x] an extension to [base layer architecture](./181107-base-layer-architecture.md)
* [ ] an alternative approach to an existing feature or proposal

### Change log

### Status

| Date       | Status    |
|:-----------|:----------|
| 2018-12-04 | Submitted |


### Goals

The aim of this proposal is to provide a safe, scalable and performant multi-tasking framework and asynchronous
messaging service between Tari modules.

### Assumptions

The Tari base layer is split into independent modules, as described in [base layer architecture](./181107-base-layer-architecture.md).

### Abstract

The [ØMQ] library is leveraged to provide thread-safe asynchronous communications between Tari modules.

This proposal describes a fan-in, fan-out message broker service. This provides a scalable, asynchronous approach to
handling the specific use case of multiple sources producing the same _type_ (or types) of message, and multiple sinks
wanting to read all of those messages (or filter on specific types).

Messages of various types are produced and pushed to a message broker that delivers the messages to ALL subscribers that
are interested in messages of that type.

Messages are serialised using the MessagePack protocol, an efficient and compact binary format.

This approach has some key benefits:
* Modules can communicate via TCP, Unix sockets, or direct memory sharing (in-process only). The choice is up to the node
  operator and is selected via configuration file.
* This means that Tari modules can run on the same machine, or on different machines.
* Although it is possible, it's recommended that modules *not* share memory directly (via `inproc` sockets) for security
  reasons. IPC and TCP should be fast enough not to be a bottleneck in full node operation.

The disadvantage of this approach is that message de-/serialisation is much slower than passing memory pointers around.
Benchmarks and optimisations will be employed to ensure that IPC is not a bottleneck for Tari node operation.

**Note:** This proposal references inter-process communication, but the proposal applies equally well without
modification to inter-thread communication on the same process.

## Description

### Overview
[ØMQ] is a mature distributed communications framework. It's very lightweight, is very fast and has excellent documentation.

This document describes MessageBus, a messaging system that offers safe, scalable and flexible asynchronous messaging.

The MessageBus architecture follows a fan-in, fan-out pattern.

An arbitrary number of `MessagePublishers` connect to a single `MessageBroker` object that passes messages on to an
arbitrary number of `MessageSubscribers`.

Every `MessagePublisher`, `MessageSubscriber` and the `MessageBroker` owns a [ØMQ] socket and each operates in their own
thread or process. Sockets communicate via TCP, Unix sockets, or (if inter-thread) IPC or memory sharing.

### Message Publishers
`MessagePublishers` are implementation-specific classes that produce messages. For example, one implementation might
relay `Transaction` messages from from peer nodes over I2P, another from HTTP and a third from a client wallet. All
three transaction messages are of exactly the same format. Thus receivers of `Transaction` messages don't care where they
come from.

Messages are serialized into the binary [MessagePack](https://msgpack.org/index.html) format before being put onto the wire.

The MessagePublishers MAY format messages if desired, but MUST deliver messages to the MessageBroker in MessagePack format.
The data in the messages is invisible to `MessageBus`. All it sees is a string of bytes.


The basic structure looks as follows:

```text
         Source A                  Source B                Source C
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

The MessageBus can also be augmented with 0MQ pipelines (using PUSH, PULL, ROUTER and DEALER sockets), which are
analogous to node.js stream pipes. For example, a `MessageSubscriber` might subscribe to new `Transaction` messages and
pass them to a transaction validation pipeline that throws out invalid messages and only passes on validated
transactions.

### Message format

Messages consist of two [ØMQ] frames. The first frame is a 32-bit integer indicating the message type. The second frame
contains the message payload.

### Message Broker

The `MessageBroker` receives messages into a queue from every attached `MessagePublisher`. Every message received is
copied and sent to *every* `MessageSubscriber` that has subscribed to that message type.

The `MessageBroker` does not modify the message in any way.

### Message subscribers

A `MessageSubscriber` connects to the `MessageBroker` and specifies a filter for messages it is interested in, based on
the message type field.

MessageSubscribers subscribe to *either* one message type, or all messages.

Each `MessageSubscriber` also manages a request-response socket that allows other threads or processes to submit control
messages to `MessageSubscriber` and/or request status updates.

 [ØMQ]: http://zguide.zeromq.org