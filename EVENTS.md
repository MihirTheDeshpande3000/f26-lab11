# EVENTS.md

One section per milestone. Fill each one in order as you go.

The prediction in milestone 2 should be written before you run the replay. That is the
part a TA will ask about.

Keep it short and specific. Point at operations, call sites, and what a caller
gets back.

---

## Milestone 1: what runs in the request path, and what runs after it

### The operations table

One row per operation. Read the code, not the file
names. "As wired today" is what the code does now. "Your call" is what you
would do if this were your service. List up to eight operations, and remove
the rows you don't use.

| # | Operation | As wired today | Your call | Why (in one line) |
|---|---|---|---|---|
| 1 | Index a confirmed booking for search | Direct synchronous call from `BookingService.book`, before the store write | Subscribe an index handler to `booking.confirmed` | Search is a secondary view, so its outage must not prevent the reservation. |
| 2 | Save the confirmed booking | Direct synchronous call to `store.save` in `book` | Keep it direct and synchronous | The caller needs the room to be held before receiving success. |
| 3 | Record booking creation in the audit trail | Direct synchronous call in `book` | Keep it direct and synchronous | A successful booking must not exist without its audit record. |
| 4 | Queue the confirmation email | `EmailConfirmationHandler` subscribes to `booking.confirmed` | Keep it on the bus | Email can follow the committed booking and should not decide whether the room is held. |
| 5 | Generate the receipt | `ReceiptGenerator` subscribes to `booking.confirmed` | Keep it on the bus | The receipt is derived from an already-confirmed booking and may be regenerated. |
| 6 | Schedule the reminder | `ReminderScheduler` subscribes to `booking.confirmed` and `booking.cancelled` | Keep it on the bus | Reminder state is downstream work that can follow booking state changes. |
| 7 | Refresh dashboard statistics | `StatsUpdater` subscribes to `booking.confirmed` and `booking.cancelled` | Keep it on the bus | Dashboard freshness should not control whether a booking or cancellation succeeds. |
| 8 | Record and pay a refund | `RefundLedger` subscribes to `refund.requested` | Keep it on the bus | Cancellation should complete even if the payment provider is temporarily unavailable. |

### Two calls in detail

Pick two rows and give each a paragraph rather than a line. One of them has to
be the operation `TICKET.md` implicates.

**Index a confirmed booking for search.** The caller of `book` needs a durable
room reservation back during the request; it does not need the room-finder's
search document before that response. I would move indexing to a subscriber
for the existing `booking.confirmed` event, which is published after the store
write. This bus still invokes the handler synchronously, but it catches a
handler exception, records a dead letter, and lets `publish` return. As wired
today, an unavailable search cluster makes `indexBooking` throw
`SearchIndexUnavailableError`, so the page or kiosk gets an error and the
booking is never saved. With the proposed boundary, the caller gets the
confirmed booking while failed indexing is visible for retry or repair, and
search may be temporarily stale.

**Save the confirmed booking.** The caller needs `store.save` to finish before
`book` reports success because that write is the operation that actually holds
the room and makes later conflict checks see it. I would therefore keep this
call synchronous and in the request path rather than put it on the bus. If the
store is unavailable or the write throws, the caller should receive an error
and no confirmation should be reported or published, because the service
cannot honestly promise the reservation until its source of truth accepts it.

### The incident

**The call behind `TICKET.md`.** In `src/service/bookingService.ts`, method
`BookingService.book`, line 94 calls
`this.searchIndex.indexBooking(booking, room, now)`. When that call throws,
`book` does not catch it, so its caller gets `SearchIndexUnavailableError`
instead of a `Booking`; execution never reaches `store.save` or the
`booking.confirmed` publish.

---

## Milestone 2: what a second delivery does

### Your prediction (before you run anything)

**What you expect `redeliver` on a refund message to do.** `RefundLedger.handle`
has no duplicate check, so I expect each redelivery of the same message to
append another ledger entry and call the payment provider again, producing
another settlement for the same booking and amount. The bus should increase
that message's delivery count on each redelivery, but its message log should
still contain only the original `refund.requested` message with the same id;
`redeliver` delivers the stored message rather than publishing a new one.

### Before your fix

**The run.** Paste what `npm run demo:replay` printed without editing it. A TA
may ask you about it in milestone 3.

```text
npm notice run campus-reservations@1.0.0 demo:replay
npm notice run tsx src/demos/replayDemo.ts
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3

=== after the cancellation (one delivery of the refund message) ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3

=== after one redelivery ===
ledger entries: 2
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-2 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 2
gateway settlements: 2
  stl-1 bk-1001 $85.00
  stl-2 bk-1001 $85.00
money returned in total: $170.00
dead letters: 0

redelivering msg-3 again

=== after two redeliveries ===
ledger entries: 3
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-2 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-3 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 3
gateway settlements: 3
  stl-1 bk-1001 $85.00
  stl-2 bk-1001 $85.00
  stl-3 bk-1001 $85.00
money returned in total: $255.00
dead letters: 0

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=3
```

