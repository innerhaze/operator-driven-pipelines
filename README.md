# Operator-driven pipelines

Design notes on building a pipeline that is operated day to day by someone who knows the domain but is not an engineer. Not every pipeline should run on cron. For a lot of real work, the right primary user is a human who understands what the output should look like and will tell the pipeline when to start.

The notes below describe the design choices that followed from committing to that stance.

## Two stages: preflight, then execution

The pipeline has two separable stages that the operator runs in order.

**Preflight** validates the ingredients: the input artefact exists and parses, external sources respond, the right accounts are reachable, the schema the pipeline is going to write to is the one it was built against. It does not change any shared state. Its only output is a report that lists what is ready and what is not, with a short description of each missing thing and what action the operator would take to resolve it. "The external source for record set X returned 503 twice" is a sentence an operator can act on; "AssertionError at line 213" is not.

The preflight report is the contract. If it passes, the operator can start the execution stage with reasonable confidence. If it does not, the operator fixes the items on the report and reruns preflight. The fix step is explicitly manual; the pipeline does not try to repair the inputs on the operator's behalf.

**Execution** runs end to end without supervision once preflight is clean. It does not stop in the middle to ask questions. It writes a final report when it finishes.

The two-stage split exists because interleaving validation with execution means the operator has to be present when the slow part runs, which is the opposite of what the operator-driven stance is for.

## Never interrupt mid-run

Within execution, the hard rule is that the pipeline does not ask the operator for a decision while it is running. No modal, no `input()`, no "press Y to continue". A pipeline that stops mid-run every time it encounters an ambiguous case stops being useful: the operator queues it, walks away, comes back to find it waiting on a question from an hour ago, answers it, walks away again, repeats.

The argument that justifies the rule has two parts. First, the cost of a wrong automated decision has to be bounded by something downstream — a deterministic verifier, a sampling audit, a guard on the write path — before you earn the right not to ask. Second, with that bound in place, the cost of asking is high and the cost of deciding automatically is low, so the automatic decision wins.

The failure mode this rule is designed against is specifically "the operator is away and the pipeline is waiting for them." That failure is more expensive than almost any wrong decision an automated step could make within the bounds set by the verifier.

## Punch-list at the end, not questions during

Anything the pipeline could not resolve on its own does not become a prompt. It becomes a punch-list entry.

A punch-list entry has four fields: what happened, which identifier it affects, what the pipeline did in the meantime (usually "skipped" or "marked for review"), and what action the operator could take. The entries accumulate during execution and are presented together at the end as a single document the operator reviews once.

The structural property that matters: the operator reads the punch-list once, in a known place, after the pipeline is done. They are not interrupted during their other work, and they do not have to context-switch into the pipeline every time it hits something unusual.

A good punch-list is a design artefact. Every entry on it is also a candidate for a future rule: if the same shape of entry appears every run, the pipeline should learn to handle it, and the punch-list should get shorter over time.

## When "bounded" earns automation and when it does not

The distinction that drives what gets automated is whether the cost of being wrong is bounded.

**Bounded**: the decision writes an observation that another layer re-derives from the underlying signals and refuses to accept if the derivations disagree. Or the decision writes to a shared table through a guard that insists on a reason from a closed enum and keeps an audit record. Or the output flows to a sampling check that will block the batch if agreement drops below a threshold. In these cases, the worst outcome of a wrong automated decision is a flagged record that an operator handles later. That is a cost you can price.

**Unbounded**: the decision triggers a side effect that cannot be undone cheaply — a message sent to a client, a deletion from a store you cannot reverse, a merge that another step relies on and will not notice is wrong until much later. These cases do not get automated. They go on the punch-list even when the pipeline is confident, because confidence is not the same as a cheap way to recover.

The question to ask about a new automated step is not "is it correct on average" but "if it is wrong on this specific record, what is the cost of recovery?". If that number is small and known, automate. If it is large or unknown, surface it.

## Documentation as insurance against the bus factor

A pipeline authored by one person is a liability as long as the knowledge of how to operate it lives only in that person's head. The documentation that mitigates this is not optional.

**README.** Written for someone who has never run the pipeline. Covers what it does, what it expects as input, what it produces as output, and how to run both stages. Avoids referring to things the reader does not have yet.

**Runbook.** Written for someone who runs the pipeline occasionally and has forgotten the specifics. Covers the step-by-step for the common cases, the known failure modes with their resolution, and the sanity checks that confirm the output looks right.

**Onboarding doc.** Written for the person who will inherit the pipeline. Covers the architecture at a level above the code: what the stages are, where the bounds are, which decisions were taken and why. Links to the decision log for the full history.

**Decision log, numbered.** Each entry is a dated decision with the alternatives considered, what was chosen, and what trade-off was accepted. Numbered because a later entry can reference an earlier one. The numbering is cheap and the reference ability pays off within six months.

The documentation is the handoff artefact. If the author steps away, the next person has to be able to operate the pipeline from the first three documents and extend it from the fourth. If any of those three is missing, the pipeline is not finished, regardless of how well it runs.

## What this stance is not

It is not an argument against automation. The whole point is that most of the work gets automated — just not every decision, and not in a way that requires a human to be present for the execution.

It is also not a defence of ad-hoc scripts. The two-stage structure, the bounded-decision rule, the punch-list format, and the documentation suite are a specific pattern. They are more disciplined than a script and less infrastructure-heavy than a scheduler, which is the point.
