Anonymous Delivery of Secret Events
===================================

The following deals with a hypothetical system of _secret events_ on nostr. A
secret event has no publicly visible sender or recipient. Every secret event is
signed with a unique pubkey. Other than this, the only distinguishable metadata
is the `created_at` field and the size of the contents.

For anonymous delivery, there are some non-trivial problems that need to be
solved:

- Making sure the intended recipient sees the event
- Not revealing the intended recipient publicly
- Preventing relays from associating sender and recipient

Details on things like key management, encryption, authentication or message
structure are not covered here.


Solution 1: Send to All
-----------------------

A trivial solution is to have every recipient subscribe to every secret event.
Recipients simply try to decrypt each event received, discarding the ones that
fail to decrypt.

- The intended recipient sees the event
- The intended recipient is completely hidden from public
- Relays only learn who is sending and who is able to receive, not who is
  messaging whom

The obvious problem with this approach is that it does not scale well. Some
amount of proof of work on every secret event is necessary.

One could imagine having users subscribe to different prefixes of secret events,
in order to receive only a subset of them. This would reduce the anonymity set
by the same factor. Relays would be able to get a better idea of who is
messaging whom, especially for messages between users at different prefixes.


Solution 2: Rendezvous Beacons
------------------------------

A _rendezvous beacon_ is like a secret event, but with a pubkey that the
recipient uses to derive a shared secret with the sender. This shared secret is
used to derive a _rendezvous point_, which locates the associated secret event,
letting the recipient request the secret event directly. The contents of the
rendezvous beacon is a short hash derived from the shared secret, used by the
recipient to confirm that the rendezvous beacon was meant for them.

A rendezvous beacon is small and only used for the first contact between two
users. Users with prior public interactions do not need to use rendezvous
beacons, but can instead use implicit rendezvous points. As rendezvous beacons
are less frequent, a sender's tolerance for high proof of work might be higher,
allowing for a higher proof of work requirement and thus less chance of spam.
All this combined means that a system where everyone subscribes to all
rendezvous beacons may be a viable option.

The solution may also allow for users to individually select the prefix that a
rendezvous beacon id must match, letting them receive only a subset of all
rendezvous beacons. This may be more tolerable than in Solution 1, as it reveals
much less public information about messaging patterns. As recipients receive the
secret event from the rendezvous point directly, relays know the recipients of
secret events anyway.

A global prefix of "00000" (20 bits) would establish a baseline proof-of-work
requirement on rendezvous beacons in the largest anonymity set. Recipients
wanting to see fewer rendezvous beacons can choose one extra random prefix
digit, reducing the bandwidth as well as their anonymity set by a factor of 16,
and increasing the proof-of-work requirement on senders by the same factor.

The sender is free to post the rendezvous event and the actual event in either
order and at completely different times.

With this solution, if different prefixes are used, the recipient of the
rendezvous event is a little less anonymous, as only a subset of all recipients
have a filter that would match the rendezvous event. However, as long as the
filter is wide enough to match mostly false positives, this match is by itself
only a very weak signal. A rendezvous beacon also gives minimal information
about the associated secret event being received, assuming the events are posted
at different times.

Subsequent secret events are completely anonymous to the public.

The remaining problem is to make the sender and recipient anonymous to relays.
As recipients are filtering directly on events sent by senders, it should be
assumed that relays are able to link their identities together. Even though
temporary pubkeys are used, relays may have picked up the users' nostr identity
from other queries.

To give the recipient some plausible deniability, the recipient could subscribe
to only short pubkey prefixes of all secret events they are looking for,
possibly throwing in several fake prefixes as well. For one-off messages, this
could help give some protection in case the sender has failed to stay anonymous
to the relay.

For sender anonymity, two alternatives are discussed below.

### 2.1: Meticulous Use of Tor ###

If all clients used Tor and made sure to use a separate Tor circuit for every
secret event they subscribed to and every secret event they sent, relays
would learn when events are received, but not much more beyond what's public.
However, it's unlikely that this approach can be implemented by most clients.
Many relays also have issues with Tor connections.

### 2.2: Anonymous Messenger ###

An anonymous messenger acts as a _client_ listening to _ephemeral rendezvous
beacons_ on one or more relays. For every matching ephemeral rendezvous beacons,
it subscribes to the corresponding _ephemeral secret event_. This ephemeral
secret event contains an event to be posted to a given destination relay.

1. Messenger is anonymously connected to Relay 1, listening for ephemeral
   rendezvous beacons.
2. Sender constructs and posts a rendezvous beacon to Relay 1.
3. Messenger receives beacon, derives shared secret, subscribes to an ephemeral
   secret event with a pubkey derived from the shared secret.
4. Sender constructs the actual event to be posted to Relay 2.
5. Sender wraps the event in an ephemeral secret event for Messenger, with Relay
   2 as the destination and signed with the key derived from the shared secret.
6. Messenger receives and decodes the event.
7. After some delay, Messenger connects to Relay 2 anonymously and posts the event.

Unless _all 3_ of Relay 1, Messenger and Relay 2 collude, they are not able to
associate sender and recipient. If Relay 1 and Messenger collude, they may
identify the sender. If the event is a rendezvous event, they may get some
indication as to who the recipient may be. Therefore, given the choice between a
messenger that might only collude with Relay 1 and a messenger that might only
collude with Relay 2, the latter is the better option.

If Relay 1 and Relay 2 collude, they may try to associate the sender and
recipient by identifying messengers. Messenger should therefore connect
anonymously to the relays, such as by receiving and posting through individual
circuits on Tor.

A similar messenger solution may be devised for recipient anonymity.
