# AI Work Standard

## Purpose

Universal process standard for AI-assisted work across software, documents, spreadsheets, presentations, research, data, and mixed deliverables.

## 1. Classify the Work

Determine the applicable work type before substantive execution:
- Software Development
- Document Work
- Spreadsheet Work
- Presentation Work
- Research Work
- Data Work
- Mixed Work

Apply only the smallest applicable profile and adapter set.

## 2. AI Session Initialization

For substantive work:
1. Load governing project context and instructions.
2. Classify the work.
3. If a Graphify knowledge graph exists and is applicable, **update Graphify first**.
4. Verify applicable knowledge/tooling availability.
5. Use Graphify as the primary codebase knowledge layer for codebase work.
6. Apply applicable profiles/adapters.
7. Use Context7 for version-sensitive technical research when applicable.
8. Use environment tooling such as EnvKit when applicable.
9. Begin Discovery.

Do not force a tool onto work where it has no useful role.

## 3. Discovery

Determine objective, scope, governing requirements, current state, affected artifacts/code, dependencies, constraints, security/privacy concerns, verification requirements, unknowns, and decisions requiring the user.

For software, use Graphify first for structural relationships and impact, then verify important findings directly.
For documents/data, inspect the authoritative artifact and applicable sources.

## 4. Plan

For non-trivial work:
- define scope in/out;
- define execution tasks;
- define evidence;
- define stop conditions;
- identify material decisions;
- Council-audit the plan;
- save it under `docs/00-project/ai/plans/pending/`;
- obtain explicit user approval.

`continue` never bypasses approval.

## 5. Council

The Council consists of Product/PM, Security, Architecture, QA, and Skeptic.

Council reviews requirements, plans, material implementation concerns, and evidence/completion where appropriate. It identifies ambiguity and risk but does not replace user authority.

## 6. Execute

Execute only within approved boundaries. Use applicable profiles and adapters. Do not silently expand scope or make material decisions for the user.

After material code changes, refresh Graphify before relying on graph results again.

## 7. Verify

Verification is work-type-specific. See the profiles.

Never equate an attempted action with a verified outcome.

## 8. Evidence Before Done

A completion claim requires appropriate evidence. Report what was actually done, verified, failed, or left unresolved.

## 9. Security

Apply universal controls for privacy, access, data handling, secrets, external effects, validation, integrity, destructive actions, and dependency/tool risks. Profiles add specialized controls.

## 10. Package / Dependency Gate

Before adding, removing, or replacing dependencies: establish need, check existing capabilities, research compatibility/security/maintenance, assess impact, and obtain approval unless covered by an approved plan.

## 11. Destructive Change Gate

Destructive or consequential actions require explicit approval unless the exact action is already covered by an approved plan.

## 12. Deviation Protocol

Stop for material deviation. Explain it, identify impact, request approval, update the plan, then continue.

## 13. Completion

Verify, refresh applicable knowledge layers, record durable knowledge when appropriate, and report completion accurately.
