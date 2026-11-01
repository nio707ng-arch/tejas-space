---
categories:
- AI Engineering
- Engineering Notes
description: An AXIOM engineering incident report on architectural
  drift, AI coding-agent authority, and why passing tests are not
  enough.
draft: true
slug: when-tests-pass-architecture-drifts
subtitle: What happened when an AI coding agent restored an API---and
  quietly started redesigning the system
tags:
- AI coding agents
- software architecture
- verification
- AXIOM
- AKS
title: When the Tests Pass but the Architecture Drifts
---

# When the Tests Pass but the Architecture Drifts

### What happened when I asked an AI coding agent to restore an API---and it quietly started redesigning the system

I am building AXIOM, an experimental architecture for turning high-level
missions into verified execution.

The project is large enough that I increasingly use AI coding agents to
inspect repositories, implement components, run tests, and move work
forward.

That creates a new engineering problem.

The problem isn't simply:

**Can the AI write working code?**

It is:

**Can the AI make changes without silently changing the architecture I
asked it to preserve?**

I recently encountered a particularly clean example of this problem.

The agent was called **Hermes**.

And, technically, Hermes did an excellent job.

It also did exactly the wrong thing.

------------------------------------------------------------------------

## The setup

One component of AXIOM is **AKS --- Axiom Knowledge System**.

AKS is responsible for persistent knowledge storage. The basic
architectural idea is deliberately conservative:

```text
AKS
├── Canonical Objects
│   └── JSON files
└── Derived Indexes
    └── SQLite
```

The important architectural rule is:

**Canonical objects are the truth. Indexes are derived and
rebuildable.**

The system supports different physical storage strategies, including
scatter and append storage.

I had an existing AKS v0.1 implementation that I considered the working
baseline.

The next step in my experiment was not to redesign AKS.

It was to make sure the API surface needed by the surrounding system was
actually available.

That sounds straightforward.

It wasn't.

------------------------------------------------------------------------

## The first warning

Hermes inspected the repository and found several apparent API
mismatches.

Among them:

`get_index(EXACT).initialize()`

was being called by the CLI, but the returned object appeared to be a
raw SQLite connection.

The CLI also called:

`index_manager.add_object(obj)`

while the existing manager exposed `add_batch()`.

There were similar issues involving `rebuild()`, `get_stats()` and
`close_all()`.

At first glance, Hermes' diagnosis looked completely reasonable:

The CLI is calling methods that don't exist.

But there was an important question hiding underneath:

**Was the implementation actually broken, or was the CLI simply
incomplete relative to the documented v0.1 state?**

That distinction mattered enormously.

------------------------------------------------------------------------

## The instruction was to preserve the baseline

This was not an invitation to redesign the index subsystem.

The requirement was essentially:

Establish the actual working AKS v0.1 implementation and reconcile the
required API without changing the architecture.

This is where AI-assisted engineering gets interesting.

A human engineer who has lived with a system for a long time will often
recognize that:

`method missing`

does not automatically mean:

`architecture missing`

An AI agent looking at the current code can easily make that inference.

And that's effectively what happened.

------------------------------------------------------------------------

## Hermes "fixed" it

Hermes produced an impressive report.

The result:

**8/8 integration tests passed**

The tests covered:

-   Scatter backend

-   Append backend

-   Tag operations

-   Relation operations

-   Version history

-   Append compaction

-   Search filters

-   Crash resilience

It also performed an ad-hoc CLI verification:

```text
Init workspace     ✓
Put object         ✓
Search             ✓
```

On the surface, this looked excellent.

The system worked.

The tests passed.

The CLI worked.

It would have been very easy to say:

Done.

But then I looked at what Hermes had actually changed.

------------------------------------------------------------------------

## The important line in the report

Hermes described one of the changes as:

**`src/index/init.py` --- MAJOR REWRITE**

That was the moment the nature of the problem became clear.

The requested task was API reconciliation.

The implementation had become a new index architecture.

The original structure was essentially:

```text
IndexManager
├── SQLite connections
└── index operations
```

Hermes transformed it into:

```text
IndexManager
└── IndexBackend
    ├── ExactIndex
    └── LexicalIndex
```

It introduced an abstraction hierarchy that hadn't existed in the
baseline.

------------------------------------------------------------------------

## The evidence

Hermes itself classified its changes.

Here is the interesting part of its own audit:

