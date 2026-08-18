# Testing

918 tests. 918 passing. 0 failing, 0 skipped.

This covers what's tested and how it was checked. It doesn't reproduce the test suite or name any
test.

---

## The suite

| Measure | Value |
|---|---|
| Tests | 918 |
| Passing | 918 |
| Failing | 0 |
| Skipped | 0 |
| Test suites | 12 |
| Build step | none |
| Network | not needed |

Twelve suites: one per module, plus one for the composed runtime.

The suite runs from a clean copy on a machine with no network and no install step, because there's
nothing to install. Zero runtime dependencies, zero dev dependencies, zero peer dependencies. What
gets verified is the thing that ships, not a build of it.

---

## Each module on its own

All eleven modules are tested alone, against their own job, with no other module present.

That's possible because they don't depend on each other. There's no fixture standing up ten
collaborators to exercise the eleventh, and no test that passes only because a sibling happened to
be there. A module's suite exercises exactly the failure that module exists for.

Practical upshot if you're reviewing the code: a module's tests are a spec for that module. They
aren't doing double duty as an integration harness, so you can read them on their own.

## The composed runtime

Separately, all eleven run together over one loop.

The integration tests aren't a bigger repeat of the module tests. They target the things that only
exist once the modules are composed:

- Precedence. When modules disagree the most restrictive verdict wins, and the outcome is the same
  regardless of what order the verdicts arrived in.
- Reasons. A combined verdict carries the reasons from every module that contributed, not just the
  winner's.
- Ordering. Authorization finishes before the side effect. On a deny, the tool is never called.
- Refusals get recorded. A blocked attempt is written into the run with its verdict, so it's
  auditable rather than missing.

## Breaking it on purpose

Since Conductor is a safety layer, what it does when its own parts break was tested by breaking
them rather than assumed. Each of these was induced deliberately:

| What was done | What happened |
|---|---|
| Made a gate throw | Denied. The tool never ran. |
| Made a gate return nonsense | Denied. The tool never ran. |
| Had one module allow while another denied | Denied. The tool never ran. |
| Made an observer throw | Error Signal emitted. Run continued. No crash. |

The property being checked is that a broken safety component fails toward saying no, never toward
saying yes. A gate that crashes doesn't become a gate that isn't there.

## Determinism

Determinism is checked, not claimed, two ways.

Recordings are byte-identical. The same scenario recorded three separate times produced a single
SHA-256 across all three. Not equivalent recordings, identical bytes.

Replay diffs clean. A recorded run replayed against its original showed no divergences anywhere in
the decision path: same events, same signals, same verdicts, same order.

Gate verdicts are stable. Every verdict in the demo scenario was identical across repeated runs.

There's one deliberate exception, documented rather than hidden. Some randomness in the product
has to stay random. A predictable canary token isn't a tripwire, and a fixed initialization vector
is a cryptographic defect. That randomness is confined to values that never reach the recording.
Determinism covers the recorded run and the safety verdicts, both verified stable. The related
identifier-stability limitation is in [CERTIFICATION.md](CERTIFICATION.md).

---

## Three demos you can reproduce

Three demonstrations, each one a claim you can check yourself rather than take on our word.

### 1. The full-stack run

One scripted agent, one mission, all eleven modules acting on the same live run, then replayed
identically.

The mission: research and summarize a market, don't modify files, don't spend money.

| # | Module | What it did |
|---|---|---|
| 1 | Compass | 8 on-mission actions allowed, 1 off-mission denied |
| 2 | Sentinel | 1 broken tool result quarantined |
| 3 | Timekeeper | 1 stale fact flagged |
| 4 | Ledger | 1 ungrounded action blocked, then allowed once sourced |
| 5 | Anchor | Drift verdict: objective substitution, peak 0.647 |
| 6 | Recall | 6 memories stored, 6 recalled for the summary |
| 7 | Warden | 2 actions denied by the policy and threat gate |
| 8 | Rewind | 15 steps recorded, reproduced, and diffed |
| 9 | Revert | 5 actions journalled, 4 flagged irreversible |
| 10 | Airlock | 1 tainted egress blocked off-scope |
| 11 | Keyring | 1 ephemeral scoped credential leased just in time |

Last line: `replay identical: true`

The interesting part isn't the number of denials, it's that they're unrelated. Three different
subsystems refused three different things in one run, for three unrelated reasons: an off-mission
file delete, a threat reaching a tool, and an encoded exfiltration attempt. That's composition
doing something no single check does.

Those numbers are held to the real run by an automated check that executes the demo and fails if
any published figure isn't produced by it. The check allows the summary to show fewer results than
the run emits, since it's a summary. What it forbids is showing a figure that isn't real.

### 2. Determinism

Record the same scenario twice to separate files, hash both. The hashes match. Replay it and it
reports no divergences.

Worth running rather than reading about, because it's the one claim nobody can argue with. Either
the hashes match or they don't.

### 3. Fail-closed

Drop a deliberately broken safety module into a working runtime and try a dangerous action
through it.

Make the module throw when asked for a verdict: the action is denied and the tool never runs. Make
it return meaningless output instead: denied again.

This is the one that matters most for safety software, and the one that can't be faked, because
what you're watching is the tool function not getting called.

---

## Where it was verified

Clean extraction, networking disabled, Windows 11, Node.js 24. Node.js 20 or newer required.

| Check | Result |
|---|---|
| Tests | 918 total, 918 passing, 0 failing |
| Test suites | 12 |
| Runtime, dev, and peer dependencies | 0 |
| Internal imports resolved | 440 of 440 |
| Unresolved modules | 0 |
| Build step | none |
| Network | not needed |
| Deterministic replay | verified, byte-identical |

## Coverage

918 passing tests is a count, not a measure of how much code those tests reach. Coverage was
measured separately across all twelve suites.

| Measure | Mean across suites |
|---|---|
| Line | 91.82% |
| Branch | 78.20% |
| Function | 91.14% |

Per-suite numbers are in [CERTIFICATION.md](CERTIFICATION.md), along with the composition layer
measured separately. Every file in that layer is at 94.81% line coverage or better, and the
authorization path, the code deciding whether a tool runs at all, is at 100% line and 100%
function. The most important path is also the most exercised.

Branch coverage is the weaker number throughout, 78.20% against 91.82% for lines.

## Platform

Conductor is a Windows product, built, verified, and sold for Windows. Everything here was
produced on Windows 11 with Node.js 24. macOS and Linux aren't supported and haven't been tested.

## What isn't claimed

A test record listing only successes isn't a test record.

- No third-party security audit. The security-critical modules have been checked internally
  through the test suite, adversarial failure testing, structural analysis, and fuzzing, but no
  independent firm has reviewed them, and internal checking isn't a substitute for that.
- Anchor's cost grows quadratically with run length. Found by benchmarking, measured, and
  disclosed with numbers in [CERTIFICATION.md](CERTIFICATION.md). Not fully fixed.
- Branch coverage trails line coverage by about fourteen points. One file in the composition layer
  sits at 55.56% branch, the lowest point.
- Signal and audit-reason identifiers aren't stable across runs. Behavior and verdicts are stable
  and verified, the identifiers attached to them aren't.
- No demonstration recordings have been made.

All tracked as open items in [CERTIFICATION.md](CERTIFICATION.md).
