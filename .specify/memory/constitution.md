<!--
Sync Impact Report:
- Version change: N/A → 1.0.0
- Added principles: Simplicity, Scope Discipline, Documentation & Testing,
  Open Source First, Clean Minimal Code
- Added sections: Technology Decisions, Quality Standards
- Removed sections: None (initial creation)
- Templates requiring updates:
  - .specify/templates/plan-template.md ✅ no changes needed (Constitution
    Check section is generic gate)
  - .specify/templates/spec-template.md ✅ no changes needed
  - .specify/templates/tasks-template.md ✅ no changes needed
- Follow-up TODOs: None
-->

# Hosting Projects Constitution

## Core Principles

### I. Simplicity

All solutions MUST favor the simplest viable approach. When multiple
implementation paths exist, choose the one with the fewest moving parts,
least abstraction, and lowest cognitive overhead. Complexity MUST be
justified with a concrete, present-tense requirement—not a hypothetical
future need.

### II. Scope Discipline

Design and implement ONLY what is currently in scope. Do not build
abstractions, extension points, or infrastructure for anticipated future
requirements. When scope expands, adapt the existing solution at that time.
YAGNI (You Aren't Gonna Need It) is a hard constraint, not a suggestion.

### III. Documentation & Testing

Every feature MUST ship with exceptional documentation and a clear test
plan. Automated test result documents MUST be generated and maintained.
Documentation is a first-class deliverable, not an afterthought. Test
plans MUST cover acceptance criteria and be verifiable without manual
interpretation.

### IV. Open Source First

Before writing custom code, MUST evaluate existing open source packages,
libraries, and tools that solve the problem. Custom code is permitted only
when no adequate open source solution exists OR when adapting an existing
solution requires low effort. "Low effort" means the customization is
minor relative to building from scratch.

### V. Clean Minimal Code

Production code MUST be clean, readable, and consolidated into as few
files as practical. Do not fragment logic across many small files for the
sake of "separation of concerns" when a single well-organized file
suffices. Favor straightforward imperative code over deep abstraction
hierarchies. The reader MUST be able to understand the full solution
without jumping between many files.

## Technology Decisions

When evaluating tools, libraries, or approaches:

- Search for an existing open source solution first
- Evaluate fitness: does it solve ≥80% of the need out of the box?
- If yes and customization is minor: adopt and adapt
- If no adequate solution exists: build the minimum custom code needed
- Document the decision and alternatives considered

## Quality Standards

- Every feature MUST include a test plan in its spec
- Automated tests MUST produce machine-readable result documents
- Documentation MUST be written for the end user, not the implementer
- Code MUST be reviewable in a single sitting without extensive navigation

## Governance

This constitution supersedes ad-hoc decisions. All implementation work
MUST be evaluated against these principles. When a principle is violated,
the violation MUST be explicitly acknowledged and justified in the
relevant spec or plan document.

Amendments to this constitution require:
1. A written proposal describing the change and rationale
2. Update to this document with version increment
3. Review of dependent templates for consistency

**Version**: 1.0.0 | **Ratified**: 2026-05-04 | **Last Amended**: 2026-05-04
