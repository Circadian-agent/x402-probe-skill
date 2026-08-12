---
name: x402-probe
description: Read an endpoint's x402 payment challenge without paying it, and report the real price, network, asset, payee and protocol version.
runx:
  category: research
---

# x402 Probe

Ask an endpoint what it would charge, and get an answer you can act on, without
spending anything to find out.

## What this skill does

`x402-probe` sends one request to a URL, reads the HTTP 402 payment challenge it
comes back with, and reports what that endpoint actually wants: the price in USD,
the settlement network, the asset contract, the address that would be paid, the
payment scheme, and which version of the x402 protocol it speaks.

It never pays. It holds no key, signs no authorization, and moves no money. The
whole point is to answer "what would this cost, and is it even real" before any
value leaves a wallet.

The answer includes a digest of the raw challenge, so a later step can cite the
probe rather than re-running it, and a reader can check the report against the
wire rather than trusting the summary.

## Why this exists

A public catalogue of x402 endpoints currently lists on the order of fifteen
thousand resources. Measurements published against that catalogue put the median
listing at roughly three cents of revenue a month, and listings go stale: an
endpoint can be delisted, repriced, moved to a different chain, or simply stop
answering, while its catalogue entry says otherwise.

Until now the only way for an agent to find out whether a listing was real was to
pay it. That is a poor trade when the listing might be dead, and a worse one when
the price in the catalogue is not the price on the wire.

This skill closes that gap. It costs nothing to ask.

## When to use this skill

- Before paying an unfamiliar endpoint, to confirm the advertised price matches
  what the endpoint actually quotes.
- When auditing a catalogue or directory, to separate listings that still answer
  from listings that only used to.
- When an integration starts failing, to see whether the endpoint changed network,
  asset or protocol version underneath you.
- Any time you want the price of something without committing to buy it.

## When NOT to use this skill

- **It will not buy anything.** If you want the resource, pay the challenge with a
  payment skill; this one deliberately cannot.
- It reports one endpoint per run. Sweeping a whole catalogue is the caller's
  loop, not this skill's job.
- A `200` on the first request means the endpoint did not ask for payment at all.
  That is a valid and useful answer, not a failure.

## Inputs

| input | required | default | meaning |
|---|---|---|---|
| `url` | yes | | The endpoint to probe. |
| `method` | no | `POST` | The runtime's `http.query` admits body-carrying methods. **`GET` is not admitted here** - use the `x402-probe-get` runner instead, see the limits section. |
| `body` | no | `{}` | Many endpoints return their challenge for an empty body, which is why this defaults to empty. |

## What it reports

- `speaks_x402` - whether a payment challenge was found at all.
- `x402_version` - **1 or 2, and this distinction is the whole trick.** A version
  1 server puts the challenge in the response body. A version 2 server puts it in
  a `PAYMENT-REQUIRED` header **and answers with an empty `{}` body**. A reader
  that checks the body first sees valid JSON with no payment options and concludes
  the endpoint is free or broken. It is neither. Read the header first.
- `price_usd`, `network`, `asset`, `pay_to`, `scheme` - the terms actually quoted.
- `challenge_digest` - a digest over the raw challenge, so the report is checkable
  against the wire.

## How to read the result

A quoted price is what the endpoint says **now**. It is not a promise, and it is
not the catalogue's price. Where the two disagree, this report is the one taken
from the endpoint itself, and the catalogue is the one that is stale.

`speaks_x402: false` means no challenge was found for that method and body. It
does **not** prove the endpoint is ungated: an endpoint can gate a path this
probe did not ask for.

**`GET` needs the other runner, and this used to be an outright gap.** The
runtime routes body-carrying methods through `http.query`, which refuses `GET`.
So a `speaks_x402: false` from the default runner is evidence about the `POST`
path only.

Use the **`x402-probe-get`** runner for a `GET`-gated endpoint. It goes through
`http.read` - "bounded allowlisted GET batch through the governed native
transport" - with the same `allowed_hosts` contract, so the authority story is
identical: hosts are declared and the runtime refuses anything outside the list
before a request leaves.

It is a separate runner rather than a second step on purpose. A second step would
fire a `GET` on every probe, doubling the requests made against somebody else's
endpoint to satisfy our own curiosity. The caller chooses, so the second request
only happens when someone asks for it.

**So the honest reading of a negative is still narrow:** absence of a challenge on
one method and body is not proof the endpoint is ungated. Probe both if it
matters.

## A price is only half an answer: the `x402-probe-docs` runner

An x402 challenge carries a price, an asset, a network and a payee. It carries no
description of the thing being bought. So a caller that probes an unfamiliar URL
learns it would pay a cent and still does not know for what.

The **`x402-probe-docs`** runner reads the provider's own machine-readable manual,
which by convention lives at `https://<host>/llms.txt`, within the same declared
allowlist and enforced at every redirect hop. Run it alongside a probe and the two
answer the question a buyer actually has: what is this, and what does it cost.

Given no `url`, it does not guess a host and does not return an empty read dressed
up as an answer. It **seals a `needs_agent`** naming what it is missing
(`url is missing`, `allowlist is missing`) and makes no request at all. A stop is
a legitimate outcome with a receipt over it, not an error.

## Provenance and limits

- One request per run. No retries that would multiply load on someone else's
  endpoint.
- `POST` only, for the reason above.
- **The caller must name the hosts it may reach.** Authority narrows: naming one
  host does not grant another, and a host outside `allowed_hosts` is refused by
  the runtime **before any request leaves**. Verified: probing `example.org`
  while only `example.com` is allowed fails with
  *"host \"example.org\" is outside allowed_hosts"* and no request is made.

  That refusal surfaces as a run status of `failure` rather than a sealed
  receipt, so it is documented here rather than shipped as a harness case: the
  refusal aborts the run mid-flight, after preparation has already succeeded, and
  an aborted run has no outcome for a fixture to compare against.

  **An earlier version of this section drew a wider conclusion than the evidence
  supported**, and it is corrected here rather than quietly rewritten. It said a
  fixture expecting a failure *could never* clear the release gate. That is not
  true - a published third-party skill declares exactly such a case - and the
  belief cost a release. What is true is narrower and is the sentence above: an
  abort inside a native tool has nothing to compare. The stop case this skill
  ships instead is `x402-probe-docs` with no url, which **seals** a `needs_agent`.
- Read-only: declared `execution: read`, no credential, no signing.
- The reported terms are copied from the challenge, not inferred. If a field is
  absent from the challenge it is absent from the report rather than guessed.
