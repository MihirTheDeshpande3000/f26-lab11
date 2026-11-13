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

**What you expect `redeliver` on a refund message to do.** Name what you expect
to change in the ledger, at the payment provider, and in the bus log. Write it
down, then run the demo.

### Before your fix

**The run.** Paste what `npm run demo:replay` printed without editing it. A TA
may ask you about it in milestone 3.

**Your prediction against the run.** Where they agree and where they do not.

### Your fix

**The diff.** Paste it.

**What your fix uses to decide what to do on a second delivery, and what it
does not protect.** One or two lines.

### After your fix

**The run.** Paste what `npm run demo:replay` printed on the fixed code.

**What stays the same across its three states.** Name the numbers you
compared.

---

## Milestone 3: a delivery that stops halfway

### The failure

**The run.** Run `npm run demo:failure` and paste what the delivery left
behind (the ledger, the provider, and the dead-letter list).

### Your recovery

**What, if anything, the failed delivery left undone.**

**The option you took and its evidence.** If you redelivered the
message, paste the state after the redelivery and the bus log line for that
message. If you left the message for a person to handle, write the runbook line
they would follow, and name which fields of the dead-lettered message they read to
follow it and what each one tells them.

### The other option

**What the option you did not take would have cost.** One line, in terms of who
does the work, when the organizer gets their money, and what has to be true for
it to be safe.
