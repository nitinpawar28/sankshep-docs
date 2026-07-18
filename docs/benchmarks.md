# Benchmarks

Sankshep's pitch — *maximum context, minimum tokens* — is only credible if the savings are **measured,
not marketed**. Two things are quantified: how many tokens are saved, and whether answer quality holds.

## Latest measured results

A real, **proprietary production C# service** (private) — 8 questions across its order-execution
subsystem, **50 atomic, verified facts**, each read and checked against the cited method. Individual
files range from ~750 to ~37,000 tokens, so the set exercises both small files and a file far larger
than a typical budget. An LLM judge (Claude Opus) scores key-point recall; compression is delivered
tokens against the original size of the files actually delivered.

| Level | Mean recall | Mean compression |
|---|---|---|
| Conservative | 0.94 | 19.1% |
| **Balanced** *(default)* | **0.94** | **30.4%** |
| Aggressive | 0.11 | 87.9% |

!!! success "The headline"
    **Balanced holds Conservative's recall (0.94) while compressing ~11 points more (30.4% vs 19.1%).**
    Query-targeted body-collapse keeps the code that answers the question and elides the rest, so the
    model pays fewer tokens for the same answer. On the largest file (~37k tokens) Balanced delivers
    **1.00 recall at 35–37% compression** — a file too big to want whole, cut to what the question needs
    without dropping a fact.

Per question (recall @ compression):

| Question (order-execution subsystem) | Conservative | Balanced | Aggressive |
|---|---|---|---|
| routing resolution | 1.00 @ 3% | 1.00 @ 10% | 0.00 @ 82% |
| rule validation | 1.00 @ 3% | 1.00 @ 5% | 0.17 @ 79% |
| signal fan-out *(37k-token file)* | 1.00 @ 30% | 1.00 @ 35% | 0.25 @ 95% |
| risk gates *(37k-token file)* | 1.00 @ 30% | 1.00 @ 37% | 0.00 @ 95% |
| placement guard | 1.00 @ 34% | 1.00 @ 56% | 0.00 @ 88% |
| broker dispatch | 0.50 @ 9% | 0.67 @ 13% | 0.17 @ 88% |
| status sync | 1.00 @ 29% | 0.86 @ 40% | 0.14 @ 87% |
| terminal settlement | 1.00 @ 16% | 1.00 @ 48% | 0.17 @ 89% |

!!! warning "Stated honestly"
    - **Aggressive is lossy by design.** It collapses the very bodies that hold the answers (recall 0.11),
      so it is for maximum compression of structure and signatures — never point it at a "how does X work"
      question.
    - **A file larger than the budget is delivered whole or not at all.** `get_context` treats each file as
      one atomic chunk; size the budget to clear the largest relevant file (sub-file chunking is on the roadmap).
    - **A fact that lives only in a doc-comment drops at Balanced.** Balanced strips doc-comments, so a key
      point documented *only* in a `///` comment (not in the code itself) is recalled at Conservative but not
      Balanced — this is why *status sync* dips to 0.86 vs Conservative's 1.00.
    - **Some misses are retrieval, not minimization.** *broker dispatch* sits at 0.50–0.67 even at
      Conservative and Balanced — those facts are missing regardless of minimization. (Note Balanced *beats*
      Conservative there: eliding noise surfaced a fact the fuller context buried.)
    - The exact percentage is **repo- and query-dependent** — this is one measured run on one codebase, not a
      universal figure.

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

## Reproduce it yourself

The numbers above come from the eval harness running a question set through the real server and an LLM
judge — regenerate them on any repo:

- **Live, per-tool:** `token_report` returns cumulative compression on your own repo. No dollar figure:
  Sankshep does not know your model or your negotiated rate.
- **Reproducible:** the eval harness takes a YAML of `{ id, query, paths, token_budget, key_points }` (facts
  made **atomic** and verified against the cited code), drives the server over stdio, and emits a
  recall-vs-compression table (plus a composed-vs-naive comparison) you can regenerate.

!!! note "Roundtrips"
    The composer's headline benefit — *fewer roundtrips* — is the hardest to quantify in a single-shot
    harness (it needs an agent simulation). Token reduction and quality retention are measured directly;
    roundtrips-avoided is reported qualitatively until an agent-sim benchmark lands.
