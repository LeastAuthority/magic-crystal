Magic Crystal
=============


<!--Not really a specification, can give recommendations, but not capitalized specification-y things, more prose style. Primary objective: someone building an app with these.-->

This document provides a specification and recommendations for building an application for [secret sharing](https://darkcrystal.pw/what-is-secret-sharing/) with [Dark Crystal](https://darkcrystal.pw/), using [Magic Wormhole](https://magic-wormhole.readthedocs.io/en/latest/welcome.html) as a transport mechanism, and the Public Key Infrastructure (PKI) of your choice.

Readers of this document are software developers who have:

    - some secret data to split up and store;
    - public-key identities for participants;

...but lack:

    - a way to communicate

We show how to use [Magic Wormhole](https://magic-wormhole.readthedocs.io/en/latest/welcome.html) for transporting pieces of the secret to participants.

TODO:
 - go over spec
 - look at the questions in ref to ^
 - ask Peg about outstanding questions



Prerequisites
-------------

_SB: If the Protocol Overview and the Example sections will be merged, we should change the above to OpenPGP specifically._

The idea is that you "bring your own" PKI, so SSH keys (or OpenPGP) are just examples of that .. the "protocol" should be more generic about that part (although they do need to be ed25519).

_SB: Add TOC when doc is ready_

Background
----------

[Dark Crystal](https://darkcrystal.pw/) facilitates storing **secrets** (i.e., sensitive data such as access credentials or cryptographic keys) between trusted peers, rather than relying on individual responsibility or third party providers.
Dark Crystal is **transport agnostic**, meaning that in can be used in combination with any transfer/communication mechanisms.

The transfer mechanism used in this document is [Magic Wormhole](https://magic-wormhole.readthedocs.io/en/latest/welcome.html), a tool for transporting arbitrary-sized files or directories between two computers.
Magic Wormhole relies on the assumption that the two humans in charge of the two computers can communicate with each other either in person, on the phone, or via some messaging service.

In addition, some PKI (Public Key Infrastructure) is needed to carry out the protocol described here, and all recipients and sender already need to have public keys for each other.
We use [OpenPGP](https://www.openpgp.org/) as an example in this document.



### Why Dark Crystal?

Encrypting files and messages provides a high level of protection from data being compromised.
However, many people forego using encryption because they are worried that losing their encryption keys will result in them losing access to their data.

Dark Crystal aims to solve this problem by providing a set of protocols that enable users to split their data into **shards** and then distribute these shards to a set of their trusted peers (called **custodians**).
It is not possible to reconstruct the original secret from a single shard, so the secret remains protected if a shard is compromised.
On the other hand, a subset of shards is enough to recover the original secret, so the owner's access to the secret is preserved even if a shard is lost.

See the Dark Crystal documentation for more details about [the challenges of adopting encryption methods](https://darkcrystal.pw/why-dark-crystal/) and [potential use cases](https://darkcrystal.pw/use-cases/) for secret sharing with Dark Crystal.


### Why Magic Wormhole?

Dark Crystal does not specify a transport mechanism for distributing the shards between custodians.
This means that custodians either need to sign up for a specific service (like [Secure Scuttlebutt](https://scuttlebot.io/more/protocols/secure-scuttlebutt.html)) for securely distributing the shards, or resort to less secure methods such as email or instant messaging.
The latter channels also have the disadvantage that the data is stored on the service provider's servers, increasing the risk of it being compromised.

Magic Wormhole provides secure file transfer **directly between two computers**, without storing the secret on third-party servers.
Integrating Magic Wormhole into an application using Dark Crystal provides a secure transport mechanism without the need for the custodians to sign up for a secure transport service or using less secure channels.
While Magic Wormhole still relies on an external communication channel (e.g. email, instant messaging or voice) to relay the wormhole code, both the likelihood and the negative consequences of this code being compromised are minimal compared to the secret itself being shared on one of these channels.

A further advantage of Magic Wormhole is that it is **identity-less**, meaning that the participants do not need to supply personal information or obtain access credentials to complete the transfer.
It is important to note that, while the file transfer itself does not rely on identities, the secret sharing process itself is not entirely identity-less.
The basic premise of Dark Crystal is that the secret owner trusts their custodians, which presumably implies that they know at least some aspects of their identity.
In addition, using a PKI also means that public/private keys are assigned to (partially) identified individuals.
The advantage of using Magic Wormhole is that they do not need to disclose this to any other parties.


Application Specifications
--------------------------

### Prerequisites


* some secret data to socially back up;
* an Ed25519 public key for each intended recipient (custodian);
* an Ed25519 private/public key for the sender (secret owner).

_SB: is Ed25519 a requirement, or would other PKIs work as well?_

meejah: yes, they need to be Ed25519 per Dark Crystal.


### Protocol Overview

_SB: The **Protocol Overview** and the **Example** sections are very similar right now. Are we going to add more concrete details to the **Example** section later? If not, I would merge these two section into one, because it feel very repetitive as it is now._

meejah: probably being more concrete in the example would be good

If only the sender should be able to read the **secret data**, it should first be encrypted by the sender.
On the other hand, some use-cases may want any **threshold** number of Custodians (recipients)
to be able to read the secret data.

XXX: can we help readers decide which to use?

XXX: if we use e.g. **github ssh keys** or **OpenPGP keys** does this
     scheme leak to observers _who_ is storing shards for _whom_? (That is,
     does it tell an observer who to apply the rubber-hose to in order to
     receive enough shards to recover the secret data?)

 - produce shards (up to step 8, above) which is some bytes _SB: there is no step 8 above_ (might be ref'ing the dark-crystal specs? I don't remember :/ )
 - put each shard into an **outbox** with intended recipient _SB: what's the outbox?_(a database, or a file, or ...)
     - e.g. save shard into a file corresponding to public-key of recipient
 - create a magic wormhole with any remaining intended recipient
    - either recipient or sender may create the code
    - the other one enters the code to complete the wormhole
 - once connected, confirm public-keys:
 - each side sends a **Challenge** message, containing the following:
    - timestamp
    - random nonce
 - each side responds with a **Response** message
    - Ed25519 signature of the received **Challenge** message
 - if the signature doesn't match, disconnect
 - select the shard from the **outbox** (based on the public-key of the other side)
    - if there is no shard for this recipient, send an Error message and disconnect
 - send the shard to the recipient (**Shard** message)
    - for large shards, a Transit connection should be opened (or Dilation)
 - receiving side sends **Received** message (_after_ storing the Shard)
 - sending side deletes shard file from **outbox**
 - sending side closes the wormhole

This protocol is repeated for each recipient.
Note that the sender has confirmation that the Shard is written to the recipients disk _before_ they delete the outbox copy.
Simiarly, the recipient receives confirmation that the sender has deleted their copy (by waiting for the wormhole to close).


### Example: OpenPGP

Many people already know an OpenPGP key for (some of) their colleagues.

An application could leverage this as **the PKI** to use for their secret-sharing needs.

An application with some secret data may leverage Magic Wormhole to share Dark Crystal shards.

That is, we have:

* some secret data to back up amongst several participants
* a list of Ed25519 public-keys of those participants
* a private (and public) Ed25519 key pair of the sender

To back up the secret data with the help of a number of peers, perform these steps:

1. Produce the Shards
2. Save the Shards in an outbox, keyed on recipient
3. Transmit Shards to Each Recipient
 
We review each of these steps in detail below.

1. Produce the Shards


     - Dark Crystal (can we use the public-keys _directly_ in the protocol? or will that leak stuff to passive observres, like **who to rubber-hose for the shards**?)


2. Put Shards in Outbox

    - Write Shard bytes to a file named `"<base64 pubkey>.shard"`
    - ...
    - (could be more vague here and "allow" use of a database)


3. Transmit Shards to Each Recipient


   - Sender and Receiver set up a magic wormhole:
       - either the Sender or the receiver creates a human-pronounceable code, and sends it to the other party
   - the other party enters the code
   - software should allow either way whenever possible
   - the code should be transmitted securely
    - A **wormhole** channel is now established.
    - If the secret data is large, Shards will be large -- in such a case, both sides should set up a direct Transit connection to the peer
        - XXX what is **large**?
    - For Shards below this threshold, the mailbox itself may be used
    - Both sides send a **Challenge** message
        - this consists of the 8 bytes of timestamp concatenated to the 32 bytes of nonce in the Challenge, then signed
    - Both sides send back a **ChallengeResponse** message
    - this is the Ed25519 signature of the received **Challenge** message
    - Each side confirms that the signature matches; if not, they send **Error** message and disconnect.
    - The **Sender** sends the Shard corresponding to the public key of the Recipient
  (If there is no matching Shard, send an **Error** message and disconnect)
    - Once the Shard is written to disk, the Recipient sends an **Acknowledge** message to the Sender
    - Once the Sender side receives an **Acknowledge** message, it deletes the Shard from the outbox
    - The Sender side closes the mailbox
    - The Receiver side sees the mailbox close
    - the protocol is complete

This protocol is repeated once for each Shard + Recipient pair in the outbox.
Note that if one of the Shard holders is the Sender themselves, this Shard should simply be stored and the below protocol not followed for that Shard.


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
