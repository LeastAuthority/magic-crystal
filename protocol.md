Magic Crystal
=============

HOWTO use magic-wormhole as a transport mechannism for Dark Crystal
shards.



Given: you have: some secret data to socially backup; an Ed25519
public-key for each intended recipient; an Ed25519 private/public key
for the sender. (That is, some PKI so that recipients and sender
already know public keys for each other)

If only the sender should be able to read the "secret data" it should
first be encrypted (XXX right?) by the sender. Otherwise any
"threshold" number of recipients can retrieve the "secret data" (this
is intended; some use-cases _want_ this).

XXX: if we use e.g. "github ssh keys" or "OpenPGP keys" does this
     scheme leak to observers _who_ is storing shards for _whom_? (That is,
     does it tell an observer who to apply the rubber-hose to in order to
     receive enough shards to recover the secret data?)

 - produce shards (up to step 8, above)
 - put each shard into an "outbox" with intended recipient
     - e.g. save shard into a file corresponding to public-key of recipient
 - create magic-wormhole with any remaining intended recipient
 - confirm public-keys:
 - each side sends a "Challenge" message
    - timestamp
    - random nonce
 - each side responds with a "Response" message
    - Ed25519 signature of the received Challenge
 - if the signature doesn't match, disconnect
 - select the shard from the "outbox"
 - send the shard to the recipient ("Shard" message)
 - receiving side closes wormhole
 - sending side deletes shard file from "outbox"

The above protocol would be repeated for each recipient.
