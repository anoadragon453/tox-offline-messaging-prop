# Tox DHT Offline Messaging

## Overview

This proposal aims to provide a general and technical walkthrough of an
offline messaging protocol for use with the Tox protocol using its existing
Distributed Hash Table (DHT). Messages are only stored in the DHT when the
message receiver is currently offline. The sender does not have to be online
for the receiver to retrieve messages.

As this method uses the DHT, it is currently UDP only and will not work
unless UDP is enabled. This restriction will hopefully be lifted with
further work and development of the Tox protocol.

## Terminology
PoW:
  Proof-of-Work -- Determines CPU work necessary in order to store an offline message on the DHT.

SENDER:
  The friend of RECEIVER sending an offline message to her when she is offline.

RECEIVER:
  The friend of SENDER receiving an offline message.

STORAGE_NODE:
  One of potentially many nodes storing offline messages for other users.

OM Keys:
  Offline Messaging public / private keypair.

## DHT Extension

The DHT must be extended in order to be able to store offline messages. Tox
Clients should have a reasonable default set for DHT storage size and
optionally allow users to change it. This space will be used for storing
messages of other users.

## Storage Space Allocation

The RECEIVER must request a certain amount of storage space for incoming
offline messages on a number of STORAGE_NODEs. A PoW job must be completed
by the RECEIVER to do this. The more space required per node, the higher
the PoW difficulty.

## Inserting Messages

A user creates a new public and private key pair (Current DHT keys are
expected to be ephemeral) per friend to encrypt offline messages.  Messages are stored
amongst an array of known nodes.  Users then store messages they want to send
to a store on those nodes after optionally completing a PoW function. 

## Retrieving Messages

Users simply check the SENDER's STORAGE NODES in the DHT when they come
online.  If any messages are available they will download them from the nodes
holding them and decrypt them with their OM keys. Nodes holding messages
should then delete them from their store.

## Spam

As a Proof-of-Work (PoW) hash function is used to insert messages into the
DHT. The SENDER and RECEIVER must complete a certain amount of
computational hashing before they can send a message. Messages that do not
properly complete the hashing function will be dropped by network clients
upon receival. Thus is makes spamming a single user or the network
unreliable and costly.

The difficulty of the PoW is increased relative to the storage space
required to storage messages, to ensure a malicious SENDER cannot simply
fill a STORAGE_NODE's available message space with a single message.
However, very small messages could game this multiplier, and thus even the
minimum PoW difficulty must be reasonably large.

Messages should also expire eventually, and thus a higher PoW difficulty
should be required for a longer message TTL (time to live).
        
## Privacy

Tox does not inherently provide anonymity, however this will not expose any
more information about the user than using the DHT in Tox already does.

A middle node can correlate which IPs are talking to each other by seeing who
requests messages sent by whom, however one could enumerate this simply by
watching traffic from either the SENDER or RECEIVER so this does not provide
much new information.

## Security

Messages are encrypted by the SENDER with two layers, one layer for the
STORAGE_NODE which encrypts the metadata necessary to see whom the message is
intended for, and another layer for the RECEIVER that includes the message
itself. Users will not be able to read the content of other user's messages
unless intended for them.

# Technical

## Packet details

Need a packet type for sending message, sending message TTL updates,
retreiving messages from STORAGE_NODEs, sending PoW requests and responses.

## Packet encryption

There are 2 layers of encryption for a message, first the SENDER encrypts the
message using the agreed/shared key. (TODO: Is this the Tox per-friend key we
already use for messaging, or is this OM specific?) 

The SENDER then encrypts that message for us using the SENDER -> RECEIVER
shared key. That way the STORAGE_NODE can receive the message, encrypt it,
and write it to disk forgetting it exists until requested. Then once
requested, it'll send the already encrypted message to the reciever, and
delete the data on disk.  Invalidate the current POW, and the whole cycle
will start over again.

```
SENDER -> DHT_Node [The Message]
    encrypts to friend       --> [SENDER : RECEIVER  key_pair || The Message]
    encrypts to storage_node --> [sender:DHT_node shared key [SENDER : RECEIVER  key_pair || The Message]]
DHT_node  
    decrypts from sender     --> [sender:friend shared key || The Message]
    encrypts to friend       --> [DHT_node shared key [SENDER : RECEIVER  key_pair || The Message]]
RECEIVER [sender:DHT_node shared key 
    decrypts the message from the node   --> [SENDER : RECEIVER  key_pair || The Message]
    decrypts the message from the friend --> [The Message]
RECEIVER DONE --> display [The Message]
```

## PoW function

PoW job is computed on STORAGE_NODE (low cost) then sent to either SENDER or
RECEIVER to compute for proof (high cost).

Looking at BitMessage's implementation [1], SHA512 is used.

Their [function for PoW difficulty](
https://bitmessage.org/wiki/File:POW_forumla_2.png ) looks substantial.  The
payloadLengthExtraBytes var is to give weight to smaller messages, garbage
bytes are not actually appended to message packets.

# References

[0] [Dpush -- A scalable decentralized spam resistant unsolicited messaging protocol](dpush.md)
    New implementation is not really similar to Dpush anymore, but fun read nonetheless.
    
[1] [BitMessage Spec v3 PoW Summary](https://bitmessage.org/wiki/POW) 


## TODO
    Consider or reject
        allowing friends to store messages for shared friends (maybe with a lower proof of work)
        multi-device compatability
        allowing senders to increase POW space
        Define and adapt use case to systems where tox may get killed trying to compute POW (android, iOS, etc.)
            Persistant notification keep processes alive, no? Android may still kill them if resources are low though.
            yes, but doze dosen't like you using the CPU that much -- understandable
            And iOS kills them after ~10m iirc.
                @dvor would know the answer to this. Also if you get push messages, and the corresponding wakeup from iOS, and then use a lot of network, or battery life. iOS will just stop sending you messages.
                
