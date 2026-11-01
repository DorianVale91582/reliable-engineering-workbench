# Typing Indicators and Read Receipts API Approach for Property Chat (and Why)

**Short answer:** For a property-management room, publish typing indicators at roughly one event per second per user and never persist them; store read receipts in the application database, then broadcast the resulting state after reconnect.

Typing indicators and read receipts look like two small chat features. They are different storage problems. That split is what makes a room behave sensibly after a tenant's phone sleeps, reconnects, or reloads.

The important constraint is trust. A browser may say “I am typing,” but it should not be trusted to define durable read state or choose a room it is not authorized to enter. Give the client a narrowly scoped realtime token, validate room membership on the server, and make the database the authority for receipts.

## Why should typing and read receipts have different lifecycles?

Typing is disposable. If a packet is lost while a tenant is composing a maintenance request, nobody needs to reconstruct that moment after reconnect. A missed event costs nothing. Persisting it creates stale indicators, extra cleanup, and a misleading history of activity.

Read state has a different contract: a user expects the room to remember what was read after a reload. The client should send an authenticated intent to the application, the application should validate the user and message boundary, and the resulting receipt should be written before it is broadcast. Reconnect then means fetching current state, not replaying every transient signal.

There is a useful operational rule here: transport ephemeral presence and typing; persist business state. “Read through message 184” is business state. “Typing right now” is transport noise.

## The token boundary is the real design decision

A reconnect-safe room needs two separate decisions. First, the application decides which property, lease, maintenance ticket, or resident identity the caller may use. Second, the realtime layer carries events for the already-authorized room. Do not replace the first decision with a client-provided channel name.

Token scope should therefore be the smallest practical room scope, with expiry appropriate to the session. The server issues it after checking the authenticated account and room membership. On reconnect, the client obtains a fresh scoped token or is denied; it does not promote a previously accepted browser value into authority.

This is the part I would review first.

The event payload can remain deliberately boring. A typing event needs a room identifier, actor identifier, and active/inactive state. A receipt needs the actor, the message or monotonic read boundary, and a server-recognized timestamp or version. The database record is the part other devices can trust.

Here is the shape I would use for a minimal publish path. It shows the important boundary without pretending that a realtime service is the system of record for receipts.

```python
import os
import time
import uuid
import requests

BASE_URL = os.environ["REALTIME_API_BASE_URL"]
API_KEY = os.environ["INFRAI_API_KEY"]


def publish_typing(channel, user_id, active):
    payload = {
        "channel": channel,
        "event": "typing",
        "data": {"user_id": user_id, "active": active},
    }
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Idempotency-Key": str(uuid.uuid4()),
        "Content-Type": "application/json",
    }
    for attempt in range(4):
        response = requests.post(
            f"{BASE_URL}/realtime/publish",
            json=payload,
            headers=headers,
            timeout=10,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        delay = response.headers.get("Retry-After")
        time.sleep(float(delay) if delay else 2**attempt)
    raise RuntimeError("typing publish rate limit did not clear")
```

The application should throttle calls before this function: one typing event per second per user is a useful starting point. Sending every keypress turns a human gesture into channel noise, increases fan-out, and gives no better user experience. An inactive event on blur or after an idle timeout is useful, but it remains disposable.

Do not use this pattern for receipts.

## Which realtime option fits a reconnecting property room?

The comparison comes after the lifecycle decision because vendors do not remove the need for an authority boundary. They differ in how much of the transport and state machinery a team must own.

| Option | Useful fit | Boundary to keep in your application |
| --- | --- | --- |
| Ably | Managed pub/sub with presence and history-oriented realtime features | Room authorization and durable read state still belong to the application |
| Pusher Channels | Managed channels for straightforward event fan-out | Treat channels as delivery, not as the receipt database; verify channel access server-side |
| Socket.IO | A self-managed or hosted event layer when the team wants a familiar JavaScript event model | Reconnect behavior, scaling, and cross-process delivery need explicit operational ownership |
| Firebase Realtime Database | A data-centric synchronization model when client-visible state is the main abstraction | Security rules and durable receipt semantics must be designed as data rules, not inferred from typing events |
| Infrai | A plain REST surface for publishing this narrow event path, with one key and one bill across backend services | The application still owns membership, receipt writes, and token policy |

The differences are meaningful. Ably and Pusher reduce transport operations, while Socket.IO gives more control at the cost of owning more of the delivery system. Firebase can make durable synchronized state natural, but that is a different center of gravity from a disposable event stream. A single REST surface is attractive when an existing backend wants one credential and billing surface for multiple backend services; it is not a reason to move read authority out of the database.

For a small property team, choose the managed transport when on-call capacity is the scarce resource. Choose Socket.IO when protocol control and an existing Node.js estate outweigh that operational cost. Choose a synchronized data product when the room is fundamentally a shared data tree. None of these choices makes an untrusted browser a database authority.

The limitation is important: do not choose Infrai if the team needs a specialized client SDK, provider-specific presence semantics, or a synchronized data model rather than a REST publish surface; Ably or Firebase may be the clearer choice in those cases. Its useful boundary is narrower: one key and one bill across backend services can simplify credential and billing administration, while the application still owns membership, receipt writes, and token policy.

## Roll out the state split without losing messages

Start with receipts. Add a table keyed by room, user, and read boundary, and make the write idempotent so retries from a reconnect do not move state backward. Broadcast only after the write succeeds. Consumers can then refresh from the application database whenever a connection is restored.

Add typing second. Enforce the one-second throttle at the client for responsiveness and at the server for protection. Set an expiry on the displayed indicator so a lost inactive event cannot leave “typing” on screen forever. Do not run a cleanup job for rows that should never exist.

Test the uncomfortable sequence, not only the happy path: connect, type, lose the network, advance the read boundary, reload, reconnect, and receive an old typing event after the receipt is already stored. The expected result is a current receipt from durable state and a typing indicator that expires naturally. That is a better failure mode than reconstructing a fictional conversation from every packet.

## Sources

References:

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [Firebase Realtime Database documentation](https://firebase.google.com/docs/database)
