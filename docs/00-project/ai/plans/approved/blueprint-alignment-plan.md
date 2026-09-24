# Blueprint Alignment Plan

Status: `APPROVED`

## Objective

Fix broken references in the blueprint to align with the Universal AI Work Standard after the AI Standard was moved from `vibe-coding/` to `ai-standard/` and ADRs were deleted.

## Scope

### In Scope
- Update AI Standard references in blueprint from `vibe-coding/` to `ai-standard/`
- Update AI Standard references in CLAUDE.md from `vibe-coding/` to `ai-standard/`
- Remove broken reference to deleted ADR 001 in blueprint
- Remove broken references to deleted documents in Reference Documents section
- Verify all references point to existing files

### Out of Scope
- Moving Phase 0-5 implementation plans (user requested they stay in blueprint)
- Removing code examples (user requested they stay in blueprint)
- Recreating deleted ADRs (user deleted them intentionally)
- Restructuring blueprint content (user approved current structure)
- Changing product requirements or architecture

## Governing Inputs
- User request: Align blueprint with Universal AI Work Standard
- Universal AI Work Standard: `docs/00-project/ai-standard/standard.md`
- Universal AI Work Guideline: `docs/00-project/ai-standard/guideline.md`
- System Blueprint: `docs/00-project/blueprint.md` (12,651 lines)
- Project CLAUDE.md: AI-facing project contract

## Discovery Findings

### Direct Source Inspection
**Current State:**
- AI Standard moved from `docs/00-project/vibe-coding/` to `docs/00-project/ai-standard/`
- Blueprint and CLAUDE.md still reference old `vibe-coding/` paths
- Blueprint contains broken reference to deleted ADR 001 (line 1693)
- Blueprint Reference Documents section referenced deleted files (already cleaned)
- ADRs deleted by user: `001-pipeline-architecture-laravel-native.md`, `002-security-architecture-defense-in-depth.md`
- Implementation plans deleted by user: `phase-0-5-implementation-plans.md`

**Fixed During Discovery:**
- ✅ Updated CLAUDE.md to reference `ai-standard/` instead of `vibe-coding/`
- ✅ Updated blueprint AI Governance Reference to reference `ai-standard/` instead of `vibe-coding/`
- ✅ Removed broken references to deleted ADRs from Reference Documents section

**Remaining Issue:**
- Blueprint line 1693 contains: "*See ADR 001 for detailed architectural decision rationale.*" - ADR 001 was deleted

### Graphify
- Not applicable (no codebase to graph)

### Context7
- Not required (framework behavior not being changed)

### EnvKit
- Not required (no environment changes)

## Council Review

### Product / PM
**Status:** APPROVE - No product changes required
- Product information remains accurate
- Only fixing broken references

### Security
**Status:** APPROVE - No security changes required
- Security architecture principles preserved
- Only removing broken ADR reference

### Architecture
**Status:** APPROVE - No architecture changes required
- Architecture documentation preserved
- Only removing broken ADR reference

### QA
**Status:** APPROVE - No QA changes required
- Requirements remain testable
- Only removing broken reference

### Skeptic
**Status:** APPROVE - Minor fix
- Removing broken reference to deleted ADR
- No unsupported assumptions after fix
- Improves blueprint accuracy

## Decisions Requiring User Approval

None - this is a minor reference fix only.

## Implementation Tasks

### Phase 0 — Discovery
- [x] Identify authoritative sources
- [x] Perform Council review
- [x] Apply Blueprint Responsibility Test
- [x] Create implementation plan
- [ ] Obtain user approval

### Phase 1 — Plan
- [ ] Plan is simple reference fix, no complex restructuring

### Phase 2 — Build
- [ ] Remove broken ADR reference from blueprint (line 1693)
- [ ] Verify no other broken references exist

### Phase 3 — Test / Refine
- [ ] Verify blueprint structure is coherent
- [ ] Verify all references point to existing files
- [ ] Verify blueprint and CLAUDE.md reference correct AI Standard location

## Verification / Evidence
- [ ] Blueprint contains no broken references
- [ ] CLAUDE.md references correct AI Standard location
- [ ] Blueprint references correct AI Standard location
- [ ] All reference paths are valid

## Stop Conditions

None - this is a minor reference fix with no material decisions.

## Approval Metadata
- Council status: APPROVE (all perspectives)
- User approval: APPROVED
- Approved date: 2025-09-20

## Completion Metadata
- Status: PENDING
- Tests: N/A (documentation task)
- Evidence: Reference verification
- Deviations: None
- Completion date: PENDING