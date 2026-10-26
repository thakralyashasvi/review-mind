# AI-Powered Code Review Agent: Adaptive, Context-Aware Code Quality Analysis

A code review can be technically correct and still be unhelpful. I have seen reviews that flag a style choice the repository uses everywhere, recommend an abstraction the codebase has deliberately avoided, or repeat feedback that was already settled three pull requests ago. The problem is not just finding defects; it is remembering what a particular codebase considers a defect, and why.

That is the problem I set out to solve with a context-aware code review agent. The central design decision is to treat review as a process with memory, not as a prompt that starts from zero every time. Static analysis and language models can identify suspicious code, but useful review also depends on repository conventions, architectural decisions, and the history of previous feedback. I use Hindsight to make that history available when it is relevant, rather than trying to cram the entire past into every request.

## A review agent needs more than a diff

The system starts with the normal unit of code review: a change set. It gathers the changed files, surrounding code, relevant repository metadata, and the results of deterministic checks. Those inputs form the evidence for a review, not a substitute for one another.

I keep the responsibilities separate:

- **Change collection** identifies what changed and retrieves enough surrounding context to interpret it.
- **Static checks** catch issues that are better expressed as rules: formatting, type errors, known vulnerability patterns, and other mechanically verifiable constraints.
- **Context retrieval** finds project conventions, prior decisions, and earlier review feedback that may change how a finding should be interpreted.
- **Review reasoning** evaluates the change against that evidence and produces findings with file locations, explanations, and actionable recommendations.
- **Memory updates** retain useful new information from the review cycle, including accepted corrections and decisions that should affect future reviews.

This separation matters because an LLM should not be the source of truth for everything. A compiler checks types; a linter enforces stable rules; a reviewer weighs design choices against local conventions and intent. The agent coordinates these sources and makes its evidence visible.

In a pull request, each finding should explain the failure mode, point to the code, and distinguish a likely defect from a suggestion. When evidence is weak, the agent should stay quiet. Developers learn to ignore noisy reviewers.

## The hard part is continuity

A generic review prompt can inspect a function and notice a suspicious null check. It is much less reliable at answering questions such as:

- Does this repository intentionally keep validation at the API boundary?
- Has the team already rejected this abstraction because it obscured the data flow?
- Is this pattern an established convention, or merely an accident in one older file?
- Was a similar warning previously shown to be a false positive?

Without durable context, every review begins as if the repository and its maintainers have never made a decision. I could solve part of that problem by placing a large set of instructions in a system prompt, but that approach gets harder to maintain as the codebase grows. It also turns every request into a contest for limited context-window space.

Instead, I use Hindsight as a memory layer for the agent. Hindsight is designed to let agents retain and retrieve information across interactions. Its role here is not to replace source code, tests, or version control. Those remain authoritative. Memory supplies the decisions and lessons that are expensive to rediscover from the current diff alone.

The agent should not remember every line it has seen. It should retain reusable information: a project convention, a decision and its rationale, a repeated false positive, or a correction that should affect future reviews. For each change, it retrieves memories relevant to the code and review question.

I treat memory as evidence, not an instruction that overrides the repository. A convention can be stale, apply to only one subsystem, or be wrong. The agent must compare retrieved context with the current implementation.

