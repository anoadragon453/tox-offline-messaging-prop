# Tox DHT Offline Messaging

## Overview

This proposal aims to provide a general and technical walkthrough of an offline
messaging protocol for use with the Tox protocol using its existing Distributed
Hash Table (DHT). Messages are only stored in the DHT when the message receiver
is currently offline. The sender does not have to be online for the receiver to
retrieve messages.

As this method uses the DHT, it is currently UDP only and will not work unless
UDP is enabled. This restriction will hopefully be lifted with further work and
development of the Tox protocol.

## Terminology
PoW:
  Proof-of-Work -- Determines work necessary in order to store an offline
  message on the DHT.
  (Currently WIP on what the exact method of PoW used will be)				

SENDER:
  The friend of RECEIVER sending an offline message to her when she is offline.

RECEIVER:
  The friend of SENDER receiving an offline message.

STORAGE_NODE:
  One of potentially many nodes storing offline messages for other users.

DHT Storage Key:
  Key pair whose public component lies within the Tox's DHT Storage Keyspace
  and points to the location in the network containing stored data between two
  users.

## DHT Extension

The DHT must be extended in order to be able to store arbitrary data. Tox
Clients should have a reasonable default set for DHT storage size and
optionally allow users to change it. This space will be used for storing
messages and/or files of other users.

A Chord-style routing algorithm will be used to find and place content within
the network. Nodes self-organize into a ring, where portions of the
circumference are dedicated to storing a portion of possible data keys. Toxcore
will keep a list of nodes spaced incrementally along the ring, and lookups will
use these lists recursively across nodes in order to get closer and closer and
eventually find the node or nodes covering their requested key space.

Lookup time in Chord is approximately log(n) time, where n is the number of
nodes in the DHT. The pre-existing known node list will not be affected by this
new change. Instead, a new list will be created that is similar in
functionality to Chord's own 'finger list'. Instead of finding nodes that are
closer and/or far away (in terms of Tox ID 'distance'), we can store nodes that
are increasingly, exponentially 'far' away in the ring.  This list is generated
and maintained seperately from the pre-existing node list and is only created
if the user has UDP enabled.

Data retention is an issue that all DHTs face. As is the ephemeral nature of
peer-to-peer systems, nodes join and leave the network randomly and without
warning. Several methods can be used to combat this. One such method uses
'multiple coordinate spaces to simultaneously improve both lookup latency and
robustness' [1] of the DHT. This method was used in Content Addressable
Networks, but can easily be applied to Chord-style DHTs by having each node sit
in a key range on multiple rings. This ensure data is backed by at least r
nodes, where r is the amount of rings each node is a part of.

## DHT Storage Key

The DHT Storage Key acts as a location identifier for offline data as well as
the authentication to modify, update and delete data in the Tox DHT. The
private component of the key should only be known to the SENDER and RECEIVER.
Data stored in the network is signed with this key, and can only be replaced
with a newer version of the data signed with the same key. The public component
of the key defines where in the Tox DHT the data should be stored. It can be
thought of as similar to a zipcode. The public component is used to find out
which nodes should act as your STORAGE_NODE, as each STORAGE_NODE handles a
certain range of the keyspace.

TODO: Keys should change during every "online session". We should keep at least the last key in case we still need to retireve old messages. (Maybe we only need the public component so we know where to look in the network?)

Keys are generated on friend's first connection to each other, and several
failsafes are in place to prevent friends from becoming out of sync. A check is
performed upon receiving an ONLINE packet from a friend for a DHT Storage Key
with that friend. If one isn't found, a request for an existing key is sent to
that friend. If they also do not posess a key, a new key is generated and is
sent to the friend. This allow both new friends and existing friends from
before offline messaging was implemented to take advantage of the feature
without having to re-add each other.

The key is designed to be re-generated every time both friends are online.

## Storage Space Allocation

STORAGE_NODEs allocate a certain amount of space for DHT data. For SENDER to
store data in that space, the SENDER must complete a PoW function that the
STORAGE_NODE deems valid.

