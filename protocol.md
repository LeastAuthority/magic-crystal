Magic Crystal
=============

HOWTO use magic-wormhole as a transport mechannism for Dark Crystal
shards.


Prerequisites
-------------

You have:

* some secret data to socially backup;
* an Ed25519 public-key for each intended recipient;
* an Ed25519 private/public key for the sender.

That is, some PKI (Public Key Infrastructure) so that recipients and
sender already have public keys for each other.


Protocol Overview
-----------------

If only the sender should be able to read the "secret data" it should
first be encrypted (XXX right?) by the sender. On the other hand, some
use-cases may want any "threshold" number of Custodians (recipients)
to be able to read the secret data.

XXX: if we use e.g. "github ssh keys" or "OpenPGP keys" does this
     scheme leak to observers _who_ is storing shards for _whom_? (That is,
     does it tell an observer who to apply the rubber-hose to in order to
     receive enough shards to recover the secret data?)

 - produce shards (up to step 8, above) which is some bytes
 - put each shard into an "outbox" with intended recipient
     - e.g. save shard into a file corresponding to public-key of recipient
 - create magic-wormhole with any remaining intended recipient
    - either recipient or sender may create the code
    - the other one enters the code to complete the wormhole
 - once connected, confirm public-keys:
 - each side sends a "Challenge" message
    - timestamp
    - random nonce
 - each side responds with a "Response" message
    - Ed25519 signature of the received "Challenge" message
 - if the signature doesn't match, disconnect
 - select the shard from the "outbox" (based on the public-key of the other side)
    - if there is no shard for this recipient, send an Error message and disconnect
 - send the shard to the recipient ("Shard" message)
    - for large shards, a Transit connection should be opened (or Dilation)
 - receiving side sends "Received" message (_after_ storing the Shard)
 - sending side deletes shard file from "outbox"
 - sending side closes the wormhole

The above protocol is repeated for each recipient.
Note that the sender has confirmation that the Shard is written to the recipients disk _before_ they delete the outbox copy.
Simiarly, the recipient receives confirmation that the sender has deleted their copy (by waiting for the wormhole to close).


Example: OpenPGP
----------------

Many people already know an OpenPGP key for (some of) their colleagues.

An application could leverage this as "the PKI" to use for their secret-sharing needs.

An application with some secret data may leverage Magic Wormhole to share Dark Crystal shards.

That is, we have:
* some secret data to back up amongst several participants
* a list of Ed25519 public-keys of those participants
* a private (and public) Ed25519 keypair of the sender

To back up the secret data amonst the recipients:

1. Produce the Shards
2. Save the Shards in an outbox, keyed on recipient
3. For each {recipient, Shard} pair:
    * connect a magic-wormhole between sender and recipient
    * confirm identities
    * transfer Shard
    * after confirmation, delete Shard from outbox


1. Produce the Shards
`````````````````````

- Dark Crystal (can we use the public-keys _directly_ in the protocol?
  or will that leak stuff to passive observres, like "who to
  rubber-hose for the shards"?)

2. Put Shards in Outbox
```````````````````````

- Write Shard bytes to a file named `"<base64 pubkey>.shard"`
- ...


3. Transmit Shards to Each Recipient
````````````````````````````````````

This protocol is repeated once for each Shard + Recipient pair in the outbox.
Note that if one of the Shard holders is the Sender themselves, this Shard should simply be stored and the below protocol not followed for that Shard.


- Sender and Receiver set up a magic-wormhole:
   - either the Sender creates a code and the Receiver enters it or;
   - the Receiver creates a code and the Sender enters it
   - software SHOULD allow either way whenever possible
   - the code SHOULD be transmitted securely
- A "wormhole" channel is now established.
    - If the secret data is large, Shards will be large -- in such a case, both sides SHOULD set up a direct Transit connection to the peer
    - XXX what is "large"?
    - For Shards below this threshold, the mailbox itself MAY be used
- Both sides send a "Challenge" message
- Both sides sign the 8 bytes of timestamp concatenated to the 32 bytes of nonce in the Challenge
- Both sides send back a "ChallengeResponse" message
- Each side confirms that the signature matches; if not send "Error" message and disconnect.
- The "Sender" side sends the Shard corresponding to the public-key of the Recipient
  (If there is no matching Shard, send "Error" message and disconnect)
- Once the Shard is written to disk, the Recipient sends an "Acknowledge" message
- Once the Sender side receives an "Acknowledge" message it deletes the Shard from the outbox
- The Sender side closes the mailbox
- The Receiver side sees the mailbox close.
- the protocol is complete


```python
class Error:
    message: str
```

```python
class Challenge:
    timestamp: int  # seconds since epoch
    nonce: bytes  # 32 bytes
```

```python
class ChallengeResponse:
    signature: bytes  # Ed25519 signature of [8 bytes timestamp || 32 bytes nonce]
```

```python
class Acknowledge:
    checksum: bytes  # blake2b hash of the Shard
```
