# Benchmarks

Sankshep's pitch — *maximum context, minimum tokens* — is only credible if the savings are **measured,
not marketed**. Two things are quantified: how many tokens are saved, and whether answer quality holds.

## What's measured

| Metric | How | Where |
|---|---|---|
| **Tokens saved** | Raw (unminimized) vs. delivered token count, per call, using a real tokenizer | `token_report` (live, per tool) |
| **Quality retention** | An **LLM-as-judge** scores key-point recall — does the delivered context let a reader recover each key fact? | eval harness |
| **Composed vs. naive** | The [prompt composer](composer.md) vs. dumping the raw whole files: recall + tokens delivered | eval harness report |

## Two levers, measured separately

Savings come from two independent levers, and it helps to see them apart:

- **Retrieval (the big win)** — semantic search sends only the handful of relevant chunks instead of
  dozens of whole files. Most of the savings come from *not sending things at all*.
- **Minimization (the second win)** — AST transforms strip comments and collapse non-target method
  bodies to signatures, shrinking what remains.

C#/.NET compresses especially well (verbose namespaces, attributes, XML docs, brace-heavy blocks);
markup (HTML/templates) relies more on retrieval than minimization.

## The honest caveat

The exact percentage is **repo- and query-dependent**. The right way to state savings is with the eval
harness's measured numbers on a *named* repository and query set — not a single marketing figure.
Sankshep is built so that "how much did we save, and did quality hold?" is a number you **measure and
publish**.

- **Live, per-tool:** `token_report` returns cumulative tokens and dollars saved on your own repo.
- **Reproducible:** the eval harness runs a golden question set through the real server and an LLM judge,
  emitting a recall-vs-savings table (and a composed-vs-naive comparison) you can regenerate.

!!! note "Roundtrips"
    The composer's headline benefit — *fewer roundtrips* — is the hardest to quantify in a single-shot
    harness (it needs an agent simulation). Token reduction and quality retention are measured directly;
    roundtrips-avoided is reported qualitatively until an agent-sim benchmark lands.