**Your prediction against the run.** The run matched the prediction: each of
the three deliveries added one ledger entry and one settlement, raising both
counts from 1 to 3 and total money returned from $85.00 to $255.00. The bus log
still had one refund message, `msg-3`, with its delivery count raised to 3.

### Your fix

**The diff.** Paste it.

```diff
diff --git a/src/handlers/refundLedger.ts b/src/handlers/refundLedger.ts
index 600ffca..2daea49 100644
--- a/src/handlers/refundLedger.ts
+++ b/src/handlers/refundLedger.ts
@@ -24,6 +24,8 @@ export class RefundLedger implements Subscriber<RefundRequested> {
   readonly name = 'RefundLedger';

   private readonly entries: LedgerEntry[] = [];
+  private readonly recordedMessageIds = new Set<string>();
+  private readonly processedMessageIds = new Set<string>();
   private readonly gateway: PaymentGateway;

   constructor(gateway: PaymentGateway) {
@@ -31,16 +33,23 @@ export class RefundLedger implements Subscriber<RefundRequested> {
   }

   handle(message: Message<RefundRequested>): void {
+    if (this.processedMessageIds.has(message.id)) {
+      return;
+    }
     const event = message.payload;
+    if (!this.recordedMessageIds.has(message.id)) {
+      this.entries.push({
+        entryId: `led-${this.entries.length + 1}`,
+        bookingId: event.bookingId,
+        organizer: event.organizer,
+        amountCents: event.amountCents,
+        reason: event.reason,
+        recordedAtMinute: message.publishedAtMinute,
+      });
+      this.recordedMessageIds.add(message.id);
+    }
     this.gateway.refund(event.bookingId, event.amountCents);
+    this.processedMessageIds.add(message.id);
   }

   /** Every ledger entry, oldest first. */
```

**What your fix uses to decide what to do on a second delivery, and what it
does not protect.** It records the message id separately after the ledger
write and after the payment succeeds: a retry can skip an existing ledger
entry but still finish an unpaid refund, while a completed message skips both.
It does not protect a different message id for the same refund, a restart that
loses these in-memory sets, or a crash after the provider accepts payment but
before the completed-message id is recorded.

### After your fix

**The run.** Paste what `npm run demo:replay` printed on the fixed code.

```text
npm notice run campus-reservations@1.0.0 demo:replay
npm notice run tsx src/demos/replayDemo.ts
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3

=== after the cancellation (one delivery of the refund message) ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3

=== after one redelivery ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3 again

=== after two redeliveries ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=3
```

**What stays the same across its three states.** After cancellation, one
redelivery, and two redeliveries, the ledger stays at 1 entry, the gateway at
1 call and 1 settlement, total money returned at $85.00, and dead letters at
0. Only the bus delivery count changes, from 1 to 2 to 3.

---

## Milestone 3: a delivery that stops halfway

### The failure

**The run.** Run `npm run demo:failure` and paste what the delivery left
behind (the ledger, the provider, and the dead-letter list).

```text
npm notice run campus-reservations@1.0.0 demo:failure
npm notice run tsx src/demos/failureDemo.ts
armed: call 1 to the payment provider will throw
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3
the cancel call itself returned: booking is cancelled

=== after the failed delivery ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 0
money returned in total: $0.00
dead letters: 1
  msg-3 refund.requested attempt 1 in RefundLedger: payment provider did not answer on call 1 for bk-1001
    failedAtMinute: 480
    payload: {"bookingId":"bk-1001","organizer":"H. Ahmad","organizerEmail":"hahmad@example.edu","amountCents":8500,"reason":"cancelled more than two hours ahead","requestedAtMinute":480}

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=1

the refund message is in the log and in the dead-letter list. Milestone 3 asks what you do with it.
```

### Your recovery

**What, if anything, the failed delivery left undone.** The handler appended
one $85.00 ledger entry, then the provider call threw. No settlement was made,
so the organizer was still owed the entire $85.00; the message id was not
marked processed because the handler never reached that final step.

**The option you took and its evidence.** If you redelivered the
message, paste the state after the redelivery and the bus log line for that
message. If you left the message for a person to handle, write the runbook line
they would follow, and name which fields of the dead-lettered message they read to
follow it and what each one tells them.

I redelivered `msg-3`. The healed provider settled the $85.00 refund, so the
organizer received the money. The recorded-message id made the retry keep the
first ledger entry instead of appending a duplicate, while the dead letter
remains as the historical record of attempt 1.

```text
redelivering msg-3

=== after redelivery of the failed refund message ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 2
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 1
  msg-3 refund.requested attempt 1 in RefundLedger: payment provider did not answer on call 1 for bk-1001
    failedAtMinute: 480
    payload: {"bookingId":"bk-1001","organizer":"H. Ahmad","organizerEmail":"hahmad@example.edu","amountCents":8500,"reason":"cancelled more than two hours ahead","requestedAtMinute":480}

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=2
```

### The other option

**What the option you did not take would have cost.** Manual triage would make
an operator inspect the provider and issue or reconcile the refund, delay the
organizer's money until that person acts, and be safe only after the operator
verifies that no settlement for this booking and amount already succeeded.
