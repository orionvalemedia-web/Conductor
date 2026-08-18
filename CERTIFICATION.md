# Certification

Measured numbers for Conductor.

Everything below came from running the software, not from copying a document. Verification was
done on a clean extraction of the shipped archive, networking disabled, Node.js 24. Node.js 20 or
newer is required.

This reports results. It doesn't explain how any of them are achieved.

---

## Headline

| Measure | Value |
|---|---|
| Modules | 11 |
| Shared contract | 1 |
| Tests | 918 passing, 0 failing, 0 skipped |
| External dependencies | 0 |

## Technical validation

| Result | What was checked |
|---|---|
| PASS | 918 of 918 tests passing, 0 failed, 0 skipped, 0 todo |
| PASS | Zero dependencies: runtime, dev, and peer, across all 12 component manifests |
| PASS | No third-party code, no external import anywhere in the shipped source |
| PASS | No build failures. There's no build step, and every source file parses and runs |
| PASS | 440 of 440 internal imports resolve to a real file |
| PASS | 0 unresolved modules |
| PASS | Hub and spokes: no dependencies between the eleven modules |
| PASS | Authorization happens before side effects. A denied action never reaches the tool |
| PASS | Gate verdicts identical across repeated runs |
| PASS | Byte-identical replay: three recordings of one scenario, one SHA-256 |
| PASS | Fails closed under adversarial conditions. A throwing gate, a corrupt verdict, and mixed verdicts each produced a deny with the tool never called |
| PASS | No hard-coded absolute paths in shipped code |
| PASS | Portable, no installation. Full suite passes from a clean copy on separate storage with no network |
| PASS | Runs offline. No sockets opened, no outbound calls |

## Test coverage

Measured with the platform's own coverage instrumentation across all twelve suites.

| Suite | Tests | Line % | Branch % | Function % |
|---|---|---|---|---|
| Airlock | 88 | 94.58 | 73.85 | 94.78 |
| Anchor | 66 | 95.14 | 83.71 | 95.16 |
| Compass | 82 | 93.19 | 77.86 | 90.91 |
| Conductor | 55 | see below | see below | see below |
| Keyring | 104 | 93.90 | 72.03 | 94.20 |
| Ledger | 74 | 94.10 | 80.76 | 91.18 |
| Recall | 60 | 94.33 | 86.71 | 95.00 |
| Revert | 72 | 94.00 | 76.87 | 93.44 |
| Rewind | 73 | 94.07 | 80.75 | 96.67 |
| Sentinel | 84 | 93.03 | 84.43 | 97.37 |
| Timekeeper | 63 | 92.50 | 75.00 | 89.29 |
| Warden | 97 | 92.86 | 83.17 | 96.88 |
| Mean | 918 | 91.82 | 78.20 | 91.14 |

The mean is an unweighted average across the eleven module suites, not a whole-product figure.

### The composition layer

The Conductor suite can't be summed up in one number the way a module suite can. Because the
runtime imports all eleven modules, the coverage tool attributes their files to this suite's
report too, and those score low here since Conductor's own tests only touch the parts they need.
Each of them is covered by its own suite at the rates above. Read the combined number as if it
described the composition layer and you'll understate it badly.

Measured on its own files only:

| Measure | Result |
|---|---|
| Lowest line coverage of any file in the layer | 94.81% |
| The authorization path: precedence and whether a tool runs | 100% line, 100% function |
| Files at 100% line coverage | 4 of 9 |

Every file in the layer is at 94.81% line coverage or better, and the code deciding whether a tool
runs at all is fully covered. The most important path is also the most exercised.

Branch coverage is the weaker number throughout, with a low of 55.56% on one file in the layer.

## Performance

Measured on Windows, Node.js 24. One iteration is one model-boundary crossing plus one
tool-boundary crossing. The work behind each crossing is fixed and trivial, so what's being
measured is the layer itself.

| Configuration | Mean | p50 | p95 | p99 |
|---|---|---|---|---|
| Bare loop, no Conductor | 0.0004 ms | 0.0003 | 0.0004 | 0.0008 |
| 6 gate modules | 0.0354 ms | 0.0287 | 0.0604 | 0.1115 |
| All 10 pipeline modules | 12.333 ms | 12.563 | 24.482 | 25.996 |

