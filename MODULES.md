# Modules

Eleven modules. Each one handles a single failure, and each one works on its own.

This covers what they do and why. It doesn't cover how any of them work inside.

---

## Roles

| Role | Meaning |
|---|---|
| Gate | Judges an action before it runs. Can stop it. |
| Observer | Looks at what already happened and emits Signals. Can't stop anything. |
| Gate + Observer | Both. Watches the run to build up what it later judges with. |
| Runtime | Hooks the runtime itself rather than taking part in the two phases. |

That split matters. Gates are the only things that can stop an action, so they're the only things
whose failure could leave a hole, which is why a gate that breaks counts as a deny. Observers
can't stop anything, so you can turn them all on without worrying.

## The list

| Module | Role | Handles |
|---|---|---|
| [Compass](#compass) | Gate | Doing things outside the job it was given |
| [Warden](#warden) | Gate | Policy breaks, runaway spend, injected instructions |
| [Ledger](#ledger) | Gate + Observer | Acting on claims nothing backs up |
| [Revert](#revert) | Gate + Observer | Permanent actions with no way back |
| [Airlock](#airlock) | Gate + Observer | Secrets going out the door |
| [Keyring](#keyring) | Gate + Observer | Long-lived keys that are too powerful |
| [Sentinel](#sentinel) | Observer | Broken or poisoned tool output |
| [Timekeeper](#timekeeper) | Observer | Facts that have expired |
| [Anchor](#anchor) | Observer | Drifting off the objective |
| [Recall](#recall) | Observer | Losing context across a run |
| [Rewind](#rewind) | Runtime | Runs you can't reproduce |

---

## Compass

Role: Gate

Checks every action against the job the agent was actually given.

The failure: an agent doing things *near* the job instead of the job. You ask it to research a
market and it deletes a file to tidy up. You ask it to draft a summary and it sends it. Every step
looks reasonable on its own and none of them were asked for. Telling it "stay on task" in the
prompt doesn't help, because it isn't disobeying, it's interpreting.

Compass holds the mission as something enforceable instead of a suggestion, and checks each action
against it. Anything off-mission gets denied before it runs, with the reason attached.

It's what makes the mission an actual boundary. It answers "was this even part of the job", which
has to be settled before any of the more specific questions matter.

---

## Warden

Role: Gate

The policy, budget, and threat check between the agent and every tool.

Three different failures that all show up at the same place. An action breaks an operational rule.
A run burns through way more than it was supposed to. Or some text that came in through the model
or a tool result contains an instruction, and that instruction reaches something with real
consequences.

Warden applies your policy rules, tracks spend for the run, and screens tool arguments for threats.
Any of the three can deny.

Where the other gates each defend one specific thing, Warden carries whatever rules you write for
yourself, plus the prompt-injection check that a security team is going to ask about first.

---

## Ledger

Role: Gate + Observer

Checks whether a claim is actually backed by anything.

The failure: the model states something with total confidence and no basis, and the agent acts on
it. This is the one that burns trust fastest, because the output reads perfectly. Nothing looks
wrong until the action built on it lands.

Ledger tracks which claims in a run are grounded and which aren't, and blocks actions resting on
ungrounded ones. Once the claim gets sourced, the same action goes through.

It's the difference between "the agent said so" and "this is established". Confidence stops being
enough on its own, which is the only real answer to a model making things up.

---

## Revert

Role: Gate + Observer

Keeps track of what can be undone, and gates what can't.

Not all mistakes are equal. A wrong sentence you can fix. A deleted record, a sent message, or
moved money you can't. Systems that treat every action as equally recoverable are wrong in exactly
the cases that hurt.

Revert journals actions as they happen along with what it'd take to undo each one, and flags or
gates the ones with no safe undo.

It puts a line between reversible and permanent, so you can hold permanent things to a higher bar
without slowing down everything else.

---

## Airlock

Role: Gate + Observer

Watches what goes out, so secrets don't go with it.

The obvious version is a secret in plain text, and any filter catches that. The version that beats
most controls is encoded: wrapped, compressed, or nested until pattern matching doesn't recognize
it anymore.

Airlock fingerprints sensitive values where they come in, follows them as they move around, and
inspects anything about to leave. Encoded and nested content gets decoded and rescanned rather
than taken at face value. Canary tokens get caught at every exit, including inside decoded layers.
Anything carrying tainted data gets blocked, and it reports the path it followed to find it.

Decoding is bounded, so the scanner can't be turned into a denial-of-service target against
itself.

Airlock is the last checkpoint before data is gone. It's also the one that assumes an attacker
rather than an accident, which is the right assumption at an exit.

---

## Keyring

Role: Gate + Observer

Hands out short-lived, narrowly scoped credentials.

The failure: an agent gets a long-lived key far broader than any single task needs, because
setting up one key per task is a pain. From then on, every other mistake is amplified by whatever
that key can reach.

Keyring issues credentials just in time, scoped to the task, and expiring. The agent never holds
standing access, it holds a lease that's small and short by construction.

This is what limits the damage from everything else. It's the difference between a mistake that
touches one record and one that touches the whole account, and it applies whether the cause was an
attack or an ordinary bug.

---

## Sentinel

Role: Observer

Checks what tools hand back.

The failure: a tool returns something malformed, truncated, empty, or hostile, and the agent
reasons over it like it's fine. The agent has no way to tell a good result from a bad one, it'll
use whatever it gets, and the corruption spreads through every step after that.

Sentinel validates tool output against what that tool is supposed to return and quarantines
anything that fails, so the problem is visible and contained instead of absorbed.

The gates protect the world from the agent. Sentinel protects the agent from the world, which is
the direction most stacks leave wide open.

---

## Timekeeper

Role: Observer

Knows when a fact has gone stale.

The failure: something that was true isn't anymore. A price changed, a policy got replaced, a
person moved roles. The agent has no sense of when it learned something or whether that's still
good, so old facts get used with the same confidence as fresh ones. And when both the old and new
version are floating around, nothing decides which one wins.

Timekeeper tracks how long facts stay valid and what supersedes what, and flags both expired facts
and contradictions between them.

It gives the run a sense of *when*. Ledger asks whether a claim is backed. Timekeeper asks whether
that backing still holds.

---

## Anchor

Role: Observer

Watches the shape of the whole run.

The failure: no single step is wrong, but the run has wandered. A series of small, reasonable
substitutions ends up somewhere the original objective never pointed. No per-action check can see
this, Compass included, because each individual step passes fine. The drift only exists in the
sequence.

Anchor analyzes the run's trajectory against its objective and reports drift, including what kind
and how bad.

It's the only module that looks at the run rather than the action, so it catches the thing
per-action gates structurally can't.

---

## Recall

Role: Observer

Remembers things, locally.

The failure: agents forget. Context runs out, runs end, and whatever was worked out earlier has to
be redone or is just lost. The usual fix is a hosted vector service, which moves the problem into
somebody else's infrastructure, and plenty of environments can't do that.

Recall stores and retrieves memories, both episodic and semantic, entirely on the local machine.
No cloud service, no network call, no external index.

It's what lets a run build on earlier work while keeping the whole layer offline. A memory module
that needed a network call would blow that property for everything else.

---

## Rewind

Role: Runtime

Records the run so you can run it again.

The failure: the agent did something and nobody can say exactly why. Logs describe what happened,
they don't let you re-run it. Without reproduction there's no real debugging, no real incident
review, and nothing that holds up in front of an auditor.

Rewind writes every crossing into one recording as the run goes. You can replay and diff it, and
it reproduces identically: same events, same signals, same verdicts, same order.

Because Rewind hooks the runtime rather than sitting in the two phases, it captures what every
other module did, including the decisions and the reasons they gave.

It turns a run from something that happened into something you can examine. It's what makes the
other ten auditable rather than just active.

---

## Why eleven and not one

You could call all of this one product with eleven features. It's kept separate for three reasons.

The failures are genuinely different. Drift isn't staleness, staleness isn't fabrication,
fabrication isn't exfiltration. Mash them together and you get something vague about all of them.

Independence is a real property. No module depends on another, so you can change or remove one
without touching the rest, and one that breaks is isolated instead of fatal.

You can adopt it gradually. Nobody has to accept all eleven judgments on day one. Turn them on as
you come to trust them, which is how safety infrastructure actually gets into production.

See [RUNTIME.md](RUNTIME.md) for how the eleven fit together.