For background on the design space, the [Hindsight GitHub repository](https://github.com/vectorize-io/hindsight) describes the underlying memory system, while the [Hindsight documentation](https://hindsight.vectorize.io/) explains how to work with it. The broader idea of [agent memory for AI systems](https://vectorize.io/what-is-agent-memory) is directly relevant: an agent becomes more useful when it can carry forward selected experience rather than treating every task as an isolated conversation.

## The review loop

I structure the review as a sequence of explicit steps. First, the agent establishes what changed and gathers deterministic evidence. Then it retrieves memories relevant to the changed files, symbols, and issue categories. Only after that does it ask the model to assess the patch.

The following snippets illustrate the key interfaces in the completed design. They show the shape of the workflow rather than claiming to be verbatim copies of repository files.

```python
async def review_change(change, repo):
    diff = await repo.get_diff(change)
    context = await repo.get_surrounding_context(diff)
    checks = await repo.run_static_checks(diff)

    memories = await memory.retrieve(
        query=build_review_query(diff, context, checks),
        filters={"repository": repo.name},
    )

    return await produce_review(
        diff=diff,
        context=context,
        checks=checks,
        memories=memories,
    )
```

The order is deliberate. Static checks run before the model forms its review so their results can be treated as concrete evidence. Memory retrieval is scoped to the repository and the change at hand; retrieving a team's entire history would be expensive and would introduce irrelevant context. The query should reflect the actual changed code and the kinds of questions the reviewer needs to answer, not just a generic request to “find bugs.”

The review prompt should also label its inputs. A compiler error, a source-code excerpt, and a remembered architectural preference are not interchangeable. The model needs to know which statements are observations, which are historical context, and which are constraints to consider. That makes it easier to explain a finding and harder for a recalled note to masquerade as current code.

## Memory needs a write policy, too

Retrieval is only half of the problem. If every review comment becomes a memory, the system accumulates noise: speculative suggestions, one-off fixes, and mistakes that happen to sound confident. I treat memory writes as a separate decision.

A useful memory candidate has a reasonable chance of helping with a future task. For example, if a maintainer repeatedly corrects the agent because the project keeps database access inside repository classes, that may be a durable convention. If a reviewer rejects a suggestion once because it is irrelevant to a particular function, that is weaker evidence. The system should preserve the distinction.

A simplified memory-write policy looks like this:

```python
async def record_review_learning(review, maintainer_feedback):
    candidates = extract_memory_candidates(review, maintainer_feedback)

    for candidate in candidates:
        if candidate.is_actionable and candidate.has_supporting_evidence:
            await memory.retain(
                content=candidate.content,
                metadata={
                    "repository": candidate.repository,
                    "category": candidate.category,
                    "source": candidate.source,
                },
            )
```

The production implementation also needs deduplication, provenance, a distinction between confirmed conventions and tentative observations, and a way to supersede stale information. The agent does not learn just because it generated a sentence; it learns when evidence supports a reusable lesson.

A memory such as “API handlers validate incoming requests” should link back to the decision or code pattern that supports it. Provenance lets maintainers challenge a memory instead of treating an old summary as unquestionable fact.

## Findings should carry evidence, not just confidence

A review agent needs a disciplined output contract: identify the affected location, explain the failure mode, and provide evidence a developer can verify. Severity should reflect impact, not rhetorical confidence.

One useful representation is a typed finding:

```python
from dataclasses import dataclass
from typing import Literal

@dataclass
class Finding:
    path: str
    line: int
    severity: Literal["critical", "high", "medium", "low"]
    title: str
    explanation: str
    evidence: list[str]
    recommendation: str
```

The exact schema can vary, but structured output makes downstream behavior predictable. The pull-request integration can render findings consistently, suppress duplicates, and associate a finding with a changed line. Tests can validate that required fields exist and that locations refer to the patch. A prose-only response makes these tasks much harder.

A confidence score is no guarantee; models can be confidently wrong. Evidence is more useful than a decimal: a code path, failing test, static-analysis result, or relevant convention. Unsupported claims should be downgraded or omitted.

Memory improves this step by supplying local expectations. Imagine a change that adds a second way to access a database. A generic reviewer might recommend extracting a shared abstraction immediately. If retrieved project history says the team has intentionally kept these paths separate because they have different transaction semantics, the agent has a reason to inspect those semantics before recommending a refactor. It may still find a real defect, but it should not repeat a previously rejected suggestion without new evidence.

## What changes in practice

Consider a pull request that introduces a new API endpoint. Static analysis finds no type errors, and the tests pass. The endpoint nevertheless performs validation in a service layer, while the repository consistently validates untrusted request data at the boundary.

A stateless model might identify the inconsistency—or it might accept the implementation because the code looks locally reasonable. With relevant memory, the agent can retrieve the established boundary-validation convention and inspect whether the new endpoint follows it. A useful finding would explain the concrete risk, point to the new handler, and cite the convention as context. It would not merely say “follow best practices.”

Now consider the opposite case: a generic reviewer repeatedly suggests adding a common helper, and maintainers have already explained that the existing duplication is intentional because the call paths have different failure behavior. That feedback can become a scoped memory. On a later change, the agent can avoid repeating the same suggestion unless the new code provides evidence that the earlier rationale no longer applies.

These examples describe intended behavior, not benchmark results. Memory cannot guarantee complete defect detection or eliminate false positives; it makes repository-specific context available during review.

## Guardrails for a reviewer that remembers

Memory introduces failure modes, so persistence cannot be assumed to be beneficial.

First, retrieval must be relevant. A billing memory should not influence authentication code just because both mention validation. Repository and subsystem metadata help narrow semantic search.

Second, memories need provenance and correction paths. If maintainers contradict a stored rule, update or supersede it instead of accumulating conflicting statements. Recency matters, but newer is not always truer.

Third, source code and executable checks are stronger evidence of current behavior than a summary of past behavior. Memory explains intent; it does not prove what the code does today.

Finally, keep reviews auditable. Developers should see the code, finding, and reasoning. If memory influenced a conclusion, expose that context rather than hiding it behind a score.

## Lessons I would carry into another agent

**1. Make deterministic tools do deterministic work.** Compilers, tests, and linters should handle checks they can perform reliably. Use the model where interpretation and trade-offs are actually required.

**2. Treat memory as evidence, not authority.** A remembered convention can improve a review, but it can also be stale or misapplied. Scope it, preserve its source, and check it against the current code.

**3. Separate learning from generation.** The fact that an agent wrote a review comment does not make the comment worth remembering. Memory writes need evidence, deduplication, and a way to correct past conclusions.

**4. Optimize for useful findings, not maximum findings.** A quiet reviewer that surfaces a few actionable issues is more valuable than one that produces a long list of low-confidence style suggestions.

**5. Keep the review contract structured.** Stable finding fields make results testable, renderable, and auditable. Free-form prose alone is difficult to integrate into a real engineering workflow.

The most interesting part of a code review agent is not that it can inspect a diff. Plenty of tools can do that. The harder engineering problem is building a reviewer that understands the difference between a general rule, a local convention, and a decision the maintainers have already made. Hindsight gives the system a practical way to retain selected experience across reviews. The rest of the design—scoped retrieval, evidence-backed findings, careful memory writes, and correction paths—is what makes that experience useful instead of merely persistent.
