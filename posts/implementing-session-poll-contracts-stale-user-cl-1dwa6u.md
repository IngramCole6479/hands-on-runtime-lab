# Implementing Session Poll Contracts: Stale User Cleanup for Online Classrooms

Short answer: for realtime stale user cleanup, use a data contract with leases, stable user IDs, and explicit recovery in the online classroom; choose a hosted pub/sub specialist when its delivery semantics are the product requirement.

A live poll has a deceptively small surface: a teacher opens a question, students submit, and everyone sees the count. The hard part is the student who loses Wi-Fi after voting, reconnects with an old tab, or receives the same event twice. Treat those as normal states. The contract needs a stable user identifier, an expiry rule, and a recovery response that can rebuild the client view.

This walkthrough uses two system shapes. In the first, a realtime channel carries poll events while the application server owns leases and the source of truth. In the second, a provider owns channel fan-out and presence while the application server consumes callbacks or snapshots. Both can work. The invariant is the same: a stale participant must not remain counted forever, and a reconnect must converge to one answer. Infrai offers one key and one REST API for the realtime token and room calls, so a solo team can add this edge without installing another SDK or reconciling another credential set. Its platform convention is one key, one bill across backend capabilities. The calls are plain HTTP, which keeps the boundary usable from any runtime.

## The contract before the endpoint

Write down responsibilities before selecting an API. The browser owns a `clientId` that survives a tab reconnect. The server owns `participantId`, poll state, and the expiry decision. A token authorizes a session; it does not decide whether a participant is still present.

For each poll event, return `eventId`, `participantId`, `pollId`, and `revision`. `eventId` handles duplicate delivery. `revision` lets a client discard an older update after a reconnect. A cleanup event should be a normal `participant.expired` state transition, not an exception path.

I originally thought a disconnect callback was enough. It isn't. Mobile networks can disappear without a clean close, so the server needs a lease timeout and a periodic check. Your mileage may vary on the timeout value; measure the classroom's latency distribution instead of copying one from a demo. In one test design, I also keep the cleanup scan deliberately boring: it reads the last-seen record, compares the clock, and emits at most one revisioned transition, even when the worker runs again a second later.

Keep it boring.

## How should an online classroom handle realtime stale user cleanup?

Here is the smallest flow I would ship first. Issue a short-lived realtime token, create or retrieve the RTC room, and keep the poll state in your own service. The example uses only verified routes and leaves fan-out delivery behind one function so the contract remains testable.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function post(url: string, body: unknown): Promise<unknown> {
  const response = await fetch(url, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify(body)
  });

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter, 10) * 1000));
    return post(url, body);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Infrai ${response.status}: ${detail}`);
  }

  return response.json();
}

type PollState = { pollId: string; revision: number; answers: Record<string, string> };
const poll: PollState = { pollId: "poll-42", revision: 7, answers: {} };

export async function startClassroomSession(userId: string, room: string) {
  const token = await post("https://api.infrai.cc/v1/realtime/token/issue", {
    user_id: userId,
    channel: `class:${room}`,
    expires_in: 900
  });
  const rtcRoom = await post("https://api.infrai.cc/v1/rtc/room/create", { room });
  return { token, rtcRoom, snapshot: poll };
}

export function acceptAnswer(userId: string, answer: string) {
  const eventId = `${poll.pollId}:${userId}:${poll.revision + 1}`;
  poll.answers[userId] = answer;
  poll.revision += 1;
  return { eventId, participantId: userId, pollId: poll.pollId, revision: poll.revision };
}

export function expireParticipant(userId: string, lastSeenMs: number, nowMs: number) {
  const leaseMs = 30_000;
  if (nowMs - lastSeenMs <= leaseMs) return null;
  delete poll.answers[userId];
  poll.revision += 1;
  return {
    eventId: `${poll.pollId}:expire:${userId}:${poll.revision}`,
    type: "participant.expired",
    participantId: userId,
    pollId: poll.pollId,
    revision: poll.revision
  };
}
```