Building the runtime costs 0.0716 ms, once per run rather than per action.

Gating is basically free. All six gate modules together add 0.035 ms per iteration, so the
enforcement path sitting in front of every action costs almost nothing.

One module accounts for nearly all the measured cost. Per module, alone, over a 400-step run:

| Module | Mean per iteration |
|---|---|
| Anchor | 7.59 ms |
| Warden | 0.0401 ms |
| Recall | 0.0298 ms |
| Compass | 0.0273 ms |
| Airlock | 0.0212 ms |
| Ledger | 0.0211 ms |
| Sentinel | 0.0191 ms |
| Keyring | 0.0149 ms |
| Revert | 0.0142 ms |
| Timekeeper | 0.0119 ms |

Ten of the eleven cost between 0.012 ms and 0.040 ms. Anchor is the outlier, and its cost isn't
constant.

### Anchor's cost grows with run length

Anchor's analysis is run-scoped rather than step-scoped, since it reasons about the shape of the
whole trajectory instead of the latest step on its own. Per-step cost grows as a run gets longer,
and total cost for a run grows quadratically with its length.

| Run length | Mean per iteration | Total |
|---|---|---|
| 50 steps | 0.36 ms | 18 ms |
| 100 steps | 0.57 ms | 57 ms |
| 200 steps | 1.04 ms | 209 ms |
| 400 steps | 1.87 ms | 748 ms |
| 800 steps | 4.19 ms | 3,356 ms |

Partly fixed, and the rest is disclosed. Benchmarking found the cost, and two optimisations were
applied to cut redundant work out of that path. Both were checked to leave behavior bit-identical:
180 trajectories and 584 Signals compared against the pre-change build with zero differences, the
full 918-test suite passing, and the reference run still producing its published figures and
replaying to the same recorded hash.

That was a 2.5x reduction. Anchor's mean cost went from 19.08 ms to 7.59 ms, and an 800-step run
from 7,894 ms to 3,356 ms.

The growth curve itself hasn't changed. Cost per step still roughly doubles as run length doubles.
What came out was constant overhead, not the growth term. Getting rid of that would mean narrowing
what run-level analysis looks at, which changes what gets detected on very long runs and therefore
changes recorded output. That's a decision about what the product detects, not an optimisation, so
it was left open rather than made quietly.

For scale: a real model call usually takes 200 to 2000 ms. At the run lengths agents actually
reach, Conductor's total overhead stays well under a single model call and isn't the bottleneck.
It matters for unusually long runs, and the cause is known, measured, and bounded.

## Fuzzing

The decode scanner, the part that unwraps nested encodings looking for hidden data, is the most
exposed parser in the product and the obvious fuzz target. It was fuzzed directly.

| Measure | Result |
|---|---|
| Iterations | 200,000 |
| Input generators | 11 |
| Elapsed | 303.7 s |
| Crashes | 0 |
| Contract violations | 0 |
| Unbounded or hung runs | 0 |
| Slowest single call | 65.06 ms |
| Peak memory | 565 MB |

The eleven generators were built to provoke different failure modes rather than pile up volume:
random noise across seven alphabets including control characters and multi-byte text, valid
encodings, base64 nested twelve levels deep, mixed nesting of compression and hex and URL
encoding, truncated and corrupted encodings, compressed data with valid headers and deliberately
damaged bodies, deeply nested structured documents with payloads buried inside, single tokens up
to 200,000 characters, thousands of decode candidates in one input, compression bombs expanding to
two megabytes, and hostile non-string inputs including null, NaN, functions, and symbols.

Nothing produced a crash, an unbounded run, or a return value that broke the component's contract.
The bounded-decoding design held throughout. Slowest single call was 65 ms against a deliberate
compression bomb, and memory stayed bounded.

The run is seeded and deterministic, so anything it had found would be reproducible from its seed.

What this doesn't prove: fuzzing shows the absence of the failures it provoked, not the absence of
failures. It's evidence of robustness against malformed and hostile input. It isn't a security
audit and doesn't replace one.

## Release gate

The product was developed behind its own acceptance gate, five checks per component: tests pass,
no stub markers left, zero external dependencies, required files present, no decorative characters
in shipped text. All twelve components passed all five.

