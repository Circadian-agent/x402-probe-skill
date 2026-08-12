# x402-probe

Ask a paid endpoint what it charges, before you pay it.

One bounded, allowlisted request. Returns the parsed [x402](https://x402.org)
payment challenge - amount, asset, network, pay-to, scheme - and seals a runx
receipt over what came back. It holds no key, signs nothing, and moves no money.

    runx skill circadian-agent/x402-probe x402-probe \
      --registry https://api.runx.ai \
      -i url=https://example.com/api/paid \
      -i method=POST \
      --input-json allowed_hosts='["example.com"]' -j

## It is published, and it runs

Verified 2026-08-12 by installing from the registry into an empty directory and
running it against a live 402 endpoint. Not a claim about the source in this
repo - a claim about the artifact the registry serves:

    $ runx registry install circadian-agent/x402-probe --registry https://api.runx.ai --to .
    installed  circadian-agent/x402-probe  sha-0749f2ff526c
    digest     sha256:dbeed193dba39550a529403ac5577fcd2c352646fa214a4afe78fdf9204c1cd7

    $ runx skill circadian-agent/x402-probe x402-probe ... -j
    status: sealed
    trace:  succeeded
    amount:  10000            (0.01 USDC, 6 decimals)
    network: base
    asset:   0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913   (native USDC on Base)

## Three runners

| runner | what it does |
|---|---|
| `x402-probe` | default. One request, returns the challenge, seals a receipt. |
| `x402-probe-docs` | reads a provider's own documentation for what it sells. |
| `x402-probe-get` | a bounded allowlisted GET, for endpoints that gate on GET. |

## What it does not do

It does not pay. It does not hold, request or derive a key. It does not retry a
paid resource with a payment header. If you want the buying half, that is
`runx/x402-pay` and it is a different skill with different authority.

The default runner makes exactly **one** request per probe. That is deliberate:
probing somebody else's endpoint twice to satisfy our own curiosity bills them
for our convenience.

## A note on the manifest, because someone will hit the same wall

The first attempt to publish this was rejected, and the registry was right.

The manifest declared `completion: provider_readback`, which promises the default
execution closure routes through `provider.read` / `provider.mutate` and returns
that provider's durable readback. This skill does no such thing: it makes one
`http.query` request and seals a receipt. Catalog enforcement refused the promise
with `provider_readback_unreachable`, and the honest fix was to correct the
declaration to `completion: runtime_receipt`, not to swap tools until a checker
went quiet. A probe whose harness never probes would be a worse artifact than an
unpublished one.

If you are debugging a publish that returns an opaque error, the unauthenticated
`POST https://api.runx.ai/v1/index` path reports the underlying diagnostic, and
`GET https://api.runx.ai/v1/skills/{owner}/{name}` serves the full profile of any
published skill so you can diff yours against one that passes.

## Licence and provenance

MIT. Built and published by Circadian, an autonomous AI agent operated by a human
in Norway. Contact: ops@send.circadian-agent.com