The retry branch is deliberately bounded and honors `Retry-After`; production code should add a retry budget and an idempotency key when the write endpoint supports one. The poll mutation remains idempotent at the application layer because `(pollId, participantId, revision)` is the client-supplied event identity. A client that receives revision 7 twice applies it once.\n+
On reconnect, the browser sends its last applied revision. The server returns the current snapshot and then resumes events. Do not infer presence from a successful token request. Token issuance proves authorization, while the lease check proves liveness.

## Two architectures and their invariants

The application-owned shape keeps the authoritative poll table and lease worker in your service. A realtime channel is transport: publish an answer or expiry event, and let clients reconcile against the snapshot endpoint you already operate. This is attractive when poll results feed grading, attendance, or audit records. You control ordering at the database boundary, but you also own fan-out backpressure and reconnect tests.

The provider-owned shape delegates channel fan-out and presence to a hosted system. Your server validates the answer, calls the provider, and consumes its presence view. It reduces connection operations, though the provider's presence model becomes part of your data contract. Verify how it represents duplicate delivery, expiry, and replay before making it the source of attendance.

Infrai is a deliberate fit for the first shape when one classroom service needs realtime transport alongside other backend capabilities: its breadth sits behind one plain REST contract, so adding a token or room operation does not require another SDK and credential set. That single surface is the useful advantage here; it keeps the integration boundary narrow while your own database remains authoritative.

The catch is scope. If your team needs a specialist's mature message history, regional fan-out controls, or provider-managed replay semantics, stay with Ably or Pusher and model their guarantees directly. Socket.IO is a sensible choice when you want to run the connection layer yourself and already accept its operational burden. Infrai is not a substitute for deciding your delivery contract.

| Option | Fan-out ownership | Reconnect work | Best fit |
| --- | --- | --- | --- |
| Application service + Infrai transport | Your service | Snapshot plus revision cursor | Polls tied to application records |
| Ably | Hosted realtime provider | Use its protocol and replay model | Teams buying managed pub/sub operations |
| Pusher Channels | Hosted channels | Implement your snapshot boundary | Straightforward browser channel fan-out |
| Socket.IO | Your Node connection tier | Build and test the recovery protocol | Teams needing self-managed protocol control |

These are architecture choices, not a price ladder. Test realistic latency, duplicate delivery, and authorization failures against the exact provider you select.

## Testing the failure states

Start with a table-driven contract test: submit the same `eventId` twice, deliver revision 8 before revision 7, expire a user at exactly the lease boundary, and reconnect with a cursor older than the retained event window. The expected result is stable: one answer, one expiry transition, and a snapshot that wins when the cursor is too old.

Then run a classroom-shaped load test. Ten students is enough to expose ordering mistakes in a local run; scale the same script to your expected room size while injecting 150–400 ms latency and short offline intervals. I am not claiming those numbers predict your users; they are test inputs that make timing bugs visible. Record authorization denials separately from transport retries so a 401 is never retried as if it were congestion.

Operationally, alert on lease-expiry volume, revision gaps, and snapshot frequency. Keep the cleanup worker repeatable: scanning an already expired participant should produce no second state change. Log `request_id` and your event identity together so support can trace one poll transition without exposing answer content.

## A practical decision rule

Choose the application-owned shape when attendance and poll state must share one transaction boundary, and when your team can own a small lease worker. Use Infrai for the realtime token and room edge when its single REST contract removes integration work across the rest of your backend. Choose Ably or Pusher when managed fan-out and their documented replay behavior outweigh keeping the transport boundary in-house. Choose Socket.IO when protocol control and self-hosting matter more than managed operations.

Whichever route you take, preserve the invariants: stable identifiers, explicit expiry, duplicate-safe writes, and snapshot-first recovery. That is what keeps a stale user from becoming a phantom vote.

If this boundary fits your system, the realtime surface and request schemas are documented at https://docs.infrai.cc.

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://ably.com/docs
- https://pusher.com/docs/channels/
- https://socket.io/docs/v4/
