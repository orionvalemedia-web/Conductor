# Conductor

The missing middleware layer under an agent stack.

Conductor sits between an AI agent and everything it touches. Every time the agent crosses a
boundary, Conductor watches it, decides whether to allow it, records it, and can replay it later.

This repo is documentation only. It explains what Conductor is, how it's built, what it enforces,
and how it's been tested. There's no source code here. Conductor is proprietary and not open
source. See [LICENSING.md](LICENSING.md).

---

## The problem

Strip an agent down and it does two things: it calls a model, and it calls a tool.

Almost everything that goes wrong in production happens at one of those two moments:

- It does something nobody asked for
- It states something it made up, then acts on it
- It uses a fact that used to be true
- It slowly wanders off the job it was given
- A tool hands back garbage and it keeps going
- A secret leaks out in something it sends
- It does something permanent with no way to undo it
- It's holding a key far more powerful than the task needed

A smarter model doesn't fix any of this. The reasoning isn't the problem, what the reasoning is
allowed to *do* is. That's an infrastructure problem, and most agent stacks have nothing sitting
in that gap.

## What Conductor does

It wraps both of those moments.

Everything crossing them becomes an event the modules can act on. Some modules watch and take
notes. Some judge an action before it runs and can stop it. The runtime writes down the whole run
so you can replay it, not just read logs about it.

Conductor assumes the agent can't be trusted. The model might be manipulated, confused, or just
wrong, so the checks live outside it rather than in a prompt asking it to behave.

## Eleven modules, one shared contract

Conductor runs eleven modules over a live agent loop. Each one handles a single failure, and each
works on its own.

| Module | Role | What it stops |
|---|---|---|
| Compass | Gate | Doing things outside the job it was given |
| Warden | Gate | Policy breaks, runaway spend, injected instructions reaching a tool |
| Ledger | Gate + Observer | Acting on claims nothing backs up |
| Revert | Gate + Observer | Permanent actions with no way back |
| Airlock | Gate + Observer | Secrets going out the door |
| Keyring | Gate + Observer | Long-lived keys that are too powerful |
| Sentinel | Observer | Broken or poisoned tool output getting reasoned over |
| Timekeeper | Observer | Using facts that have expired |
| Anchor | Observer | Drifting off the objective over a whole run |
| Recall | Observer | Losing what it already worked out |
| Rewind | Runtime | Runs you can't reproduce |

All eleven talk to one small shared contract. No module knows about any other module. That means
adding one is wiring rather than a rewrite, you can pull one out without touching the rest, and a
module that breaks takes only itself down.

More in [MODULES.md](MODULES.md) and [RUNTIME.md](RUNTIME.md).

## What's been measured

| Measure | Value |
|---|---|
| Modules | 11 |
| Shared contract | 1 |
| Tests | 918 passing, 0 failing, 0 skipped |
| Test coverage | 91.82% line, 78.20% branch, 91.14% function |
| Fuzzing | 200,000 cases against the decode scanner, 0 failures |
| Gate overhead | 0.035 ms per iteration, all six gates |
| External dependencies | 0 |
| Build step | none |
| Network access | not needed |
| Platform | Windows |

Zero dependencies means there's no supply chain to audit and nothing upstream that can break it.
A version you freeze today still works in five years.

See [CERTIFICATION.md](CERTIFICATION.md) and [TESTING.md](TESTING.md).

## It runs offline

No sockets, no outbound calls, no API keys, no accounts. Everything happens on the machine it's
installed on. That's what makes it usable in air-gapped and regulated setups where you can't ship
agent traffic off to somebody else's service.

## Three ways to use it

**As a library.** Wrap an agent loop you already have. It keeps its own model and its own tools,
and Conductor goes around them.

**From the command line.** Run, record, replay, and diff runs without writing any integration
first. Quickest way to try it against real work.

**As an MCP server.** Conductor speaks the Model Context Protocol, so an MCP host can use it with
no custom glue.

## Docs

| Document | What's in it |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | The two boundaries, and watch / judge / record / replay |
| [MODULES.md](MODULES.md) | All eleven modules and the failure each one handles |
| [RUNTIME.md](RUNTIME.md) | How they compose, stay isolated, and stay deterministic |
| [TESTING.md](TESTING.md) | The 918 tests and three demos you can reproduce |
| [CERTIFICATION.md](CERTIFICATION.md) | Measured numbers, and the known gaps |
| [SECURITY.md](SECURITY.md) | What it defends against and how it fails |
| [LICENSING.md](LICENSING.md) | Proprietary status, and what this repo is |
| [ACQUISITION.md](ACQUISITION.md) | Licensing, partnership, and technology discussions |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Why code contributions aren't taken |

---

Developed by Jesse Duncan, founder of Devadex Labs. Conductor is available for licensing,
partnership, and strategic technology discussions.

Proprietary; all rights reserved.