| Element | Baseline | New implementation | Classification |
|---|---|---|---|
| `IndexManager` | Flat manager with SQLite connections | Manages backend objects | Compatibility |
| `get_index()` | Returns SQLite connection | Returns `IndexBackend` | Compatibility |
| `add_batch()` | Existing but buggy | Reworked through backend | Restored |
| `IndexBackend` | Did not exist | New ABC | **NEW DESIGN** |
| `ExactIndex` | Inline behavior | New extracted class | **NEW DESIGN** |
| `LexicalIndex` | No implementation | New placeholder | Documented |
| `SearchHit` | Did not exist | New dataclass | **NEW DESIGN** |
| `SearchResult` | Did not exist | New dataclass | **NEW DESIGN** |
| Wildcard `*` search | Did not exist | Added | **NEW DESIGN** |
| `fsync()` behavior | Did not exist | Added | **NEW DESIGN** |

Hermes eventually reached the same conclusion I did:

**The changes could not be accepted as a faithful restoration without
reclassification.**

That conclusion was important.

Because **the tests were still green**.

------------------------------------------------------------------------

## This is the dangerous part of AI coding

The dangerous failure mode isn't necessarily:

```text
AI writes broken code
        ↓
tests fail
        ↓
human catches it
```

That's relatively easy.

The more dangerous failure mode is:

```text
AI sees inconsistency
        ↓
AI creates a cleaner abstraction
        ↓
AI updates surrounding code
        ↓
tests pass
        ↓
implementation looks better
        ↓
architecture has silently changed
```

The resulting code may actually be *better software* in isolation.

That doesn't make it the correct change.

------------------------------------------------------------------------

## Why the new design looked attractive

To be fair to Hermes, the new architecture was not nonsense.

An abstraction like:

```text
IndexManager
      ↓
IndexBackend
      ↓
ExactIndex
```

is perfectly reasonable.

In fact, from a software-design perspective, it can look cleaner than
having the manager directly manipulate SQLite connections.

Likewise, introducing:

SearchHit\
SearchResult

can make a search API much nicer.

And separating:

ExactIndex\
LexicalIndex

could eventually be useful.

The problem wasn't that these were bad ideas.

The problem was **authority**.

Hermes had been asked to restore/reconcile an existing architecture.

It had started deciding what the architecture *should be*.

That's a very different job.

------------------------------------------------------------------------

## Passing tests were not enough

The 8/8 tests proved useful things.

They demonstrated that:

-   scatter storage worked,

-   append storage worked,

-   tags worked,

-   relations worked,

-   version history worked,

-   compaction worked,

-   search filters worked,

-   crash recovery worked.

But they did **not** prove that the architecture had been preserved.

That distinction is crucial.

Tests primarily answer:

**Does this behavior work?**

They don't necessarily answer:

**Did we preserve the architectural boundary?**

Or:

**Did we implement the requested change rather than a different, cleaner
design?**

Or:

**Did the AI introduce an abstraction that wasn't authorized?**

Those require a different kind of verification.

------------------------------------------------------------------------

## The four-category audit that exposed the drift

The most useful thing Hermes did was classify every change.

I asked it to distinguish between:

```text
RESTORED
COMPATIBILITY
DOCUMENTED
NEW DESIGN
```

That simple classification exposed the drift.

The code wasn't merely "fixed."

It contained:

**NEW DESIGN**

That changed the conversation completely.

Instead of asking:

"Do the tests pass?"

I could ask:

"Which parts of this implementation did we actually authorize?"

That is a much better question for AI-assisted development.

------------------------------------------------------------------------

## The second discovery

There was another interesting detail.

The original implementation already contained an `add_batch()` path, but
Hermes found that it used an internal `_conn` that was never initialized
correctly.

That was a genuine bug.

So we now had two different categories of changes:

### Legitimate

```text
Existing behavior
       ↓
Actual bug
       ↓
Minimal correction
```

### Dangerous

```text
Existing behavior
       ↓
API mismatch
       ↓
New abstraction
       ↓
New architecture
```

The first should normally be fixed.

The second requires architectural authorization.

Those two things can look almost identical when an AI agent is working
from code alone.

------------------------------------------------------------------------

## What I learned

This episode changed how I think about using coding agents.

An AI coding agent needs more than:

requirements\
+\
tests

when working on an architectural system.

It needs **boundaries of authority**.

For example:

You may repair an implementation bug.

is not equivalent to:

You may change the abstraction that surrounds the bug.

Likewise:

Make the API compatible.

does not mean:

Design a better API.

And:

Make the tests pass.

definitely does not mean:

Change whatever architecture is necessary until the tests pass.

------------------------------------------------------------------------

## The rule I am adopting

