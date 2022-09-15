Magic Crystal
=============


| Author:                   | meejah (Least authority)
|:----------                |:-------------|
| Editor:                   | Sylvia Blaho (Least Authority)
| Contributors / Reviewers: | Chris Wood (Least Authority)|
|                           | Peg (Dark Crystal)|
|                           | [Mu (Dark Crystal)]|


In this document, we describe how to leverage [social secret sharing](https://darkcrystal.pw/what-is-secret-sharing/) with [Dark Crystal](https://darkcrystal.pw/) using [Magic Wormhole](https://magic-wormhole.readthedocs.io/en/latest/welcome.html) for identity-less transport _without_ using any Public Key Infrastructure.



# Table of contents
1. [Motivation](#motivation)
    - [Why Dark Crystal?](#why-DC)
    - [Why Magic Wormhole?](#why-MW)
2. [Protocol Design](#protocol-design)
3. [Application Specifications](#app-specs)
    - [Dark Crystal Threat Model](#dc-thm)
    - [Protocol Overview](#protocol-overview)
        - [Producing a Backup](#producing-backup)
        - [Recovering from an existing backup](#recovering-from-backup)
4. ["Out of Band" Authentication](#out-band-auth)
5. [Example with existing tools](#example-existing-tools)
6. [Discussion](#discussion)
    - [Secret size](#secret-size)
    - [Deleting Shards](#deleting-shards)



Motivation<a name="motivation"></a>
----------

Many applications have some important data,
such as private keys or cloud access credentials, that need to be kept secure and safe: such data should not be accessible to third parties, but it should be accessible to the user when needed, as it is **required** to recover access to the service in question.

[Dark Crystal](https://darkcrystal.pw/) provides a way to back up such data using social connections: the data is split into several pieces (called "Shards") and distributed among trusted peers (called "Custodians"). This is achieved using a scheme essentially based on [Shamir's Secret Sharing](http://web.mit.edu/6.857/OldStuff/Fall03/ref/Shamir-HowToShareASecret.pdf).

Dark Crystal is **transport agnostic**, meaning that in can be used in combination with any transfer/communication mechanisms.
In addition, using Dark Crystal requires a public-key infrastructure (PKI) of the users' choice.


The transfer mechanism used in this document is [Magic Wormhole](https://magic-wormhole.readthedocs.io/en/latest/welcome.html), a tool for connecting two computers on the Internet via human-transribed one-time codes.
Magic Wormhole relies on the assumption that the two humans in charge of the two computers can communicate with each other either in person, on the phone, or via some messaging service (that is, via some "reasonably secure" channel).

We describe a generic application that can backup and restore arbitrary secret data.
That said, it should have certain charateristics for this to be most useful.
The data should:

- be "one-time" (or rarely) produced;
- be "relatively small" (see [Discussion](#discussion));
- be self-contained;

Some examples of this:

- the seed / private key of a cryptocurrency wallet;
- credentials and access information for a cloud service (e.g. backup);
- confidential/sensitive information
- information identifying members of Human Rights Organizations in authoritarian countries

The main idea is that the user doing the backup has either lost access to their computer (e.g. it fell into the ocean), or the need to delete the - the data from their own hardware in cases when it could be seized by hostile parties (e.g. authoritarian governments, police raids, border crossing).

Now, given a new computer and a fresh instance of the application, they can contact some subset of the Custodians keeping Shards and thereby restore access.
Note that, in the interests of simplicity, we choose **not** to encrypt the secret data, so a sufficiently large subset of Custodians can conspire to restore access.
This feature can be useful for example in [estate planning](https://darkcrystal.pw/use-cases/#inheritance).
It also means there's no "passphrase" or similar for the user to remember; the **only** thing they need do is to re-contact some number of the Custodians.


### Why Dark Crystal?<a name="why-dc"></a>

Encrypting files and messages provides a high level of protection from data being compromised.
However, many people forego using encryption because they are worried that losing their encryption keys will result in them losing access to their data.

Dark Crystal aims to solve this problem by providing a set of protocols that enable users to split their data into Shards and then distribute these shards to a set of their trusted peers (Custodians).
It is not possible to reconstruct the original secret from a single Shard, so the secret remains protected if a Shard is compromised.
On the other hand, a subset of Shards is enough to recover the original secret, so the owner's access to the secret is preserved even if a shard is lost.

See the Dark Crystal documentation for more details about [the challenges of adopting encryption methods](https://darkcrystal.pw/why-dark-crystal/) and [potential use cases](https://darkcrystal.pw/use-cases/) for secret sharing with Dark Crystal.


### Why Magic Wormhole?<a name="why-mw"></a>

Dark Crystal does not specify a transport mechanism for distributing the shards between Custodians.
This means that Custodians either need to sign up for a specific service (like [Secure Scuttlebutt](https://scuttlebot.io/more/protocols/secure-scuttlebutt.html)) for securely distributing the shards, or resort to less secure methods such as email or instant messaging.
The latter channels also have the disadvantage that the data is stored on the service provider's servers, increasing the risk of it being compromised.

Magic Wormhole provides secure data transfer **directly between two computers**, without storing the secret on third-party servers.
Integrating Magic Wormhole and Dark Crystal provides a secure transport mechanism without the need for the Custodians to sign up for a secure transport service or use less secure channels.
While Magic Wormhole still relies on an external communication channel (e.g. email, instant messaging or voice) to relay the wormhole code, both the likelihood and the negative consequences of this code being compromised are minimal compared to the secret itself being shared on one of these channels.

A further advantage of Magic Wormhole is that it is **identity-less**, meaning that the participants do not need to supply personal information or obtain access credentials to complete the transfer.
Dark Crystal relies on the Owner and the Custodians using a PKI of their choice, which means that public/private keys are assigned to (partially) identified individuals.

Magic Wormhole relies on low-entropy, human-memorable codes to perform a PAKE (Password Authenticated Key Exchange).
These are made secure because only one guess may be used per code; if that one guess is an attacker, then the legitimate users know that code is destroyed.
A "brute force" attack thus involves convincing two humans to relay dozens or thousands of codes.
This isn't a full security argument; please see the Magic Wormhole documentation for more.

The basic premise of Dark Crystal is that the secret owner trusts their Custodians, presumably implying that they know at least some aspects of their identity, so being identifiable by the public/private keys might not be much of a disadvantage.
That said, one advantage of using Magic Wormhole is that the parties do not need to disclose any identifiable information to any other parties.

Perhaps more importantly, doing away with the need for a PKI could remove adoption barriers for less tech-savvy users, as it simplifies the list of technical requirements for Custodians.


Protocol design<a name="protocol-design"></a>
----------

In this document, we imagine and describe a generic stand-alone application that could, in principle, be used to socially back up any secret.
That said, our description should still be useful for application developers wanting to integrate this feature into their application.

Optionally, the Custodians can use a browser based client with no installation or configuration beforehand, making it a realistic option for non-technical users.
This overcomes many of the usability issues with peer-to-peer protocols.

The data to be backed up should be fairly "static", as the backup operation consumes human time, network bandwidth and storage resources.

While any amount of data could be shared in this fashion, we imagine such data is a "reasonably small" amount by modern standards.
(see [Discussion](#discussion)).


Application Specifications<a name="app-specs"></a>
--------------------------

### Dark Crystal Threat Model<a name="dc-thm"></a>

The Dark Crystal threat model includes some "Considerations for message transport".

Magic Wormhole provides robust encryption via SecretBox with an ephemeral secret key.

Indistinguishability of messages is a little more difficult to reason about.
By using the existing file-transfer protocol, Magic Wormhole sessions that transmit Shard data will look very similar to sessions that transmit other files.
It may still be possibly to distinguish based on file-size -- for example, if the secret data has a characteristic size.
Such a distinguisher will of course still have a margin of guessing: a file or text-message could also be the size of a Shard.
That said, since each Shard from one backup will be the **same** size, an attacker may be able to make a more accurate guess when several Shards are sent at nearly the same time – or from the same IP address (although Tor can be used to hide network location).

As mentioned in the Dark Crystal threat model, a mitigation is to send the Shards at different times.
This will **always** be the case in this transport, especially as the owner must establish a different out-of-band channel to each Custodian.
The software may help here by e.g. enforcing or suggesting a random time to wait before the next Shard is distributed.

The Dark Crystal document also suggests adding some padding.
We do not specify that currently, although it should be straightforward to add.


### Protocol Overview<a name="protocol-overview"></a>

By operating without identities, our protocol needs to only communicate a single chunk of data (the Shard) from the person doing the backup to the Custodian.
This mirrors the existing Magic Wormhole file-transfer application (which can do a single transfer in one direction).
One major benefits of re-using this protocol is it becomes hard(er) to distinguish a Magic Wormhole connection that is being used for social secret backup from any other file-transfer.
Another benefit is existing implementations (in several programming languages).

The social backup protocol is to produce some number of Shards and then distribute them.
To recover, the inverse is accomplished: a sufficient number of Shards are collected and used to reproduce the secret.


#### Producing a Backup<a name="producing-backup"></a>

The secret data may be encrypted (e.g. to a passphrase of the secret owner's choosing) or not.
Choosing to not encrypt the data has advantages:

- a sufficient number of Custodians can do disaster recovery;
- the secret owner has nothing additional to "back up" (e.g. remember a passphrase)
No matter the choice, our application will accept backup data as-is and anyone with a "threshold" number of Shards can reconstruct that data.

For every backup, we start a "backup session".
Every backup session has a "purpose", which is a freeform string provided by the user.
The human doing the backup should make the "purpose" meaningful as they will use it during recovery.
Backup sessions persist across many invocations of the GUI.

Once a "purpose" is set and persistent storage arranged the user gives us the "application data".
We do not encrypt it (it should be encrypted by the application or user first if they want that).

Next, we collect a list of "petnames" for the Custodians.
A "petname" is a local-only, user-defined string describing that Custodian.
They have no meaning to any other user and should be considered sensitive information.
Given the list of "petnames" we know how many Shards to produce.
We also record a "threshold" which is the number of Shards required to re-build the Application Data.
A session is completed when all Custodians have received their Shard.
(Another way to say that is that there are no more Shards left locally).

Shards are produced as per the [Dark Crystal specification](https://darkcrystal.pw/protocol-specification/).
In brief, this means:

- 256 Shards are produced
- each has a 1-byte ID
- each contains the symmetric-encrypted Application Data [*]
- each contains a Shamir Secret Sharing portion of the key for the above
- if we have fewer than 256 Custodians then, Shards are randomly selected.
Each Shard is stored locally and associated with one of the petnames for this session.

[*] Note that **this** symmetric-encryption is different from any application encryption choices and is part of the Shamir Secret Sharing method (which only encrypts up to 64 bytes in the Dark Crystal specification)

This completes the setup phase of the "backup session".
Next, the user is offered the opportunity to send any outstanding Shard to the corresponding Custodian.
This is accomplished by using Magic Wormhole.
Although a custom protocol could be defined, we instead use the existing file-transfer protocol.
Using this existing protocol makes it harder to distinguish a "Shard-sharing event" from any other file sharing.
The Magic Wormhole file-transfer protocol allows a single transfer in a single direction.

The next "distribution" phase is iterative: the user follows the procedure below for every petname in this session.
They could distribute all Shards during one invocation of the GUI, but it is more likely to take several invocations at different times.

It is important for the user to understand that it is up to them to establish the authenticity of the Custodian.
They must be sure that the wormhole code is given to the correct human and **only** to them.
Since computers are generally unreliable, it may sometimes take more than one attempt, but "lots" of attempts could indicate an attacker (see [Magic Wormhole](https://magic-wormhole.readthedocs.io/en/latest/welcome.html) documentation for more).

For every Custodian:

First, begin a magic-wormhole attempt – this generates a code (a small number followed by two words).
Establish an out-of-band connection to the human who will be the Custodian for this petname.
This channel might be a phone-call, in-person meeting, Signal message, etc
Give the other human the wormhole code, which should establish the connection.
We establish this magic-wormhole file-transfer session in "send" mode.

The Shard for this Custodian is offered as filename ``<purpose>.shard`` where "purpose" is the freeform string stored with this backup session.

The Custodian accepts the Shard and stores it locally under an associated petname (this is a **different** pentame from the sender, as it is only seen by the Custodian).
Once the Custodian accepts the file and it is successfully transferred, the local copy of the Shard is deleted.
This marks that petname as complete.
If the transfer fails for any reason, the local Shard copy is retained.

The above process repeats for each Custodian.
If this is the last local Shard, then the backup session is deemed as completed.


#### Recovering From an Existing Backup<a name="recovering-from-backup"></a>

For every recovery, we start a "recovery session".
Every recovery session has a "purpose", which is a freeform string provided by the user – it should match the purpose from a previous "backup session".
(Although the software should remember this for its users when possible, it's also possible the user is starting from nothing on a brand-new computers so freeform input should be allowed).
Similarly to backups, recovery sessions may persist longer than one GUI invocation.
A list of "petnames" of expected Custodians is provided.
If it is not already known, the human must specify the "threshold" which is the number of Shards required to recover the Application Data.

This sets up the recovery session and we now enter an iterative "collection" phase.
We remain in this phase until a "threshold" number of Shards are returned.

One round of the collection phase consists of:

Select a petname corresponding to an uncollected Shard.
Establish a magic-wormhole file-transfer connection to this shard in "receive" mode (see the "backup" section for more details).
On the Custodian side – which is the "send" side – the user must  the "purpose"; this should be prompted to them via the same out-of-band channel used to communicate the wormhole code.
The Custodian also selects the correct petname for this user (in case, for example, they have multiple different Shards for one "purpose").
If the Custodian has the correct Shard (i.e. they previously received one for this "purpose" and saved it under the same petname selected this time) they offer as ``<purpose>.shard`` in the file-transfer.
Once the transfer is completed successfully the Custodian **could** delete the Shard (we leave this as an upon UX question).

On the "recovery" side, this Shard is recorded alongside the local petname.

Once a "threshold" number of Shards are collected, the collection phase ends.

After the collection phase, we have enough Shards so they can be combined and decrypted.
This yields the Application Data (if it had been encrypted beforehand, it would be ciphertext still here).
The recovery session is now completed.
Again, we leave it as an open UX question whether to delete the Shards or not at this point.


### "Out of Band" Authentication<a name="out-band-auth"></a>

We depend on "out of band" authentication.
This means, for example, being confident you're speaking over the phone to the right human.
It could mean re-using an existing secure channel like a Signal chat.
You may be meeting in person and speaking to each other in real time.

In any case, much of the protocol depends on the correct identification of the other human.
This replaces the functionality that Public Key Infrastructure often provides.

It also opens up a prospective secret-holder to social attacks.
For instance, if an attacker might successfully impersonates a "threshold" number of friends of a secret-holder, then they can recover that secret.
Similarly, if an attacker can successfully convince enough Custodians that they are the secret holder, then they again may recover the secret.

Users must use caution when sending out Shards: the software must explain the importance of this action, in particular, the importance of being sure of the recipient of it.
That is, when the Magic Wormhole is being set up, users must take the time to correctly identify whom they are giving the code to.
One code should only be given to one other person.
Users should also be warned that each re-try is another opportunity for an attacker to guess a code (that is, having to retry many times in a row is unlikely and may indicate an attack).


Example with Existing Tools<a name="example-existing-tools"></a>
---------------------------

The Dark Crystal team have produced a proof-of-concept tool in Rust to produce Shard data. See https://gitlab.com/dark-crystal-rust/dark-crystal-wormhole
There are existing Magic Wormhole file-transfer tools including the reference Python command-line tool (see [https://github.com/magic-wormhole/magic-wormhole](https://github.com/magic-wormhole/magic-wormhole)).

The above protocol can be approximated with these tools by using some conventions.

The state of all backups is represented by a local directory (for example, ``~/magic-crystal-backups``).
A "backup session" is represented as a directory inside that named after the "purpose" for the session.
For example, ``~/magic-crystal-backups/privatestorage-recovery-key``.

The ``dark-crystal-wormhole`` tool is then used inside this directory to make all the Shards required.
For example, to create five Shards where any three can re-construct the secret: ``cargo run -- share --secret "password-manager root password" --n 5 --threshold 3``
To enumberate the petnames, each Shard file is re-named to ``<petname>.shard``.

The "distribute" phase is then entered.
Each iteration involves using ``wormhole send --file <petname>.shard`` for each remaining petname.
Every successfully-transferred Shard file is then deleted.


A recovery session is similarly represented as a local directory named after the "purpose".

For each Custodian, establish a "receive" file-transfer session: ``wormhole receive --code-words 2`` (note, this won't work until https://github.com/magic-wormhole/magic-wormhole/issues/450 is fixed).
In the out-of-band session used to communicate the code-word, also communicate the "purpose".
The Custodian then visits the right directly, and determines if there is a file for this human (i.e. their petname).
If so, they send it ``wormhole send --code <whatever the receiver said> --file <petname>.shard``
(Note: this leaks their local petname; they should first rename to ``<purpose>.shard``)
The receiver re-names whatever they got to ``<petname>.shard` with their petname for the Custodian.

Once there are enough shards, they can be decrypted with the Dark Crystal tool ``cargo run -- combine``.


Discussion<a name="discussion"></a>
---------------------------

### Secret size<a name="secret-size"></a>

We do not define exactly what a "reasonably sized" secret is.
Instead, we dicuss some considerations to bear in mind when designing secret data to back up.
The size of the secret is X bytes; the number of Custodians is N.

- final secrets include some fixed-sized additional data, but X itself dominates after "hundreds of bytes"
- how much bandwidth do we have? (we will use at least N * X of bandwidth)
- how much storage do we have? (a Custodian uses X storage, for each backup)
- the transfer is "on-line": both parties (the sender and the Custodian) must be online while the transfer completes
- the Magic Wormhole "mailbox server" may have limits
  - could use "Transit Relay" / bulk-transfer mode then
  - above mode should be used for more than "dozens of kilobytes"


### Deleting Shards<a name="deleting-shards"></a>

There are two places we leave an "open UX question" about whether to delete Shards.

One of these is on the "recovery" side and one is on the Custodian side.

It may make sense to ask the user.

They have done the hard work of completing a transfer which might indicate keeping Shards.
Shards also represent a risk: **anyone** who collects enough of them can re-construct the Application Data.
So, both sides can have an interest in deleting the Shard data.

For example, a Custodian may simply ask the backup side which is best (delete, or continue to retain the Shard).

On the recovery side, keeping the Shards around can facilitate re-extracting the data.
This could also be a risk, though, so again asking the human may be most appropriate.
In this case, the Custodian shouldn't have an interest so it's at least just a local decision.
