# Tokens, cache, and work orders

Agellic Lite is built for a base Keepa plan, where Keepa refills **1 token per
minute** and a single deep product read costs several tokens. Three mechanisms
work together so that limit stays out of your way: a token budget that never
overspends, a cache that never pays twice, and **work orders**, durable records
of accepted work that fund themselves as tokens refill. This doc explains all
three. For the per-tool token costs, see [TOOLS.md](./TOOLS.md).

---

## The token budget

Every Keepa call costs Keepa tokens, and Keepa meters them per minute. Agellic
Lite mirrors your Keepa allowance in a local **token bucket**:

- The bucket **refills at your Keepa rate** (base Keepa is 1 token per minute).
- Its capacity is **your rate times 60**, so a base plan holds at most 60
  tokens at once. Tokens stop accruing once the bucket is full.
- At startup Agellic Lite runs a **free check of your real Keepa balance** and
  aligns the bucket to it, so the very first cost estimate matches what Keepa
  will actually charge. You rarely need to set a rate by hand; the check
  detects it.
- **Nothing is held back from background work.** On Agellic Lite the whole
  balance is spendable by a running work order (the full Agellic server keeps a
  reserve for interactive calls; Lite's reserve is zero).

You can set your rate explicitly if you want (the `AGELLIC_LITE_TOKENS_PER_MINUTE`
environment variable, or the Tokens per minute field in the Claude Desktop
form), but leaving it blank and letting the startup check detect your rate is
the intended path.

### Check before you spend

`check_token_balance` previews a call's cost against your live balance without
spending anything. Ask for it whenever you want to know "can I afford this
right now, and if not, how long until I can." It reports:

- your current balance and refill rate,
- the token cost of the operation you're about to run,
- whether it is affordable now, and if not, the wait or the split you need.

Because it prices in what's already cached (see below), a re-run that is mostly
cached can read as affordable immediately even when a cold run would not be.

One limit worth knowing: the wait this tool quotes is a hypothetical for a call
made right now, with nothing else on the key. It cannot see work already
queued. Once a call has actually created a work order, `check_job_status` is
what reports that order's real, queue-aware wait.

---

## The cache: never pay Keepa twice

Every product Agellic Lite fetches and every result set it builds is cached on
your machine, and the cache is **shared across every chat and every connected
app** on that machine.

- **Cached reads are free.** Re-opening a product you already pulled, or
  re-reading a finder result set, costs zero Keepa tokens.
- **It spans conversations.** A finder run you built this morning can be paged
  through in a fresh chat this afternoon without re-charging Keepa.
- **It spans hosts.** Configure once, and Claude Desktop and Claude Code on the
  same machine share the same cache and credentials.
- **Cost estimates account for it.** When you ask what a batch will cost, the
  estimate subtracts the ASINs you have already pulled and only prices the
  uncached ones.

On a 1-token-per-minute plan, the cache is the difference between paying for a
lookup once and paying every time you revisit it.

---

## Work orders: what happens when a call costs more than you have

At 1 token per minute, most real requests cost more than the bucket holds right
now. That is not an error, and there is no queue of "failed" calls waiting to be
retried. Every Keepa-calling tool turns your request into a **work order**: a
durable record, with its own id (`wo_...`), of work that has been accepted.

An accepted call comes back as exactly one of three answers.

### 1. The result, right now

If the balance covers the quote, the order runs inside the call and you get the
products, the match count, or the resolution table as usual. There is a work
order behind it, but you only hear about the id if the run stopped short, in
which case the reply opens with `PARTIAL` (resumable, keep polling),
`INCOMPLETE`, or `CANCELLED` (final).

### 2. Accepted for the background

If the balance is short, the call is **still accepted**. You get an `orderId`
and two separate claims:

- **Cost**, a flat figure: worst-case tokens at the rate it was quoted at. No
  time claim.
- **Queue and ETA**, underneath it: how many orders are ahead of yours and what
  they still have to spend, then a **bound** on the wait, phrased "done within
  ~X of token refill".

Cost and time are two different claims and Agellic Lite states them separately,
because only one of them is a fact about your order. What it costs depends on
your request; how long it takes depends on the whole key.

### 3. A quote waiting for your go-ahead

If the work needs more than **60 minutes of token refill** (on a base plan at 1
token per minute, roughly a quote above 60 tokens), nothing starts. You get the
`orderId`, a one-time `confirmToken`, the `Quote:` line, and the same Queue and
ETA pair framed as "If you confirm now:". Nothing is charged. The work begins
only when you agree and the assistant calls `confirm_work_order` with that id
and token.

On base Keepa this gate fires readily, and that is deliberate: a deep-dive of
four or more uncached products, or a manifest of more than about twenty
supplier codes, is already more than an hour of refill. You are being asked
before your afternoon's tokens are committed.

`execute_keepa_finder` is the one exception: a finder query is a single
indivisible Keepa call, so it has no consent step. It discloses its cost and
ETA as it accepts. If a query is priced above what your bucket could ever hold
at once, it is refused at the door with advice on the cheaper query to run, and
nothing is created or charged.

---

## Reading the ETA honestly

The ETA is a **bound, not a prediction**, and it is recomputed from scratch
every time you poll. It counts down as the queue drains and your balance
refills. It is never a stored number that sits still.

It carries three conditions, and they are worth relaying whenever you act on
the figure:

1. **It bounds the work as currently quoted.** Each order ahead of yours is
   priced at its own quote, and an order can outgrow its quote (cached entries
   expiring mid-run, for instance). It is not an unconditional ceiling.
2. **It assumes nothing else is spending this Keepa key.** Another app, another
   Agellic edition, or your own interactive lookups all draw from the same
   bucket. When that happens the next poll simply reprices, and the ETA can go
   up as well as down.
3. **The wall-clock figure assumes a session open about 8 hours a day.** Work
   only advances while a connected app is running. An always-open session
   finishes roughly three times sooner than the wall-clock estimate; a closed
   laptop makes no progress at all.

Polling costs nothing and speeds nothing up. Only the passage of time and
refill moves an order along.

---

## Watching, reading, and stopping an order

`check_job_status` is the hub for everything after a work order exists. It is
free, reads local state only, and never calls Keepa. Four actions:

- **`list`**: every recent order, newest first, with kind, state, rows done,
  and tokens charged.
- **`status`** (the default): the full report on one order. Rows settled versus
  total, tokens charged, the original quote, drift if the remaining work now
  needs more refill than the whole order was quoted at, the live Queue and ETA
  pair, and the deadline if one was declared.
- **`fetch`**: page through the rows the order has already settled, **mid-run
  or after it finishes**, rendered the way the tool that created it renders. You
  do not have to wait for a long order to finish to start reading it.
- **`cancel`**: stop it. An order that has not started yet stops instantly. A
  running one stops at its **next dispatch boundary**, and everything it
  settled before stopping stays readable with `fetch`.

### Authorised is not started

The status view separates two events that used to be conflated:

- **`authorised:`** is when the work was agreed to, either automatically or by
  your `confirm_work_order`.
- **`started:`** says only whether a Keepa request has actually been dispatched
  for it. An order still waiting its turn reads `not yet started`, naming how
  many orders are ahead.

An order can be authorised for a long time before it starts. That is the queue
working, not a stall.

### When the ETA is withheld

Agellic Lite would rather say nothing than promise a completion time it cannot
keep, so the Queue and ETA pair is **withheld entirely** for four kinds of
order: one still being staged (a draft, not yet priced), one that has been
asked to cancel, one whose commit log is unreadable, and one whose deadline has
passed. None of those finishes on refill alone, so a completion time would
contradict the message it sits in.

---

## What things cost, and what they settle at

A quote is a **worst-case ceiling**, not a price. A deep-dive order is quoted at
16 tokens per uncached product and typically settles at 6 to 8. Read the quote
as the ceiling you are authorising and the ledger figure as the cost.

Two details follow from that:

- **Charges settle at Keepa's own figure.** Each batch reserves its worst case,
  fires, and then settles to whatever Keepa actually billed. There is no
  whole-order reservation sitting on your balance.
- **A mid-run "charged" total can go down.** While requests are in flight, the
  charged figure includes tokens reserved for requests that have not come back
  yet. Those release when they settle, so the number you see mid-run is a
  high-water mark. The final figure on a finished order is the real cost.

An order waiting on refill holds nothing at all, and an order still awaiting
your consent has been charged nothing.

---

## Results persist, and so do the orders

- **Results are handles.** A finished order leaves a result set you can re-read
  for free: `lookup:<orderId>` for a product deep-dive, `finder:<orderId>` for
  a search, `codes:<orderId>` for a code resolution. The stored view holds a
  24-hour TTL; if it lapses, `check_job_status` rebuilds it from the order's own
  record at zero tokens. Re-running the original tool would pay Keepa twice.
- **Orders outlive the chat.** Finished work orders are kept for 14 days, so
  you can look up what a run cost and what it settled long after the
  conversation scrolled away.
- **They survive restarts.** The order lives on disk. Quitting the app pauses
  it; relaunching resumes exactly where it left off, nothing lost.

### It runs locally, so keep the app awake

Work orders run on **your machine, not in the cloud**. For an order to make
progress, leave the Claude app (or Codex, or ChatGPT desktop) open and the
machine awake. Close everything and the order pauses, durably; reopen and it
resumes.

---

## Putting it together

A typical base-Keepa session looks like this:

1. You ask a question that touches more products than 60 tokens can cover.
2. Agellic Lite prices it. If it is under an hour of refill it accepts the work
   and hands you a cost plus an ETA bound; if it is over, it hands you a quote
   and waits for your go-ahead.
3. You keep working. The order funds itself in batches as tokens refill, and
   you can read the rows it has already settled at any point with
   `check_job_status` `action: "fetch"`.
4. Results land in the cache and in a durable handle, so every later look at
   the same products is free, in this chat or any other.

The metered plan is the constraint; the budget, the cache, and durable work
orders are how Agellic Lite makes a base Keepa key genuinely usable inside a
conversation.
