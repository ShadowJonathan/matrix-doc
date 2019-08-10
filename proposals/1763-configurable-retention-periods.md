# Proposal for specifying configurable per-room message retention periods.

A major shortcoming of Matrix has been the inability to specify how long
events should stored by the servers and clients which participate in a given
room.

This proposal aims to specify a simple yet flexible set of rules which allow
users, room admins and server admins to determine how long data should be
stored for a room, from the perspective of respecting the privacy requirements
of that room (which may range from a "burn after reading" ephemeral conversation,
through to FOIA-style public record keeping requirements).

As well as enforcing privacy requirements, these rules provide a way for server
administrators to better manage disk space (e.g. to enforce rules such as "don't
store remote events for public rooms for more than a month").

This proposal originally tried to also define semantics for per-message retention
as well as per-room; this has been split out into MSCxxxx in order to get the
easier per-room semantics landed.

## Problem:

Matrix is inherently a protocol for storing and synchronising conversation
history, and various parties may wish to control how long that history is stored
for.

 * Users may wish to specify a maximum age for their messages for privacy
   purposes, for instance:
   * to avoid their messages (or message metadata) being profiled by
     unscrupulous or compromised homeservers
   * to avoid their messages in public rooms staying indefinitely on the public
     record
   * because of legal/corporate requirements to store message history for a
     limited period of time
   * because of legal/corporate requirements to store messages forever
     (e.g. FOIA)
   * to provide "ephemeral messaging" semantics where messages are best-effort
     deleted after being read.
 * Room admins may wish to specify a retention policy for all messages in a
   room.
   * A room admin may wish to enforce a lower or upper bound on message
     retention on behalf of its users, overriding their preferences.
   * A bridged room should be able to enforce the data retention policies of the
     remote rooms.
 * Server admins may wish to specify a retention policy for their copy of given
   rooms, in order to manage disk space.

Additionally, we would like to provide this behaviour whilst also ensuring that
users generally see a consistent view of message history, without lots of gaps
and one-sided conversations where messages have been automatically removed.

At the least, it should be possible for people participating in a conversation
to know the expected lifetime of the other messages in the conversation **at
the time they are sent** in order to know how best to interact with them (i.e.
whether they are knowingly participating in a ephemeral conversation or not).

We would also like to set the expectation that rooms typically have a long
message retention - allowing those who wish to use Matrix to archive their
conversations to do so, and to allow Matrix to evolve as repository of
knowledge... unless participants explicitly request for a conversation history
to have limited lifetime.

This proposal does not try to solve the problems of:
 * GDPR erasure (as this involves retrospectively changing the lifetime of
   messages)
 * Bulk redaction (e.g. to remove all messages from an abusive user in a room,
   as again this is retrospectively changing message lifetime)
 * Specifying history retention based on the number of messages (as opposed to
   their age) in a room. This is descoped because it is effectively a disk space
   management problem for a given server or client, rather than a policy
   problem of the room. It can be solved as an implementation specific manner, or
   a new MSC can be proposed to standardise letting clients specify disk quotas
   per room.
 * Per-message retention (as having a mix of message lifetime within a room
   complicates implementation considerably - for instance, you cannot just
   purge arbitrary events from the DB without fracturing the DAG of the room,
   and so a different approach is required)

## Proposal

### Room Admin-specified per-room retention

We introduce a `m.room.retention` state event, which room admins can set to
mandate the history retention behaviour for a given room. It follows the
default PL semantics for a state event (requiring PL of 50 by default to be
set).

The following fields are defined in the `m.room.retention` contents:  

`max_lifetime`:
  the maximum duration in seconds for which a server must store
  this event.  Must be null or in range [0, 2<sup>31</sup>-1]. If absent, or null,
        should be interpreted as 'forever'.

`min_lifetime`:
  the minimum duration for which a server should store this event.
  Must be null or in range [0, 2<sup>31</sup>-1]. If absent, or null, should be
        interpreted as 'forever'.
  
`expire_on_clients`:
  a boolean for whether clients must expire messages clientside
  to match the min/max lifetime and/or self_destruct semantics fields. If absent,
  or null, should be interpreted as false.

Retention is only considered for non-state events.

If set, these fields SHOULD replace other retention behaviour configured by
the user or server admin - even if it means forcing laxer privacy requirements
on that user.  This is a conscious privacy tradeoff to allow admins to specify
explicit privacy requirements for a room.  For instance, a room may explicitly
require all messages in the room be stored forever with `min_lifetime: null`.

In the instance of `min_lifetime` or `max_lifetime` being overridden, the
invariant that `max_lifetime >= min_lifetime` must be maintained by clamping
max_lifetime to be equal to `min_lifetime`.

If the user's retention settings conflicts with those in the room, then the
user's clients MUST warn the user when participating in the room.  A conflict
exists if the user has configured their client to create rooms with retention
settings which differing from the values on the `m.room.retention` state
event.  This is particularly important to warn the user if the room's
retention is longer than their default requested retention period.

For instance:

```json
{
	"max_lifetime": 86400,
}
```

The above example means that servers receiving messages in this room should
store the event for only 86400 seconds (1 day), as measured from that
event's origin_server_ts, after which they MUST purge all references to that
event ID (e.g. from their db and any in-memory queues).

We consciously do not redact the event, as we are trying to eliminate metadata
and save disk space at the cost of deliberately discarding older messages from
the DAG.

```json
{
	"min_lifetime": 2419200,
}
```

The above example means that servers receiving this message SHOULD store the
event forever, but MAY choose to purge their copy after 28 days (or longer) in
order to reclaim diskspace.

### Server Admin-specified per-room retention

Server admins have two ways of influencing message retention on their server:

1) Specifying a default `m.room.retention` for rooms created on the server, as
defined as a per-server implementation configuration option which inserts the
state events after creating the room (effectively augmenting the presets used
when creating a room).  If a server admin is trying to conserve diskspace, they
may do so by specifying and enforcing a relatively low min_lifetime (e.g. 1
month), but not specify a max_lifetime, in the hope that other servers will
retain the data for longer.

  XXX: is this the correct approach to take? It's how we force E2E encryption
  on, but it feels very fragmentory to have magical presets which do different
  things depending on which server you're on.  The alternative would be some
  kind of federation-aware negotiation where a server refuses to participate in
  a room unless it gets its way on retention settings, however this feels
  unnecessarily draconian.