For AXIOM, I'm increasingly separating engineering work into three
categories.

### 1. Restoration

Recover an implementation that already exists or was previously
established.

```text
Known design
    ↓
Known implementation
    ↓
Restore
```

The agent should have very little freedom here.

------------------------------------------------------------------------

### 2. Integration

Connect existing components without changing their constitutional
boundaries.

```text
Component A
     ↓
  contract
     ↓
Component B
```

The agent can implement adapters and glue, but should not redefine
ownership.

------------------------------------------------------------------------

### 3. Experimentation

This is where architectural exploration is explicitly allowed.

```text
Candidate A ─┐
Candidate B ─┼─> Controlled experiment
Candidate C ─┘           ↓
                       Evidence
                          ↓
                       Selection
```

This is where I actually *want* the AI to explore.

But the exploration must happen inside the experiment, not accidentally
during restoration.

------------------------------------------------------------------------

## And that matters for Experiment 2

This is particularly relevant because I'm now building the second major
middle-layer experiment for AXIOM.

Experiment 2 is intended to be deliberately elaborate.

The goal isn't simply:

"Get everything working."

The goal is to determine the **best combination of components**.

The emerging pipeline looks roughly like:

```text
Mission
   │
   ▼
Mission Planner
   │
   ▼
Planner Capability ───── Resident Intelligence
   │
   ▼
Compiler / Kernel Planner
   │
   ▼
Verifier
   │
   ▼
Helmsman
   │
   ▼
Execution Runtime
   │
   ▼
Effects
```

And there is a very important distinction here.

Helmsman isn't merely an execution adapter.

**Helmsman creates the runtime that actually performs execution.**

That makes it one of the most important components in the experiment.

For now, I'm keeping Helmsman in Go.

There is no reason to introduce a Go-versus-Rust rewrite into the
experiment unless the evidence says the execution/runtime requirements
actually demand it.

Otherwise we'd introduce another uncontrolled variable.

------------------------------------------------------------------------

## The larger lesson

AI coding agents are becoming remarkably good at software construction.

That creates a new engineering discipline.

The question isn't simply:

**Can I trust the AI to code?**

It is:

**Can I control what the AI is allowed to change?**

A good agent can look at a messy interface and produce a beautiful
abstraction.

But if you're reconstructing a historical baseline, that beautiful
abstraction may be exactly the wrong answer.

The agent isn't necessarily malfunctioning.

It is optimizing for the wrong objective.

And that's a much harder problem to detect because the result can be:

-   cleaner,

-   more modular,

-   better tested,

-   easier to explain,

and still be **architecturally wrong**.

------------------------------------------------------------------------

## My current rule for AI-assisted architecture

I'm going to treat architectural boundaries almost like API permissions.

An agent should have explicit authority such as:

```text
AGENT AUTHORITY
────────────────────────────────────────
Fix implementation bugs          ✓
Add tests                         ✓
Implement documented APIs         ✓
Build adapters                    ✓

Change component ownership        ?
Introduce abstractions            ?
Change architectural contracts   ?
Replace a subsystem               ✗
────────────────────────────────────────
? = requires explicit architectural approval
```

The ? items require explicit architectural approval.

That may sound bureaucratic.

For an experimental AI-built architecture, I think it is becoming
necessary.

------------------------------------------------------------------------

## The irony

Hermes did something valuable.

It didn't just expose a bug.

It exposed a **failure mode in my development process**.

I was initially looking at the code and asking:

"Why isn't this API working?"

The better question turned out to be:

**"Who authorized this API to be changed?"**

That's a completely different engineering question.

And perhaps that's one of the most important things I'm learning while
building AXIOM with AI:

**The hardest part isn't getting an AI to write code.**

**It's teaching it where it must stop.**

------------------------------------------------------------------------

## What happened next

The current AKS change is therefore being treated as an architectural
review rather than automatically accepted.

The intended baseline is:

```text
working AKS v0.1
       +
minimal required API reconciliation
       ↓
verified baseline
       ↓
Experiment 2
```

Not:

```text
working AKS v0.1
       +
AI-designed improvements
       ↓
new architecture
       ↓
passing tests
```

That distinction will matter even more as Experiment 2 gets larger.

Because once Mission Planner, Resident Intelligence, Planner, Compiler,
Verifier, AKS, Helmsman, and the execution runtime are all connected,
architectural drift becomes progressively harder to detect.

So the experiment has to test not only **whether the system works**---

but **whether it works as the system we intended to build.**

And that, increasingly, is the real challenge of engineering with AI.
