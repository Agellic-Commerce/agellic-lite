# Tokens, cache, and the queue

Agellic Lite is built for a base Keepa plan, where Keepa refills **1 token per
minute** and a single deep product read costs several tokens. Three mechanisms
work together so that limit stays out of your way: a token budget that never
overspends, a cache that never pays twice, and a background queue that drains
itself as tokens refill. This doc explains all three. For the per-tool token
costs, see [TOOLS.md](./TOOLS.md).

---

## The token budget

Every Keepa call costs Keepa tokens, and Keepa meters them per minute. Agellic
Lite mirrors your Keepa allowance in a local **token bucket**:

- The bucket **refills at your Keepa rate** (base Keepa is 1 token per minute).
- Its capacity is **your rate times 60**, so a base plan holds at most 60
  tokens at once. Tokens stop accruing once the bucket is full.
- On the first costly call after startup, Agellic Lite runs a **free check of
  your real Keepa balance** and aligns the bucket to it, so the very first cost
  estimate matches what Keepa will actually charge. You rarely need to set a
  rate by hand; the check detects it.

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

## The queue: too big to run now, so it runs later

At 1 token per minute, most real requests cost more than the bucket holds right
now. That is not an error. When a call needs more tokens than you have on hand,
Agellic Lite **queues it as a background job** and drains it automatically as
tokens refill. You keep chatting; the job works in the background.

### What a queued job shows you

`check_job_status` makes the queue transparent instead of a black box. For a
pending job it reports:

- its **position** in line,
- the **token balance it is waiting to reach** before it can start (on Agellic
  Lite that is simply the job's token cost; there is no extra reserve held
  back),
- an **ETA** derived from your Keepa refill rate,
- and a plain-English explanation of what it is waiting for.

### Cancel, resume, dedup, refund

- **Cancel before it starts.** `check_job_status` with `action: "cancel"`
  abandons a pending job that has not begun. A cancelled job reads as
  cancelled, not failed, and spends zero tokens. A job that has already started
  finishes on its own.
- **Durable across restarts.** The queue lives on disk. Quitting the app pauses
  it; relaunching resumes right where it left off.
- **Duplicates collapse.** Submitting an identical request while a matching job
  is still queued or running reuses that job instead of minting a second one
  that would double-spend tokens.
- **Failed calls are refunded.** If a call errors after reserving tokens, the
  reservation is returned to the bucket, so a transient failure does not cost
  you.

### It runs locally, so keep the app awake

The queue runs on **your machine, not in the cloud**. For jobs to make
progress, leave the Claude app open and the machine awake. Close the app and
the queue pauses (durably); reopen it and the backlog resumes.

### When a job is too big to ever fund

If a request is larger than your bucket could ever hold (more tokens than
`rate times 60`), no amount of waiting will fund it. Agellic Lite recognises
this and tells you the **exact maximum batch size** to split into, rather than
advising a wait that would never complete. Split the request into batches at or
below that size and each one queues and drains normally.

---

## Putting it together

A typical base-Keepa session looks like this:

1. You ask a question that touches more products than 60 tokens can cover.
2. Agellic Lite prices it, sees it exceeds the current balance, and queues it,
   telling you the position and ETA.
3. You keep working. The job drains in the background as tokens refill, at
   roughly one token per minute.
4. Results land in the cache, so every later look at the same products is free,
   in this chat or any other.

The metered plan is the constraint; the budget, cache, and queue are how
Agellic Lite makes a base Keepa key genuinely usable inside a conversation.
