# merge-queue-like-bors (ARCHIVED)

**This repository is archived. Do not use this design.** It was an attempt to make
GitHub's native merge queue behave like [bors](https://github.com/rust-lang/bors) — run
expensive CI once, on the tip of a batch, instead of once per PR. Live testing proved the
design unsafe: under-tested code can merge to `main`. This README summarizes what we
found, for anyone tempted to build the same thing.

## The idea (and why it fails)

Bors solves two problems: tip-of-tree is always green (no "PR A and PR B pass individually
but break combined"), and CI is batched (expensive jobs run once per batch, not once per PR).
GitHub's native merge queue solves the first problem but not the second. This repo tried to
add batching on top by detecting the queue "tip" via the GraphQL API and running the full
suite only there, with earlier entries running cheap checks only and "waiting" for the tip.

**Root cause of the failure:** GitHub's merge queue has an asymmetric eviction model. When
an *earlier* entry fails, all *later* entries are rebuilt with new speculative-merge SHAs and
re-tested. But when a *later* entry (the tip) fails, *earlier* entries are never rebuilt —
they keep whatever status they already reported, and a "skipped" cheap-tier job counts as a
pass. GitHub has no notion that an earlier entry's "success" was conditional on the tip.

This breaks under **both** grouping strategies, confirmed empirically in `bootc-dev/ci-sandbox`:

- **ALLGREEN**: each entry merges independently the moment its own checks pass, with no
  regard for the tip at all. PR #76 (non-tip, heavy jobs skipped) merged at 21:54:24Z, two to
  three seconds *before* tip PR #77's heavy checks even finished failing (21:54:26–27Z).
- **HEADGREEN**: GitHub does wait for the tip's own result before merging earlier entries,
  but once the tip fails, it merges the earlier entry on its stale cheap-tier "success" rather
  than re-testing or evicting it with the batch. PR #78 (non-tip) merged at 23:50:24Z, three
  seconds after tip PR #79's checks had already failed, and 16 seconds before the tip was
  formally evicted.

Neither setting gives this design the guarantee it depends on: "the batch only merges once a
real, full-suite result on the tip is known good."

## Could this be fixed with custom GitHub Actions logic?

We looked into replacing the immediate cheap-tier "success" with a job that polls and blocks
until the real tip result is known (mirroring bors's gating), including having an AI agent
orchestrate retries/bisection on batch failure. Conclusion: not worth building.

- A polling/deferred-status design is possible in principle, but depends on several
  undocumented GitHub merge-queue behaviors (e.g. whether an in-flight `merge_group` run is
  cancelled when its entry is evicted or rebuilt), has no clean way to do bors-style
  bisection statelessly, and trades one kind of fragility (a stale-status bug) for another
  (a hand-rolled distributed state machine fighting an undocumented API).
- Real bisection (binary search to find the guilty PR in a failed batch) is a well-understood,
  small, deterministic algorithm. An AI agent adds no value there and introduces new risk
  (non-deterministic behavior, hallucinated tool calls) in the one place — the merge gate —
  that most needs to be boring and predictable. Agents are a good fit for *advisory* work
  alongside a deterministic queue (e.g. triaging a CI failure log against a diff and
  suggesting a likely culprit, or flagging a known flake), never for deciding what merges.

## What to use instead

Don't reimplement this on top of GitHub's native merge queue. Either:

- **Accept the cost of full CI on every queue entry.** For a public repo this is mostly a
  runner-queue-latency concern, not a billing one (GitHub-hosted Linux runners are free for
  public repos).
- **Use [Mergify](https://mergify.com)**, free for open source. It already solves this
  correctly: batch mode with real deterministic bisection on failure (fresh CI on each
  sub-batch, never a reused/guessed status), plus two-step CI and speculative checks. Several
  other open source projects (including other Red Hat projects) already use it.
- **Run an actual bors** (e.g. [rust-lang/bors](https://github.com/rust-lang/bors),
  bors-ng, homu) if you'd rather self-host than depend on a third-party GitHub App.

## Files

- [`.github/workflows/ci.yml`](.github/workflows/ci.yml) and
  [`.github/scripts/compute-ci-level.mjs`](.github/scripts/compute-ci-level.mjs) — the
  original tip-detection implementation, kept for reference.
- [`ci/`](ci/) — sentinel files (`ci/fail-validate`, `ci/fail-build`, `ci/fail-integration`)
  used to simulate failures at each tier during testing against `bootc-dev/ci-sandbox`.

## AI disclosure

Most of the code and this writeup were produced with a mix of foundation models via OpenCode,
including the empirical testing against `bootc-dev/ci-sandbox` referenced above.
Ref <https://github.com/cgwalters/cgwalters/commit/8eab755d45901dba6eacd3aaa54cabd7c02f1b03>

## License

Licensed under either of [Apache License, Version 2.0](LICENSE-APACHE) or
[MIT license](LICENSE-MIT) at your option.