2) By adjusting how aggressively their server enforces the the `min_lifetime`
value for message retention within a room.  For instance, a server admin could
configure their server to attempt to automatically purge remote messages in
public rooms which are older than three months (unless min_lifetime for those
messages was set higher).

A possible implementation-specific server configuration here could be
something like:
 * target_lifetime_public_remote_events: 3 months
 * target_lifetime_public_local_events: null # forever
 * target_lifetime_private_remote_events: null # forever
 * target_lifetime_private_local_events: null # forever

...which would try to automatically purge remote events from public rooms after
3 months (assuming their individual min_lifetime is not higher), but leave
others alone.

These config values would interact with the min_lifetime and max_lifetime
values in the different classes of room by decreasing the effective
max_lifetime to the proposed value (whilst preserving the `max_lifetime >=
min_lifetime` invariant).  However, the precise behaviour would be up to the
server implementation.

Server admins could also override the requested retention limits (e.g. if
resource constrained), but this isn't recommended given it may result in
history being irrevocably lost against the senders' wishes.

## Pruning algorithm

To summarise, servers and clients must implement the pruning algorithm as
follows:

If we're a client, apply the algorithm if:
  * if specified, the `expire_on_clients` field in the `m.room.retention` event for the room is true.
  * otherwise, don't apply the algorithm.

The maximum lifetime of an event is calculated as:
  * if specified, the `max_lifetime` field in the `m.room.retention` event for the room.
  * otherwise, the message's maximum lifetime is considered 'forever'.

The minimum lifetime of an event is calculated as:
  * if specified, the `min_lifetime` field in the `m.room.retention` event for the room.
  * otherwise, the message's minimum lifetime is considered 'forever'.
  * for clients, `min_lifetime` should be considered to be 0 (as there is no
    requirement for clients to persist events).

If the calculated max_lifetime is less than the min_lifetime then the max_lifetime
is set to be equal to the min_lifetime.

The server/client then selects a lifetime of the event to lie between the
calculated values of minimum and maximum lifetime, based on their implementation
and configuration requirements.  The selected lifetime MUST not exceed the
calculated maximum lifetime. The selected lifetime SHOULD not be less than the
calculated minimum lifetime, but may be less in case of constrained resources,
in which case the server should prioritise retaining locally generated events
over remote generated events.

Server/clients then set a maintenance task to remove ("purge") old events and
references to their IDs from their DB and in-memory queues after the lifetime
has expired (starting timing from the absolute origin_server_ts on the event).

If possible, servers/clients should remove downstream notifications of a message
once it has expired (e.g. by cancelling push notifications).

If a user tries to re-backfill in history which has already been purged, it's
up to the server implementation's configuration on whether to allow it or not,
and if allowed, configure how long the backfill should persist before being
purged again.

Media uploads must also be expired in line with the retention policy of the
room. For unencrypted rooms this is easy; when the event that references a
piece of content is expired, the content must be expired too - assuming the
content was first uploaded in that room.  (This allows for content reuse in
retention-limited rooms for things like stickers).

For encrypted rooms, there is (currently) no alternative than have the client
manually delete media content from the server as it expires its own local
copies of messages.  (This requires us to have actually implemented a media
deletion API at last.)

## Tradeoffs

This proposal deliberately doesn't address GDPR erasure or mega-redaction scenarios,
as it attempts to build a coherent UX around the use case of users knowing their
privacy requirements *at the point they send messages*.  Meanwhile GDPR erasure is
handled elsewhere (and involves hiding rather than purging messages, in order to
avoid annhilating conversation history), and mega-redaction is yet to be defined.

It also doesn't solve specifying storage quotas per room (i.e. "store the last
500 messages in this room"), to avoid scope creep.  This can be handled by an
MSC for configuring resource quotas per room (or per user) in general.

It also doesn't solve per-message retention behaviour - this has been split out
into a seperate MSC.

## Issues

Should room retention be announced in a room per-server?  The advantage is full
flexibility in terms of servers announcing their different policies for a room
(and possibly letting users know how likely history is to be retained, or conversely
letting servers know if they need to step up to retain history).  The disadvantage
is that it could make for very complex UX for end-users: "Warning, some servers in
this room have overridden history retention to conflict with your preferences" etc.

## Security considerations

It's always a gentlemen's agreement for servers and clients alike to actually
uphold the requested retention behaviour; users should never rely on deletion
actually having happened.

## Conclusion

Previous attempts to solve this have got stuck by trying to combine together too many
disparate problems (e.g. reclaiming diskspace; aiding user data privacy; self-destructing
messages; mega-redaction; clearing history on specific devices; etc) - see
https://github.com/matrix-org/matrix-doc/issues/440 and https://github.com/matrix-org/matrix-doc/issues/447
for the history.

This proposal attempts to simplify things to strictly considering the question of
how long servers (and clients) should persist events for.