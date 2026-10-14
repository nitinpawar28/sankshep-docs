# Benchmarks

Sankshep's pitch — *maximum context, minimum tokens* — is only credible if the savings are **measured,
not marketed**. Two things are quantified: how many tokens are saved, and whether answer quality holds.

## Measured results

!!! warning "Measured on 3.0.0 — not yet re-measured for 4.0.0"
    Every figure on this page was measured on **2026-09-20 against the code released as 3.0.0**, with
    `claude-opus-4-8` as the judge. **It is not a measurement of 4.0.0 and is not evidence about 4.0.0 in
    either direction.** 4.0.0 changed both halves of what this page measures. First, what `get_context`
    delivers when a call fills its budget: a file that does not fit now arrives as its relevant
    declarations instead of being truncated from the top or withheld whole, and the header now fits inside
    `tokenBudget`, so such a call carries up to 116 fewer tokens of code. Second, how compression is
    credited: a truncated
    excerpt used to be credited with its whole file's original size, so budget-bound compression included
    source that was withheld; 4.0.0 credits only the source that came back. So even the deterministic
    compression column is not comparable with a 4.0.0 run. Which way the numbers move, and by how much, is
    not something to estimate in a table: this one is labelled, not adjusted, until it is re-measured
    against the 4.0.0 tag. The same label applies to *Composed vs. naive* below, because the composer wraps
    the same engine.

    These figures replaced a table that was measured at **v1.7.0** (2026-07-18) and sat here labelled
    *v1.8.0* — wrong by a release — through two major versions. They are **not comparable to it**: that run
    used an 8-question set that lived in an untracked script and was never committed, so nobody, including
    its author, could re-run exactly it. This set is committed, so a number can be argued with.

A real, **proprietary production C# service** (private) — 8 questions across its order-execution
subsystem, **52 atomic, verified facts**, each read and checked against the cited method and then
confirmed by an adversarial re-read that refused 12 of 64 candidates. Individual files range from ~750
to ~37,000 tokens, so the set exercises both small files and a file far larger than a typical budget.
Seven of the eight token budgets are about a quarter of what their own question's paths hold; the eighth
question's paths hold about 200,000 tokens and its budget is capped at 8,000, about 4%. So no question
measures compression while saying nothing about retrieval.

| Level | Mean recall | Mean compression |
|---|---|---|
| Conservative | 0.50 | 38.5% |
| **Balanced** *(default)* | **0.67** | **59.5%** |
| Aggressive | 0.10 | 77.5% |

!!! success "The headline"
    **Balanced recovers more facts than Conservative — 0.67 against 0.50 — while compressing 1.5× as
    much.** Minimizing more did not cost quality here; it bought quality, and two different mechanisms
    are visible per question.

    **A fixed budget buys more relevant code.** On *upstox*, Conservative delivers from 5,352 tokens of
    original source and Balanced from 16,815 — three times as much source under the same budget, because
    each file arrives smaller. Both score 1.00; Conservative has no room to spare.

    **The same budget spent better.** On *kite*, both levels select the same 3,766 tokens and deliver
    within three tokens of each other — and recall goes from 0.33 to 0.83. Nothing about *how much*
    changed; what changed is *which* code survived.

Per question (recall @ compression):

| Question (order-execution subsystem) | Conservative | Balanced | Aggressive |
|---|---|---|---|
| broker HTTP transport & serialization | 0.50 @ 24% | 0.50 @ 51% | 0.00 @ 69% |
| broker auth & token lifecycle | 0.43 @ 69% | 0.43 @ 69% | 0.14 @ 75% |
| kite orders wire | 0.33 @ 39% | 0.83 @ 39% | 0.17 @ 75% |
| dhan super-order path | 0.88 @ 24% | 0.88 @ 53% | 0.25 @ 73% |
| upstox v3 lifecycle *(37k-token file)* | 1.00 @ 5% | 1.00 @ 73% | 0.00 @ 78% |
| broker order-service routing & guards | 0.25 @ 26% | 0.62 @ 40% | 0.25 @ 83% |
| strategy-runtime signal-to-order | 0.17 @ 61% | 0.50 @ 61% | 0.00 @ 76% |
| application order model & state | 0.57 @ 23% | 0.57 @ 89% | 0.14 @ 79% |

### How much the judge itself moves

An LLM judge is not a ruler, so the set was run **five times**: twice against one build, then three times
after the ranking change described below. The spread among runs of the same code is the honest error bar on
every recall figure above:

| | Balanced mean recall |
|---|---|
| Before the ranking change | 0.703, 0.649 |
| After it | 0.682, 0.661, 0.667 |

**Every compression figure was identical between runs of the same code** — both runs before the ranking
change, and all three after it — as it must be: compression is arithmetic on token counts and no judge is
involved. So **a recall difference smaller than about 0.05 on this set is
not evidence of anything** — including any difference you might compute between two rows of the table
above.

It is also why this page claims no recall gain from the ranking change shipped alongside these numbers.
Weighting a query term by how rare it is among the files being ranked raised Balanced compression from
56.2% to 59.5% — deterministic, identical on every run since — while recall stayed inside the spread it
already had. It changed what gets selected; whether that helps recall is not something five runs can say.