## Inserting Messages

Data is encrypted with the long term public key of the offline
friend they are sending too.

As the extended DHT acts as a distributed key/value store, messages are stored
under the STORAGE_NODE handling the DHT Storage Keyspace in which the DHT
Storage Key known to both the SENDER and RECEIVER resides within.

There is no limit on the size of data that may be stored, however a STORAGE_NODE
is able to refuse the storage of data if the stored data exceeds a local limit.

## Retrieving Messages

Users simply check the vault of their STORAGE_NODES in the DHT when they
come online.  If any messages are available they will download them from the
nodes holding them and decrypt them with their OM keys. Nodes holding messages
should then delete them from their store.

## Updating Existing Data



## Re-announcing

As the DHT is considered to only be temporary storage, each piece of data stored
within it is given a TTL value (Time-to-live). This value is determined when the
data is stored and STORAGE_NODEs keep track of each data's TTL in their cache,
which is procedurally decremented, finally removing them when their TTL value
reaches zero.

A SENDER can _re-announce_ data by completing another PoW function, resetting
that data's TTL to its original value (or longer/shorter, depending on the
PoW's difficulty).

All offline data on a storage node is cleared on shutdown.

## Spam

As a Proof-of-Work (PoW) hash function is used to insert messages into the DHT.
The SENDER must complete a certain amount of computational hashing before they
can send a message. Storage messages that do not properly complete the hashing
function will be dropped by network clients upon retreival. Thus spamming a
single user or the network is unreliable and/or costly.

The difficulty of the PoW is increased relative to the storage space required
to storage messages, to ensure a malicious SENDER cannot simply fill a
STORAGE_NODE's available message space with a single storage request.  However,
very small messages could game this multiplier, and thus even the minimum PoW
difficulty must be reasonably large.

Messages should also expire eventually, and thus a higher PoW difficulty
should be required for a longer message TTL.

## Privacy

Tox does not inherently provide anonymity, however this will not expose any
more information about the user than using the DHT in Tox already does.

As Offline Messaging requires UDP, this cannot be used in conjunction with Tor,
as Tox-over-Tor (ToT) requires TCP-only connections. This may change in the
future with a TCP implementation of the DHT, but that is out of the scope of
this design.

A storage_node can correlate which IPs are talking to each other by seeing who
requests messages with the same key, however one could enumerate this simply by
watching traffic from either the SENDER or RECEIVER so this does not provide
much new information.

## Security

Messages are encrypted by the SENDER with two layers, one layer for the
STORAGE_NODE which encrypts the metadata necessary to see whom the message is
intended for, and another layer for the RECEIVER that includes the message
itself. Users will not be able to read the content of other user's messages
unless intended for them.

Perfect Forward Secrecy is ensured on a similar level to how Toxcore currently
handles messages encryption keys - on a per-session level. In the future keys
may be changed on the per-message level, but this is out of the scope for this
design.

Messages are encrypted with keys generated between each Offline Session. An
Offline Session is the period of time between when SENDER and RECEIVER are
online at the same time. During this time, both parties attempt to retrieve any
remaining offline data from the DHT, and generate a new set of Offline Session
keys when they both come online again for encrypting further offline data.

Clients should keep a record of the public component of the last 1 key in case
they find a trailing message in the DHT during a later Offline Session.

# Technical

## Packet Details

Need a packet type for generating/checking for available DHT Storage Keys,
storing data (including data/TTL updates), getting data, retrieving messages
from STORAGE_NODEs, sending PoW requests and responses, and those for building
the DHT (joining, leaving, updating key range).

Checking if friend already has DHT Key:

DHT_KEY_DOES_EXIST

| Length | Contents      |
|--------|---------------|
| 1      | uint8t (0x??) |

Newly generated key sent to friend:

DHT_NEW_KEY


