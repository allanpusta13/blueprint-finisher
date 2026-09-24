# AI Work Guideline

## Purpose

Explains WHY and HOW TO THINK when applying the AI Work Standard.

## Universal model

```text
Intent → Context → Discovery → Evidence → Plan → Decision → Execution → Verification → Evidence
```

## Graphify

Graphify is the primary **codebase** knowledge layer when applicable. Update it first before substantive codebase analysis. Use it for structure, relationships, dependencies, callers/callees, and impact. Verify important findings against source, tests, or runtime evidence. Treat inferred/ambiguous relationships as hypotheses. Refresh after material code changes.

Graphify is not universal authority and is not required for a document-only task with no meaningful graph.

## Context7

Use Context7 for version-sensitive external technical knowledge when applicable. Project-specific requirements, artifacts, source, approved plans, and verified behavior remain authoritative for the project.

## Environment tools

Environment tools provide operational context. They do not decide product requirements, architecture, or approval boundaries.

## Work profiles

Profiles contain only rules that materially change discovery, execution, verification, or security for a work type. Avoid profile explosion.

## Adapters

Technology/tool adapters isolate stack-specific knowledge from the universal core. A Laravel adapter, for example, may exist in a Laravel project without making Laravel part of the universal standard.

## Epistemic discipline

Use the categories Stated, Observed, Inferred, Proposed, and Approved. Do not turn assumptions into facts.

## Outcome truthfulness

Report what actually happened. Do not say a file, test, change, or result was verified unless it was actually verified.

## Scope and autonomy

Routine choices clearly implied by approved requirements and established conventions may proceed. Material product, architecture, security, data, dependency, or consequential decisions require the user's decision.

## Evidence

Different artifacts need different evidence, but the principle is universal: completion is a claim supported by evidence appropriate to the work.

## Durable knowledge

Promote important decisions, research, plans, and meaningful history to durable documentation. Temporary tool output is not automatically authority.

## Core principle

> AI should be autonomous within approved boundaries, truthful about evidence, deliberate about risk, and deferential to the user's authority over material decisions.
