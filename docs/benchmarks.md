# Benchmarks

Sankshep's pitch — *maximum context, minimum tokens* — is only credible if the savings are **measured,
not marketed**. Two things are quantified: how many tokens are saved, and whether answer quality holds.

## What's measured

| Metric | How | Where |
|---|---|---|
| **Compression** | Delivered tokens vs. the original size of the files actually delivered, using a real tokenizer. Context *searched* is reported separately and is never a baseline. | `token_report` (live, per tool) |
| **Quality retention** | An **LLM-as-judge** scores key-point recall — does the delivered context let a reader recover each key fact? | eval harness |
| **Composed vs. naive** | The [prompt composer](composer.md) vs. dumping the raw whole files: recall + tokens delivered | eval harness report |

## Two levers, measured separately

Sankshep works two independent levers, and they are **measured differently on purpose**:

- **Retrieval** — semantic search sends only the handful of relevant chunks instead of dozens of whole
  files. This is where most of the practical benefit lives, but it is deliberately **not reported as
  tokens saved**: you can always "save" more by sending less, so the figure would be unbounded and
  meaningless. What matters is whether the *right* files were picked — that is **recall**, scored by the
  judge.
- **Minimization** — AST transforms strip comments and collapse non-target method bodies to signatures,
  shrinking what remains. This *is* reported, as **compression**: delivered tokens against the original
  size of the same files, so like is compared with like.

The tokens Sankshep *searched* to make that selection are reported for context and never subtracted from
anything. Counting them would claim credit against a baseline nobody could have sent — on a large
repository that arithmetic will cheerfully assert it saved more tokens than the model can even accept.

C#/.NET compresses especially well (verbose namespaces, attributes, XML docs, brace-heavy blocks);
markup (HTML/templates) relies more on retrieval than minimization.

## The honest caveat

The exact percentage is **repo- and query-dependent**. The right way to state savings is with the eval
harness's measured numbers on a *named* repository and query set — not a single marketing figure.
Sankshep is built so that "how much did we save, and did quality hold?" is a number you **measure and
publish**.

- **Live, per-tool:** `token_report` returns cumulative compression on your own repo. No dollar figure:
  Sankshep does not know your model or your negotiated rate.
- **Reproducible:** the eval harness runs a golden question set through the real server and an LLM judge,
  emitting a recall-vs-savings table (and a composed-vs-naive comparison) you can regenerate.

!!! note "Roundtrips"
    The composer's headline benefit — *fewer roundtrips* — is the hardest to quantify in a single-shot
    harness (it needs an agent simulation). Token reduction and quality retention are measured directly;
    roundtrips-avoided is reported qualitatively until an agent-sim benchmark lands.
