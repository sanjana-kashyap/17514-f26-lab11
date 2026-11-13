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
| 1 | Validate request and price it | In the request path (`bookingService.ts:65-68`) | In the request path | Caller needs a yes/no and a price now |
| 2 | Check the room is free | In the request path, reads the store (`:71`) | In the request path | Must be decided before the booking is held |
| 3 | Index booking in the search index | In the request path, before the save (`:94`); throws if search is down, so the booking fails | After the request path, off the `booking.confirmed` event | Only the room-finder page reads it; it can catch up later |
| 4 | Save the booking in the booking store | In the request path (`:95`) | In the request path | It is the booking; the caller's answer depends on it |
| 5 | Add a record to the audit trail | In the request path (`:96`) | In the request path | Local and cheap; the record should exist whenever the booking does |
| 6 | Publish an event that denotes confirmed booking | In the request path (`:114`), after the save | In the request path | Cheap hand-off; it is what lets the rest run after |
| 7 | Send the confirmation email | Subscriber, runs inside `publish` | After the request path | Caller does not need it; a mail failure should not block a booking |
| 8 | Receipt, reminder, stats | Subscribers, run inside `publish` | After the request path | Nothing in the response depends on them |

### Two calls in detail

Pick two rows and give each a paragraph rather than a line. One of them has to
be the operation `TICKET.md` implicates.

**First operation and your call: indexing in the search index (row 3).** The
caller needs the booking held and a `Booking` back now, not the room-finder row.
Today `indexBooking` (`bookingService.ts:94`) runs before `store.save` and throws
`SearchIndexUnavailableError` when search is down. `book()` doesn't catch it, so
the caller gets an error and nothing is saved. I would save, publish
`booking.confirmed`, and index from a subscriber, so an outage becomes a dead
letter to redeliver later.

**Second operation and your call: the confirmation email (row 7).** The caller
needs nothing from it; the organizer wants it soon, not in the request. It is
already a subscriber (`emailConfirmationHandler.ts:34-52`) that only queues the
mail, and a handler failure becomes a dead letter without failing `publish`. I
would keep it. Redelivery is safe because the outbox is keyed by template and
booking id (`mailer.ts`), so a second delivery replaces the first.

### The incident

**The call behind `TICKET.md`.** `src/service/bookingService.ts`, `BookingService.book`,
line 94: `this.searchIndex.indexBooking(booking, room, now)`. When the cluster is
down it throws `SearchIndexUnavailableError`, which `book()` doesn't catch, so
the caller gets that exception instead of a `Booking`. It throws before
`store.save` (line 95), so nothing is saved, audited or published. Cancellations
never touch the index, which is why they kept working.

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