The gate script is a development tool and isn't part of the shipped product, so unlike everything
else here this one can't be reproduced from a delivered copy. It's history rather than a
reproducible result. The properties it checked are verifiable on their own: the test results, the
dependency count, and the component inventory are all above.

## Scale

| Measure | Value |
|---|---|
| Code files | 261 |
| Source lines | 21,740 |
| Test lines | 10,387 |
| Test-to-source ratio | 0.48 |
| Components | 11 modules plus the runtime |
| Command-line interfaces | 12 |
| MCP servers | 12 |
| Runnable examples | 24 |
| Internal imports, all resolving | 440 |

Line counts are raw lines of source files, blanks and comments included, with test directories
counted separately from the rest. The method is stated because a line count means nothing without
one.

## Attack surface

| Property | Value |
|---|---|
| Network | none, no sockets, no outbound calls |
| External dependencies | 0, so no supply chain |
| Platform facilities used | 8 standard Node.js built-ins |
| Credentials | none, no API keys, tokens, or accounts |
| Dynamic code execution | none |
| Filesystem writes | only paths you hand it |

## Supported platform

Conductor is a Windows product. Built, verified, and sold for Windows.

| Property | Value |
|---|---|
| Supported platform | Windows |
| Verified on | Windows 11, Node.js 24 |
| Runtime required | Node.js 20 or newer |
| macOS | Not supported. Not tested. |
| Linux | Not supported. Not tested. |

That's scope, not a limitation found late. Every result here was produced on Windows, and Windows
is what's on offer.

The code has no hard-coded absolute paths and uses only platform-independent facilities, so it
would probably run elsewhere. Probably isn't verified, and nothing here rests on it. Support for
another platform is a conversation, not something to assume from outside.

## Release

| Field | Value |
|---|---|
| Product release | 1.0.1 |
| Component versions | Anchor 1.0.1, the component that changed; all others 1.0.0, code untouched |
| Publisher | Devadex Labs |
| Copyright holder | Jesse Duncan, an individual trading as Devadex Labs |
| Licence | Proprietary. Not open source. |

---

## Open items

A certification listing only passes isn't a certification. These are the known gaps, stated here
rather than left to be found.

| Severity | Item |
|---|---|
| MEDIUM | No third-party security audit. The security-critical modules have been checked internally but never reviewed by an independent firm. A licensee should expect to commission one. |
| MEDIUM | Anchor's cost grows quadratically with run length. Cut 2.5x by optimisation, with behavior verified bit-identical, but the growth curve is unchanged. Removing it would narrow what run-level analysis looks at, changing recorded output on very long runs. Numbers above. |
| MEDIUM | A runtime built with no gate modules allows every action. Only reachable through misconfiguration, and the integration instructions are explicit, but a safety product should warn or refuse to start rather than run quietly permissive. |
| MEDIUM | Signal and audit-reason identifiers aren't stable across runs. Behavior and verdicts are stable and verified, the identifiers attached to them aren't. |
| LOW | Branch coverage trails line coverage, 78.20% mean against 91.82%. One file in the composition layer sits at 55.56% branch, the lowest point. |
| LOW | No demonstration recordings have been made. |

Closed since the previous record: coverage measurement, performance benchmarking, and fuzzing of
the decode scanner have all been done, and the results are above. Platform scope is now stated
rather than listed as a gap.

## Deployment

Conductor is in production use inside StudioOS, a Devadex Labs product, where it's the reliability
layer under that product's agent functionality. It's working software carrying real work, not a
prototype.

It hasn't been deployed by anyone outside Devadex Labs, and no third party has ever received a
copy in source or any other form.

## Ownership

| Property | Status |
|---|---|
| Copyright holder | Jesse Duncan, sole author. Single unbroken chain of title. |
| Third-party code | None. Zero dependencies means no other licence attaches. |
| Prior licences granted | None |
| Copies held by any third party | None |
| Encumbrances | None |
| Available for exclusive transfer | Yes |

Conductor has never been sold, licensed, or distributed. For a technology asset that's an
advantage rather than a gap. The rights are whole and sit with one identifiable owner, there are
no existing licensees whose terms would survive a transfer, no copies out in the world to reclaim,
and nothing to disclose to a counterparty. An exclusive transfer is available precisely because no
prior grant has split it up.

Provenance and chain-of-title documentation are available under a signed agreement.
