# Case Study: AI-Assisted Root Cause Analysis of Shared-Cache Contamination

*How a months-long flaky-test problem was diagnosed by treating an LLM as a reasoning accelerant — and treating its output as something to validate, not trust.*

---

## Summary

A suite of backend end-to-end tests passed reliably in isolation but failed intermittently in parallel, producing measurable flakiness across the team's suites and frequently failing the shared CI/CD gate for months. The root cause was not in the test code. It was a shared mock-cache resource being contaminated by concurrent test runs against a lower environment.

The investigation is worth documenting not because the defect was exotic — it was a classic resource-cleanup bug — but because of *how* it was found: by using an LLM as a structured RCA accelerant while keeping a human firmly in the validation loop. The model produced fast, confident hypotheses. Some were correct. Some over-asserted certainty. The diagnostic value came from supplying the model the domain ground truth it lacked, narrowing its questions until it could only answer from evidence, and rejecting the parts of its output that outran the data.

That loop — generate with AI, validate against ground truth, drive to an upstream fix — is the practice this case study is really about.

---

## Context

- **System under test:** a multi-marketplace ticket aggregation platform composed of multiple microservices.
- **Test type:** backend-only end-to-end. No browser. Each test exercised several services and wrote to multiple database tables.
- **The cache layer:** a Redis instance used to *mock* outbound third-party HTTP API responses, so tests could run without hitting live external marketplaces.
- **Where it ran:** a shared lower (UAT) environment, executed in parallel by the whole team's CI/CD pipeline.

The shared, mutable cache plus concurrent execution is the entire setup for the failure. Shared mutable state under concurrency is the highest-prior cause of nondeterminism, and that is where the investigation started.

---

## Symptom

A test would pass when run alone and fail when run alongside others — sometimes, not always. The failure surfaced as a downstream service returning null/empty content where valid data was expected, which the test correctly flagged as a failure.

Because the failure was intermittent and only appeared under parallelism, the standard reflexes — retries, added waits, pipeline reruns — were applied along the way. Like all such mitigations, they masked or deferred the symptom without touching the shared mutable state at the cache layer that actually caused it. Eliminating the root cause meant going past those stopgaps, not relying on them.

---

## Investigation

**1. Frame the hypothesis from systems thinking.** A test that only fails in parallel must be interacting with something that changes between isolated and concurrent runs. That points at shared state. The shared, mutable Redis mock cache was the obvious first candidate.

**2. Pull the high-cardinality data.** Rather than reasoning from "the test failed," the actual cache state at failure time was captured from the Redis profiler. This is the wide-event substrate the rest of the analysis stands on — what was *in* the cache, keyed how, at the moment things broke.

**3. Use the LLM as a scoped accelerant — and validate its output.** The profiler logs were handed to an LLM with a deliberately narrow question: *what is in this cache that should not be?* — not the generic "what's wrong?" Narrowing the prompt forces the model to answer from the evidence in front of it rather than from plausible-sounding priors.

This is the part worth being honest about, because it is the part that generalizes. An LLM does not proactively ask the questions that would expand its own working knowledge of the problem domain — it answers what it is given. So the domain context had to be supplied to it: that this Redis layer was a *shared mock cache* hammered by parallel runs, and — critically — that Redis Insight was a second observability layer whose profiler output correlated log entries to individual test executions. Recognizing that Redis Insight was the right instrument, and knowing how to read it well enough to tie specific log lines to single test runs, was human knowledge the model had no awareness of and could not have requested.

Once given that context and those logs, the model was genuinely useful at the thing it is good at: parsing giant, densely packed log lines into human-readable form and locating the exact strings that marked the point of corruption — repeatedly, across multiple test runs. That is real acceleration. But it also produced confident assertions, including over-stated ones ("definitively infrastructure, not test code"). Some of that confidence was warranted by the logs; some of it was the model optimizing for a decisive-sounding answer. Separating the two was a human judgment anchored to the cache state, not a model output to be accepted at face value.

**4. Read the mechanism out of the cache.** The logs showed a single cache key — a `GET` against a marketplace venue-search endpoint — holding valid response data under one request's tracking identifier, then being overwritten with an **empty array** by a concurrent test using the *same* key. The next read returned empty, the dependent service produced a null/empty-content failure, and the test failed. Same key, different concurrent writers, last write wins — and the last write was empty.

---

## Root Cause

A message handler consumed its message but never cleaned up the corresponding cache entry. Stale entries — including empty ones — persisted under a 60-minute TTL and poisoned subsequent runs that collided on the same key. The cache keys carried no per-test isolation, so concurrent tests competed for the same slots.

This was a resource-cleanup defect in the handler, invisible without the cache state at failure time. The test code was correctly detecting a real data-integrity problem; it was not the cause of it.

---

## The Fix

- **Source fix:** corrected the handler so it cleans up its cache entry rather than leaving stale/empty data behind. The defect was fixed where it lived, not papered over at the test layer.
- **Test-side defense:** introduced per-test isolation in the cache keys (a run-scoped identifier) and domain-specific pre-scenario cleanup of the relevant key namespaces, so a contaminated slot from one run could not bleed into the next.

---

## Result

After the source fix and key isolation, this contamination failure mode stopped recurring. The intermittent parallel failures that had persisted for months were eliminated — not suppressed behind retries — and the fix held across the microservices teams sharing that environment, because it removed the shared-state defect itself rather than working around it in one suite. The investigation also produced a reusable internal RCA reference so the same workflow could be run by the rest of the team.

---

## What Generalizes — The Quality Strategy

The specific defect is narrow. The method is not.

Every shared resource the system under test touches — database rows, cache keys, message-queue state, file system, object storage, external rate limits — is a candidate contamination source under parallelism. The repeatable loop is:

**hypothesis → high-cardinality data at failure time → scoped, AI-accelerated analysis with human validation → root cause → upstream fix.**

What does not appear as the *fix* in that loop: retries, sleeps, and pipeline reruns. Those are stopgaps that keep a gate green; applied here, they masked the symptom without eliminating the defect. Going past them is what converts a flaky-test *triage* activity into a flaky-test *elimination* activity — a software-engineering fix that happens to begin in the test layer.

## Why this belongs in an AI-native quality practice

The increasingly common requirement is the ability to *validate AI-generated engineering output and improve quality strategy at a senior level.* This investigation is a concrete instance of exactly that. An LLM generated fast, partially-correct, sometimes over-confident analysis. The senior contribution was not the prompting — it was supplying the domain ground truth the model lacked, introducing the observability instrument the model had no awareness of (Redis Insight) and reading it well enough to correlate log lines to individual test runs, constraining the model's questions to the evidence, rejecting the assertions that outran the data, and carrying the result to an upstream code fix and a generalized cleanup pattern adopted across teams.

The model accelerated the work. It did not own the conclusion. That distinction — AI as accelerant, human as the verification layer that anchors output to ground truth — is the quality discipline, not an aside to it.
