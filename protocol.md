Magic Crystal
=============


<!--Not really a specification, can give recommendations, but not capitalized specification-y things, more prose style. Primary objective: someone building an app with these.-->

| Author:       | meejah (Least authority)
|:----------    |:-------------|
| Editor:       | Sylvia Blaho (Least Authority)
| Contributors/ | Chris Wood (Least Authority)|
| Reviewers:    | Peg (Dark Crystal)|
|               | [Mu (Dark Crystal)] |



<!--XXX meejah: I capitalized some terms as a way to emphasize them as jargon (Shard, Custodian) but maybe that could be done a different way.-->

<!--XXX meejah: I'm trying to use "semantic newlines", which means a newline only after every sentence (sometimes sooner) instead of e.g. "flowing" paragraphs.
Part of the reasoning here is diffs, I think? (If you don't like it, reformat ;)
-->


In this document, we describe how to leverage social [secret sharing](https://darkcrystal.pw/what-is-secret-sharing/) with [Dark Crystal](https://darkcrystal.pw/) using [Magic Wormhole](https://magic-wormhole.readthedocs.io/en/latest/welcome.html) for identity-less transport _without_ using any Public Key Infrastructure.


_SB: Add TOC when doc is ready_


Motivation
----------

Many applications have some important data, 
such as private keys or cloud access credentials, that need to be kept secure and safe: such data should not be accessible to third parties, but it should be accessible to the user when needed, as it is **required** to recover access to the service in question. 

[Dark Crystal](https://darkcrystal.pw/) provides a way to back up such data using social connections: the data is split into several pieces (called "Shards") and distributed among trusted peers (called "Custodians"). 

Dark Crystal is **transport agnostic**, meaning that in can be used in combination with any transfer/communication mechanisms.
In addition, using Dark Crystal requires a public-key infrastructure (PKI) of the users' choice.


The transfer mechanism used in this document is [Magic Wormhole](https://magic-wormhole.readthedocs.io/en/latest/welcome.html), a tool for connecting two computers on the Internet via human-transribed one-time codes.
Magic Wormhole relies on the assumption that the two humans in charge of the two computers can communicate with each other either in person, on the phone, or via some messaging service (that is, via some "reasonably secure" channel).

We describe a generic application that can backup and restore arbitrary secret data.
That said, it should have certain charateristics for this to be most useful.
The data should:

- be "one-time" (or rarely) produced;
- be "relatively small" (see discusion XXX);
- be self-contained;

Some examples of this:

- the seed / private key of a cryptocurrency wallet;
- credentials and access information for a cloud service (e.g. backup);
- ...

The main idea is that the user doing the backup has lost acces to their computer (e.g. it fell into the ocean).
Now, given a new computer and a fresh instance of the application, they can contact some subset of the Custodians keeping Shards and restore access.
Note that in the interests of simplicity we choose _not_ to encrypt the secret data, so a sufficiently-large subset of Custodians can conspire to restore access.
This feature can be useful for example in estate planning.
It also means there's no "passphrase" or so to remember for the user; they really do _just_ need to re-contact some number of the Custodians.


### Why Dark Crystal?

Encrypting files and messages provides a high level of protection from data being compromised.
However, many people forego using encryption because they are worried that losing their encryption keys will result in them losing access to their data.

Dark Crystal aims to solve this problem by providing a set of protocols that enable users to split their data into Shards and then distribute these shards to a set of their trusted peers (Custodians).
It is not possible to reconstruct the original secret from a single Shard, so the secret remains protected if a Shard is compromised.
On the other hand, a subset of Shards is enough to recover the original secret, so the owner's access to the secret is preserved even if a shard is lost.

See the Dark Crystal documentation for more details about [the challenges of adopting encryption methods](https://darkcrystal.pw/why-dark-crystal/) and [potential use cases](https://darkcrystal.pw/use-cases/) for secret sharing with Dark Crystal.


### Why Magic Wormhole?

Dark Crystal does not specify a transport mechanism for distributing the shards between custodians.
This means that custodians either need to sign up for a specific service (like [Secure Scuttlebutt](https://scuttlebot.io/more/protocols/secure-scuttlebutt.html)) for securely distributing the shards, or resort to less secure methods such as email or instant messaging.
The latter channels also have the disadvantage that the data is stored on the service provider's servers, increasing the risk of it being compromised.

Magic Wormhole provides secure file transfer **directly between two computers**, without storing the secret on third-party servers.
Integrating Magic Wormhole and Dark Crystal provides a secure transport mechanism without the need for the custodians to sign up for a secure transport service or using less secure channels.
While Magic Wormhole still relies on an external communication channel (e.g. email, instant messaging or voice) to relay the wormhole code, both the likelihood and the negative consequences of this code being compromised are minimal compared to the secret itself being shared on one of these channels.

A further advantage of Magic Wormhole is that it is **identity-less**, meaning that the participants do not need to supply personal information or obtain access credentials to complete the transfer.
Dark Crystal relies on the Owner and the Custodians using a PKI of their choice, which means that public/private keys are assigned to (partially) identified individuals.

The basic premise of Dark Crystal is that the secret owner trusts their custodians, presumably implying that they know at least some aspects of their identity, so being identifiable by the public/private keys might not be much of a disadvantage.
That said, one advantage of using Magic Wormhole is that the parties do not need to disclose any identifiable information to any other parties.

Perhaps more importantly, doing away with the need for a PKI could remove adoption barriers for less tech-savvy users, as it simplifies the list of technical requirements for Custodians.

Protocol design
----------


In this document, we imagine and describe a generic stand-alone application that could, in principle, be used to socially back up any secret.
That said, our description should still be useful for application developers wanting to integrate this feature into their application.

Optionally, the Custodians can use a browser based client with no installation or configuration beforehand, making it a realistic option for non-technical users and overcoming many of the usability issues with peer-to-peer protocols.

The data to be backed up should be fairly "static", as the backup operation consumes human time, network bandwidth and storage resources.

While any amount of data could be shared in this fashion, we imagine such data is a "reasonably small" amount by modern standards.
(see [Discussion](#discussion)).


Application Specifications
--------------------------

### Threat Model notes

- ref'ing Dark Crystal "Threat Model", "Considerations for message transport"

- robust encryption
   - evesdroppers see SecretBox encrypted (over tls, probably) messages
   - transit relay (if used) sees IP addresses, can know they're communicating
     (can use Tor to mitigate)
- indistinguisable
   - all wormhole connections look _similar_ (two IPs, maybe-relay, SecretBox encryption)
   - but: fingerprinting; probably can distinguish "file-transfer protocol" from other application-level protocols
   - meejah: aside: good argument to "just use" the file-transfer protocol?


### Protocol Overview

_SB: The **Protocol Overview** and the **Example** sections are very similar right now. Are we going to add more concrete details to the **Example** section later? If not, I would merge these two section into one, because it feel very repetitive as it is now._

meejah: probably being more concrete in the example would be good

#### Producing a Backup

The secret data may be encrypted (e.g. to a passphrase of the secret-owner's choosing) or not.
Choosing to not encrypt the data has advantages:
- a sufficient number of Custodians can do disaster recovery;
- the secret owner has nothing additional to "back up" (e.g. remember a passphrase)
No matter the choice, our application will accept backup data as-is and anyone with a "threshold" number of Shards can reconstruct that data.

- start a backup session:
    - set "purpose" of the session
    - (list petnames of Custodians?)
- Application Data is produced and handed to our application as bytes (e.g. in a file, stdin, API, ...)
- our application produces Shards, using the "random" method, as per the Dark Crystal specification
- This means:
    - 256 Shards are produced
    - each has a 1-byte ID
    - each contains the symmetric-encrypted Application Data
    - each contains a Shamir Secret Sharing portion of the key for the above
    - if less than 255 Custodians are seleced, we select a random number of these Shards
- each Shard is stored in a file, named like: `.../<purpose>/<petname>.shard`
- the "petname" is the user-supplied local-only nickname for the intended Custodian
- the "purpose" is an application-specific string (e.g. "secure-scuttlebutt")
- for each Custodian who has not yet received their Shard:
    - create a magic-wormhole with them
        - out-of-band user authentication (e.g. phone-call, Signal thread, ...)
    - send the Shard file data, named like `<purpose>.shard`
    - the Custodian saves the file as `.../<petname>/<purpose>.shard` where `<petname>` is their local-only nickname for the sender of the Shard
    - once the recipient acknowledges receipt -- which they should only do one the file is written -- the sender deletes their corresponding Shard file

This protocol is repeated for each recipient.
Note that the sender has confirmation that the Shard is written to the recipients disk _before_ they delete the outbox copy.
Simiarly, the recipient receives confirmation that the sender has deleted their copy (by waiting for the wormhole to close).


XXX meejah: didn't include identity or challenge/response above ... I _think_ that means we can just use normal file-transfer ... or we could put challenge/response in (but optional?) ... I like the idea of identity-less though


#### Recovering Using Existing Backup

- start a recovery session
- set the "purpose" for the recovery (arbitrary string)
- (list the petnames of all known Custodians?)
- until we have a "threshold" number of Shards:
    - create a wormhole to a Custodian
        - out-of-band authentication of the correct human
    - tell them the "purpose"
        - this could be a protocol message .. but "file transfer" doesn't have this
        - it could instead be "out of band" (e.g. human tells other human the purpose)
        - means: "Custodan end" of the wormhole sends the right file (or terminates)
    - if the Custodian has a local Shard matcing the petname and purpose of this human, send it back (named like `<purpose>.shard`).
    - otherwise, terminate the protocol
    - save the shard as `..../<purpose>/<petname>.shard` where `petname` is the local-only nickname for this human
- once we have at least a "threshold" number of Shards, decrypt them
- the original Application Data is now recovered (and can be passed back, e.g. via file, stdout, ...)

Discussion
----------

XXX meejah: considerations to put in ^ discussion (elsewhere, probably?):

- consider a secret of size X.
- consider N Custodians
- how much bandwidth do we have? (we will use at least N * X of bandwidth)
- how much storage do we have? (a Custodian uses X storage, for each backup)




XXXX
----------

XXX meejah: Leaving some things around for now, in case we decide to go "custom protocol" and have challenge/response


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
