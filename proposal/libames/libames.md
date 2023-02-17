# libames

It should be possible to build apps outside Urbit that communicate with Urbits using the Ames protocol.  For this to work, we need an implementation of Ames written in an Earth language that other Earth programs can easily embed.  Languages that might be embeddable enough include Lua, Zig, Rust, Chez Scheme, Guile, and of course C itself.

This project will be broken into four major milestones:

- moon-specific: can only talk to sponsoring planet, no persistence
- persistent: writes packets to disk to work across moon restarts
- universal: can pretend to be any ship and talk to any ship, tracks azimuth
- portable: can run on Windows and MacOS, not just Linux

## Phase I: Moon-Specific

A lot of clients, such as IoT devices and bridges between Urbit and other networks (e.g. blockchain nodes and STUN servers), only need to talk to one planet.  A moon-specific libames is the smallest useful subset of Ames functionality.

## Phase II: Persistent

However, if the "moon" device restarts, it will need to be breached, which is annoying and limits the uses.  A true Ames implementation persists incoming packets to nonvolatile storage, and it can also remember negative acknowledgments it's emitted (Urbit's equivalent of HTTP 5xx errors), to honor Ames's transactionality.  In practice, anything other than a moon or comet needs persistence to avoid frequent on-chain breaches.

## Phase III: Universal

A "universal", fully featured libames is indistinguishable from a "real" Urbit over the network.  Achieving this property entails tracking Azimuth so that the node can communicate with any ship.  It also requires handling breaches of other ships, forwarding packets through itself, and tightening and loosening routes.

## Phase IV: Portable

Finally, a portable library will work on any host OS.  The first versions of this could be written just for Linux, to keep the project smaller.  Many, if not most, external clients will likely be run on cloud servers that run Linux, so an ecosystem could develop without full portability from the beginning -- although paying the price for portability up-front might lessen the total amount of work compared to rewriting pieces of libames for cross-platform support later.

# An Aside on Embedding Nock

An alternative approach to libames would embed a Nock interpreter and use that for most of the meat of the system, probably with a bespoke Nock kernel.  The library would use events and effects to deliver events into the Nock kernel and receive effects.  An effect might be "look up this galaxy's DNS entry", and an event would be "here's the DNS entry for this galaxy".  Receiving a UDP packet would be an event; emitting one would be an effect.

Such a design would have some advantages, such as sharing Hoon code for Ames business logic and memory safety for at least some parts of the system.  Current Vere is not easy to embed, though, so making Vere embeddable would have to be done first, and it's not a trivial project.

# Basic Structure

There are two basic layers in libames: the packet layer and the message layer.  The packet layer encodes and decodes packets into and out of a UDP socket.  The message layer splits a message into a fragment per packet, reassembles those packets into a message on the receiver, and maintains message-level invariants, such as exactly-once delivery.

Like ames.c, libames needs to bind a UDP port and perform DNS lookups for galaxies.  It should support callback functions for when an outgoing message is acked and when an incoming message is received.  Packet-level events should be handled internally and not exposed to client programs.

The basic interface to libames should take in a jammed (serialized) message to send, as a byte buffer, and call callbacks with jammed received messages.  Since different clients could be written in different languages, they will likely want to represent nouns differently, so libames should not constrain clients' in-memory noun representation by privileging one layout in its interface.  A client written in any language should be able to send and receive buffers across an FFI boundary without too much trouble.

## Comparisons to the Ames Vane

The Ames vane has two almost entirely independent modules, the `+message-pump` for sending messages and `+message-sink` for receiving messages.  This structure will likely be replicated in libames, but data structures and effects will differ due to being a Unix program instead of an Arvo program.

Data structures and timers in libames will likely be different from the Ames vane.  The map from ship to ship state should probably be a hashtable instead of a treap.  Similarly, outgoing packet queues should probably be vectors instead of treaps.

Ames emits effect data structures, since Urbit is a message-passing system, but libames will instead call the relevant callback to deliver information to the client program, and instead of emitting timer effects to Behn, libames will use timer syscalls and signal handlers (or the windows equivalent).

The initial version of libames, since it only talks to one hard-coded ship, could omit the map from ship to ship state in favor of a single ship state.

Packet forwarding code could be copied from Vere's ames I/O driver, although any use of Vere nouns would likely need to be changed to use libames's own noun representation.  Libames should not use Vere's loom, which is difficult to embed.

# Persistence

The easy way to persist data is probably to embed SQLite, as is tradition.  Incoming packets need to be persisted before their acks are emitted, and nacks we emit need to be persisted until a certain number of messages after them have been received.

For performance, a client program might want to run libames in its own thread.  This would allow disk writes to happen in parallel with handling incoming messages, although keeping everything thread-safe would make the system more complex.  The first version of libames should probably not try for thread safety.

Note: It might be tricky to get the transactionality right when receiving the final packet in a message.  I think you need to write the packet to disk before processing it, then wait for the client to ack or nack the message, then 

Note:  I think you'd want to run all of libames in a thread, instead of just separating a thread for disk writes, but I could be wrong.

# Azimuth Tracking

In order to communicate with arbitrary ships on the network, libames needs to subscribe to their PKI information: public key, life, rift, and sponsor.  This information is derived from Ethereum data.  Data for L1 ships is stored directly on-chain.  Data for L2 ships must be derived from on-chain data by running the "naive rollup" transactions posted on-chain.  Real Urbit ships each listen to the chain and run their own L2 transactions.

A library that tracks the Azimuth PKI (let's call it "libazimuth") should be an independent library that can be wired up to communicate with libames.  To communicate with a peer ship, libames needs to ask for the peer's networking public key, life, rift, and sponsor.  It also needs to be notified if any of that information changes.

Libazimuth also needs to either run its own naive rollup or communicate with a provider ship who does.  This introduces a bootstrapping problem, which could be solved by initializing libazimuth with PKI information for the provider ship.

Question: could the provider ship lie about its own PKI state?  If so, would it make sense to get a second opinion from somewhere else, like a different ship on the network?  If our libames ship is a moon, its provider could be its sponsoring planet, in which case this should be safe, but if pretending to be a planet, maybe this is an issue.

Building an embeddable libazimuth library might be better off as its own project, although libames and libazimuth might end up with nontrivial mutual dependencies if using the provider model.

# Further Work

This document only describes the current Ames protocol, not the remote scry protocol.  By the end of 2022, libames will probably need to implement both protocols in order to engage in full communication with other ships.
