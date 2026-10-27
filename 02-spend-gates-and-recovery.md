# Spend Gates and Recovery for GMI Generations: Never Pay Twice

*Production patterns for running paid image, video, and audio generation through GMI Cloud from an autonomous agent, without surprise charges or duplicate work.*

## The problem

Autonomous agents generating media with real money at stake face three failure modes:

1. The agent spends without checking the price first
2. A request times out client-side, the agent retries, and gets charged twice
3. A run drains the balance on nice-to-haves before must-haves are generated

All three are solvable with a small amount of plumbing. These patterns are from several months of production use on GMI Cloud.

## Pattern 1: estimate first, get explicit approval, then submit

GMI's generation API exposes an estimate call that costs nothing. Make it a hard gate:

```
estimate_generation(prompt, params) -> price
   |
   +-- show price to the operator, wait for GO
   |
submit_generation(...)
```

Rules that matter:

- **Never chain a re-submit after a rejection without a fresh approval.** Agents love to "retry with improvements". That is a second charge
- **Estimates return the total for the requested duration**, not a per-second rate. Note the estimate is a list price; the amount actually charged can be lower after your organization's discount
- The estimate is also your sanity check: a wildly wrong price usually means a wrong parameter (see Pattern 4)

## Pattern 2: balance is checked per request, not per batch

GMI checks your balance at every submit. A cheap batch can succeed minutes before a bigger single submit is refused. Practical consequence: **generate must-have assets first** (the voiceover clips), nice-to-haves last (the music bed). If a nice-to-have gets refused at the end, you still have a complete deliverable.

Also: in our testing an insufficient-balance response arrived before queueing and nothing was charged, which is different from Pattern 3. Verify with `get_account_summary` after any failure rather than assuming.

## Pattern 3: a timeout is not a failure

This is the expensive lesson. When a submit call times out client-side (gateway 504, connection reset, tool timeout), the generation has very often **completed server-side anyway and been charged**. If your agent treats the timeout as failure and resubmits, you pay twice.

The recovery flow:

```
submit -> timeout?
   |
   +-- DO NOT resubmit
   |
list_generations(filter by model + recent) -> find the request_id
   |
get_generation(request_id) -> result URL -> download
```

Corollaries:

- Failed submissions may still charge. After any failure, check your account summary. If a phantom charge shows up, support can look at it with the request_id
- If a poller crashes after completion, the output is still retrievable by request_id, sometimes days later
- Result URLs follow a deterministic pattern per request_id. Once you hold one real result URL, you can often fetch siblings by swapping only the request-id filename segment. The folder segment is opaque per batch: copy it verbatim, guessing it 404s

## Pattern 4: parameter shape mistakes cause silent wrong-spend

Two rules that prevent wasted generations:

- **Model parameters go inside the `parameters` object**, not at the top level of the request. Top-level extras fail validation ("unexpected additional properties") on some tools and are silently ignored on others
- **Validate enum fields before submitting.** Voice IDs, duration values, and similar fields are enums with exact string values, and some tools accept the field on submit even when their schema shows a default. Passing a plausible-but-wrong value produces a wrong-but-successful generation: charged and unusable

## Pattern 5: the read path is free, use it as your flight recorder

`list_generations`, `get_generation`, `get_account_summary`, and `search_docs` cost nothing. Run them liberally:

- `list_generations` sorted by recency is the authoritative "what actually happened" log
- It also proves what is billable on your account: if a modality shows successful past generations, it bills. The console's billing page is the authoritative spend breakdown

## A minimal spend-gate pseudocode

```python
def generate_with_gate(prompt, params, must_have=True):
    price = estimate(prompt, params)
    if price > per_item_cap:
        return reject("over cap", price)
    if not approved(prompt, price):
        return reject("no GO", price)
    result = submit(prompt, params)
    if result.timed_out:
        return recover_by_listing(model=params.model)   # never resubmit
    return result
```

## What this buys you

In production this gate has never lost an asset and never double-charged. The one rule to remember: **on GMI, the expensive mistake is not spending too much on a generation, it is paying twice for one.**

*All samples in `samples/` were generated through these flows.*


---

*Disclosure: this tutorial was produced as part of the GMI Clouder program (paid). All GMI outputs shown are real generations from the author's own account.*
