# Upgrading to 3.0.0

Everything 3.0.0 asks of you, in the order it is likely to bite. Most installs need only step 1, and it
happens by itself.

Coming from 1.x? Read [Upgrading to 2.0.0](upgrading-2.0.md) first — both index rebuilds apply, and 2.0.0
asks for more than this release does.

---

## 1. Your index rebuilds once, automatically

**Who it affects:** everyone who has run `index_repo`.
**What you do:** nothing. The first `index_repo` or `search_code` after upgrading takes longer.

Sankshep records which vector-store build wrote an index, and that identity now carries the native
library's version. An `index.db` written by 2.0.0 is deleted and rebuilt on first use rather than read by
a different build of the extension.

This is derived data. **`facts.db` and `stats.db` are untouched** — everything you have remembered, and
your accumulated savings figures, survive. Only the embeddings are recomputed, which is minutes on a large
repository and seconds on a small one.

!!! note "Why it is forced rather than offered"
    The alternative is reading an index written by a different build of a native extension and hoping the
    formats agree. A rebuild is slow once; a silently wrong nearest-neighbour result is wrong every time
    and looks fine.

---

## 2. Intel Macs cannot install or update to this release

**Who it affects:** macOS on Intel hardware (`osx-x64`). Apple Silicon is unaffected.
**What you do:** stay on 2.0.0, or run the container image.

`sankshep.osx-x64` is no longer published. It existed up to 2.0.0 and should not have:
**Microsoft.ML.OnnxRuntime ships no `osx-x64` native**, so the package installed, ran, and answered
`--version` — and then failed the first time anything needed an embedding. Over stdio `index_repo` and
`search_code` failed while the other six tools worked; over HTTP readiness never flipped, so the server
answered 503 for the life of the process.

Withdrawing a platform people were partly using is a breaking change, which is why this release is 3.0.0.

**Your options, in order of least disruption:**

1. **Stay on 2.0.0.** Those packages remain on nuget.org — published packages are immutable — and behave
   exactly as they did. You keep the six tools that worked.
2. **Use the container image**, which runs fully on an Intel Mac because it is published for
   `linux/amd64`:

    ```bash
    docker run --rm -i -v "$PWD:/repo:ro" ghcr.io/nitinpawar28/sankshep:v3.0.0 serve --repo /repo
    ```

    See [Deployment](deployment.md) for the full configuration.

---

## 3. `compose_task_prompt` returns a larger prompt at the same `tokenBudget`

**Who it affects:** anyone calling `compose_task_prompt` with an explicit `tokenBudget`, or sizing a
context window against it.
**What you do:** check your arithmetic if the number was tight. Nothing, if it was not.

`tokenBudget` used to be **split** between the code and the remembered-conventions section — roughly 70/30
— so asking for 8,000 tokens gave you about 5,600 tokens of code. It now **bounds the code**, and the
conventions section is additive with its own budget of 600 tokens.

So the same request returns **more code than before**, and the rendered prompt is **larger than the number
you passed**, by the conventions plus the template's own text. `PromptTokens` in the result reports the
whole rendered prompt, and always has.

!!! info "Why it changed"
    The composer is a wrapper around the `get_context` engine, and it was running that engine at 70% of
    the caller's budget. Measured on the flagship benchmark, it scored **0.51 mean key-point recall against
    that engine's 0.66** on the same questions at the same budgets — worse than the thing it wraps, for a
    reason that was pure arithmetic. After the change it reaches 0.66–0.68. See
    [Benchmarks](benchmarks.md), which also states the judge's error bar of about ±0.05 so you can judge
    whether that difference is real.

---

## What has not changed

- **Tool names, arguments and result shapes** — apart from `compose_task_prompt`'s `tokenBudget` above.
- **`get_context`'s `tokenBudget`**, which still bounds the whole response including its header.
- **Environment variables**, the `.sankshep/` layout, and exit codes.
- **Zero egress by default.** The server makes no outbound calls; that is asserted by tests on every
  build, and the only component that talks to a model provider is the maintainer-run eval harness, which
  is not part of the shipped server.