PoW/PoN request packet. Sent by STORAGE_NODE to a SENDER. SENDER should respond
with the proof in their add/update requests. Proof should only be valid for
a finite amount of time. Proof should be invalidated as soon as a valid
action has been carried out with it.

PROOF_REQUEST

| Length         | Contents                            |
|----------------|-------------------------------------|
| 1              | uint8t (0x??)                       |
| ChallengeID    | uint8t Identifier for Proof Request |
| Challenge Data | ?                                   |

Generic data to be stored in the DHT. Can contain other Tox Packets.

DHT_OFFLINE_DATA

| Length      | Contents                            |
|-------------|-------------------------------------|
| 1           | uint8t (0x??)                       |
| ChallengeID | uint8t Identifier for Proof Request |
| ?           | Generic Data Store                  |

...

## Packet encryption

There are 2 layers of encryption for a message, first the SENDER encrypts the
message using the specific private key generated for offline data storage. In
the case

The SENDER then encrypts that message for us using the SENDER -> RECEIVER
shared key. That way the STORAGE_NODE can receive the message, encrypt it,
and write it to disk forgetting it exists until requested. Then once
requested, it'll send the already encrypted message to the reciever, and
delete the data on disk.  Invalidate the current POW, and the whole cycle
will start over again.

```
SENDER -> DHT_Node [The Message]
    encrypts/signs to friend --> [SENDER : RECEIVER  key_pair || The Message]
DHT_node  
    Stores encrypted message, will send to anyone who requests it, as they could just intercept on the
    line anyways.
RECEIVER [sender:DHT_node shared key
    decrypts the message from the friend --> [The Message]
RECEIVER DONE --> display [The Message]
```

## PoW function

PoW job is computed on STORAGE_NODE (low cost) then sent to either SENDER to
compute for proof (high cost). We could use a heavy PoW for storing new data in
the DHT, and a light PoW for data updates. Testing required.

Looking at BitMessage's implementation [0], SHA512 is used.

Their [function for PoW difficulty]
(https://bitmessage.org/wiki/File:POW_forumla_2.png) looks substantial. The
payloadLengthExtraBytes var is to give weight to smaller messages, garbage
bytes are not actually appended to message packets.

Also consider [Ethereum's PoW function]
(https://github.com/ethereum/wiki/wiki/Ethash), looks a little bit more
complicated to implement though.

A cryptocoin based solution may be beneficial both in potential features as well as a
potential source of income for the Tox Project. Further investigation needed.

Proof of Network may solve issue of required CPU resources while remaining fair
to each device (in average amount of time to store data offline).

## UX/UI

When a friend is added, a UI option appears for setting up offline messaging, or
this could be done automatically. When a user sends a message to an offline friend,
the client should notify the user when the message has been stored successfully. One
can just reuse the existing indicator of a message sending complete, or implement
something specific.

At the moment storing files offline is not supported, but with the current model this
should be trivial to add for small files.

# References

[0] [BitMessage Spec v3 PoW Summary](https://bitmessage.org/wiki/POW)

[1] [Looking up Data in P2P Systems](https://people.eecs.berkeley.edu/~istoica/papers/2003/cacm03.pdf)


## TODO
	Consider or reject
		allowing friends to store messages for shared friends (maybe with a lower proof of work)
		multi-device compatability
		Define and adapt use case to systems where tox may get killed trying to compute POW (android, iOS, etc.) - Essentially pick a better PoW algo than straight hashing
			Persistant notification keep processes alive, no? Android may still kill them if resources are low though.
			yes, but doze dosen't like you using the CPU that much -- understandable
			And iOS kills them after ~10m iirc.
				@dvor - "On iOS app would have around 30 seconds after receiving push notification to do some work in background. To speed up everything it would be good to know which STORAGE NODES to connect to. Maybe push notification can contain some encrypted metadata with information about SENDER."
	Have PoW do something more meaningful than just hashing for the sake of killing cycles.
		i.e. Medical research, folding, generating something etc.
		Or just don't require high CPU cycles at all.
