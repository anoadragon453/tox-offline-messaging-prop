# Tox DHT Offline Messaging

## Overview

This proposal aims to provide a general and technical walkthrough of an
offline messaging protocol for use with the Tox protocol using its existing
Distributed Hash Table (DHT). Messages are only stored in the DHT when the
message receiver is currently offline. The sender does not have to be
online for the receiver to retrieve messages.

As this method uses the DHT, it is currently UDP only and will not work
unless UDP is enabled. This restriction will hopefully be lifted with
further work and development of the Tox protocol.

## DHT Extension

The DHT must be extended in order to be able to store arbitrary data. Tox
Clients should have a reasonable default for DHT storage size set and
optionally allow users to change it. This space will be used for storing
messages of other users.

## Inserting Messages 

Analagous to dpush [0], a user creates a new public and private key pair
(Current DHT keys are ephemeral) to sign their inbox location. The inbox is
stored in the DHT on computers who are in charge of that key range. (TODO:
Should the public key be hashed?) Users then store messages they want to
send to a store in that inbox's location. 

(TODO: Inbox location can be changed if necessary, as long as new location
is always signed by OM key. Is this a useful feature? If you change you
lose all old messages stored in old locations).

## Retrieving Messages

Users simply check their inbox location in the DHT when they come online.
If any messages are available they will download them from the nodes
holding them and decrypt them with their OM private keys. Nodes holding
messages should them delete them from their store.

(TODO: Similar to what dpush does, we could have the receiver scan sender's
public keyspaces for messages intended for them, thus requiring no PoW from
the sender. I feel like this can reintroduce spam though, however the
receiver is fully in control of what they want to pull at this point.)

## Spam

As a Proof-of-Work (PoW) hash function is used to insert messages into the
DHT. The sender must complete a certain amount of computational hashing
before they can send a message. Messages that do not properly complete the
hashing function will be dropped by network clients upon receival. Thus is
makes spamming a user unreliable and costly.

## Privacy

Tox does not inherently provide anonymity, however this will not expose any
more information about the user than using the DHT in Tox already does.
(TODO: A correlation between who is talking to whom may be possible?).

## Security

Messages are encrypted with the same encryption they normally are, and
users will not be able to read the content of other user's messages unless
intended for them.

# Technical

Technical details below.

## Packet details

Need a packet type for sending message, sending message TTL updates,
retreiving messages from inbox.

## PoW function

Something that can be adjusted, look into BitMessage.

# References

[0] https://morph.is/v0.8/dpush-whitepaper.odt