### Composed vs. naive

The [prompt composer](composer.md) against the control an assistant would otherwise use — the whole raw
text of every file the question points at, uncapped:

| Metric | Naive (uncapped) | Composed (question's budget) |
|---|---|---|
| Mean key-point recall | 1.00 | **0.66** |
| Tokens delivered | 139,841 | **38,320** |

**A 72.6% token reduction at 0.66 recall**, against a baseline that is uncapped by definition, because
that is what it is a control for. Handing a model every byte recovers every fact in this set; what the
composer answers is what a budget costs.

!!! warning "Stated honestly"
    - **About a third of these facts are not recovered at any level.** Conservative keeps every body and
      still scores 0.50. Against atomic implementation-detail facts at a budget of roughly a quarter of
      each question's own corpus (about 4% for the largest), the delivered context supports about two
      thirds of them at best. The
      set was built to be hard on purpose, and this is what hard looks like.
    - **Aggressive is lossy by design.** It collapses the very bodies that hold the answers, so it is for
      maximum compression of structure and signatures — never point it at a "how does X work" question.
    - **The gaps that survive every level are retrieval, not minimization.** *broker auth* scores 0.43 at
      Conservative *and* Balanced; *broker HTTP transport* scores 0.50 at both. Nothing minimization does
      explains a fact that is missing when every body is kept — those facts were in files the ranker did
      not choose, or in the part of a chosen file the budget cut.
    - **A file larger than the budget was truncated, not dropped, in the measured run.** Before 2.0.0 it
      was dropped whole; on 3.0.0, the code these numbers describe, it was truncated at a line boundary
      from the top, positionally rather than query-aware, and the header said so. 4.0.0 cuts such a file at
      its declarations, the query-relevant ones first, and truncates positionally only a file that cannot
      be cut. Whether that recovers more of the missing facts is **unmeasured**.
    - **Doc-comments are stripped at Balanced**, so a fact living only in a `///` comment cannot survive
      that level. This run does not isolate how many of the missing facts are of that kind, so no number
      is offered for it.
    - The exact percentage is **repo- and query-dependent** — this is one codebase, five runs, not a
      universal figure.

## What's measured

| Metric | How | Where |
|---|---|---|
| **Compression** | Delivered tokens vs. the original size of the source actually delivered — whole files, delivered declarations, or the source a truncated excerpt maps back to (4.0.0; the 3.0.0 figures above credited a truncated excerpt with its whole file) — using a real tokenizer. Context *searched* is reported separately and is never a baseline. | `token_report` (live, per tool) |
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
  size of the source that came back, so like is compared with like.

The tokens Sankshep *searched* to make that selection are reported for context and never subtracted from
anything. Counting them would claim credit against a baseline nobody could have sent — on a large
repository that arithmetic will cheerfully assert it saved more tokens than the model can even accept.

C#/.NET compresses especially well (verbose namespaces, attributes, XML docs, brace-heavy blocks);
markup (HTML, templates) is neither retrieved nor minimized — see
[Supported languages](usage.md#supported-languages).

## Reproducing the numbers {#reproduce-it-yourself}

The numbers above come from the eval harness running a question set through the real server and an LLM
judge.

!!! danger "The judge is a third-party API, and it receives your code"

    This is the one part of Sankshep that sends anything anywhere, and it is a **maintainer tool**: the
    shipped `sankshep` server does not reference it, and still makes zero outbound calls of its own.

    For every question the harness POSTs **the delivered context**, and for the optional
    composed-vs-naive comparison **the whole raw text of every file** listed in that question's `paths`,
    exactly as it is on disk. Nothing is redacted, sampled or truncated first. It goes to
    `https://api.anthropic.com/v1/messages` under `--judge anthropic`, or
    `https://api.openai.com/v1/chat/completions` under `--judge openai`.

    **So point it only at a repository you are allowed to send to a third party.**

    For a **zero-egress run**, use `--judge openai` with `OPENAI_BASE_URL` pointed at a local
    OpenAI-compatible server (llama.cpp, Ollama, vLLM, LM Studio) — the client speaks plain Chat
    Completions, so nothing else changes. Or `--judge offline`, which needs no key and no network: it
    proves the harness runs end to end and measures nothing about quality, and says so in its own output.

What you can regenerate yourself, and what you cannot:

- **Live, per-tool:** `token_report` returns cumulative compression on your own repo. No dollar figure:
  Sankshep does not know your model or your negotiated rate.
- **The recall table:** the eval harness takes a YAML of `{ id, query, paths, token_budget, key_points }`
  (facts made **atomic** and verified against the cited code), drives the server over stdio, and emits a
  recall-vs-compression table (plus a composed-vs-naive comparison). It is **not distributed** — no NuGet
  package or container image contains it — so today only the maintainer can run it.

!!! note "Roundtrips"
    The composer's headline benefit — *fewer roundtrips* — is the hardest to quantify in a single-shot
    harness (it needs an agent simulation). Token reduction and quality retention are measured directly;
    roundtrips-avoided is reported qualitatively until an agent-sim benchmark lands.
