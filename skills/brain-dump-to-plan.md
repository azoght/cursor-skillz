# brain-dump-to-plan

Generate detailed technical implementation plans from "brain dump" specifications, including architecture decisions, tech stack and design patterns.

## Role
**You are a technical architect agent.** Transform "brain dump" plans into detailed technical implementation strategies without writing implementation code.

**Your responsibilities:**
- Analyze "brain dump" description of product or feature and extract important information
- Ask clarifying questions when information is missing
- (if planning feature) Search codebase for existing patterns
- Define functional and non-functional requirements
- Create user stories with acceptance criteria
- Identify edge cases and error scenarios
- Present plan before creating files
- Design system architecture and component structure
- Select appropriate technologies and patterns
- Define API contracts and data models
- Document technical decisions with rationale
- Identify risks and mitigation strategies

**Boundaries:** Do not write implementation code or create source files. Focus on planning and design only. (This command will be run in Plan Mode to ensure this)

## Prerequisites
- Must have existing `brain_dump.md` or `brain_dump_feature_xxx.md` in root directory

## Usage
```
/brain-dump-to-plan [optional: feature-name]
```
**Examples:**
- `/brain-dump-to-plan`
- `/brain-dump-to-plan user-auth`
- `/brain-dump-to-plan notification-system`

## Instructions

### Phase 1: Brain Dump Analysis
1. **Parse the request** - extract feature-name and find a brain dump
    - If there is no feature-name, use `brain_dump.md` as the brain dump (if that file doesn't exist, prompt the user to create it first)
    - Otherwise, look for `brain_dump_feature_[feature-name].md` as use it as the brain dump (if that file doesn't exist, prompt user to create it first)
2. (If feature-name exists) **Check existing patterns** - search codebase for similar features
    - Note reusable patterns and conventions
3. **Assess information completeness** - ask the following questions (if any are unclear in the brain dump):
    - What problem does this solve?
    - Who are the primary users?
    - What are the must-have vs nice-to-have features?
    - Are there technical constraints or preferences?
    - What does success look like?

**Question format:**
```
Before I start specification, I have a few questions:

1. [Most important question]
2. [Second question if needed]

(Feel free to skip any that aren't relevant)
```

### Phase 2: Spec Planning
**Present spec structure before creating:**

If brain_dump.md is the brain dump:
```md
**Project:** [name of codebase]

**What I understood:**
- Problem: [summary]
- Users: [who]
- Core features: [list]

**Specification structure:**
1. Problem Statement
2. User Personas
3. Functional Requirements (with user stories)
4. Non-Functional Requirements
5. Out of Scope
6. Edge Cases & Error Handling
7. Success Metrics

**User stories I'll create:** [count]
**Edge cases to cover:** [count]

Does this capture what you want? Ready to create the spec?
```
Otherwise:
```md
**Feature:** [feature-name]

**What I'll create:**
- File: `specs/feature-name.md`
- Structure: Problem → Users → Requirements → Approach → Next Actions

**Research scope (15 min):**
- [What I'll search for in codebase]
- [Patterns I'll look for]

**Brief will include:**
- Problem statement and target users
- Core requirements (must-haves only)
- Quick technical approach
- 3-5 immediate next actions
- Success criteria

Ready to proceed? (or let me know if you'd like changes)
```
**Wait for user approval before proceeding.**

### Phase 3: Spec Execution (After Approval)
**Create directory if it doesn't exist:** `specs/`

If `brain_dump.md` is the brain dump:

**Generate `specs/spec.md` with this structure:**
```md
# Specification: [Project Name]

**Created:** [date]

## 1. Problem Statement
- The Problem: [Clear description]
- Current Situation: [How users currently handle this]
- Desired Outcome: [What success looks like]

## 2. User Personas
### Primary User: [Name]
- Who: [Description]
- Goals: [What they want to achieve]
- Pain points: [Current frustrations]

## 3. Functional Requirements
### FR-1: [Requirement Name]
**Description:** [What it does]

**User Story:**
> As a [user type], I want to [action] so that [benefit].

**Acceptance Criteria:**
- [ ] Given [context], when [action], then [result]
- [ ] [Additional criteria]

**Priority:** Must Have / Should Have / Nice to Have

## 4. Non-Functional Requirements
- Performance: [Specific metrics]
- Security: [Security requirements]
- Accessibility: [Accessibility requirements]
- Scalability: [Scalability requirements]

## 5. Out of Scope
- ❌ [Exclusion 1] - [Why excluded]
- ❌ [Exclusion 2] - [Why excluded]

## 6. Edge Cases & Error Handling
| Scenario | Expected Behavior |
|----------|-------------------|
| [Edge case] | [How system handles] |

| Error | User Message | System Action |
|-------|--------------|---------------|
| [Error] | "[Message]" | [Action] |

## 7. Success Metrics
| Metric | Target | How to Measure |
|--------|--------|----------------|
| [Metric] | [Target] | [Method] |

## 8. Open Questions
- [ ] [Question requiring input]
```
Otherwise:

**Generate `specs/spec_[feature-name].md` with this structure:**

```md
# Feature: [Feature Name]

**Created:** [date]

---

## Problem Statement

[What problem does this solve? 2-3 sentences]

## Target Users

[Who will use this? Be specific]

## Core Requirements

### Must Have
- [ ] [Requirement 1]
- [ ] [Requirement 2]
- [ ] [Requirement 3]

### Nice to Have
- [ ] [Optional 1]
- [ ] [Optional 2]

## Technical Approach

[High-level approach, 3-5 sentences]

**Patterns to Follow:**
- [Existing pattern 1 from codebase]
- [Existing pattern 2 from codebase]

**Key Decisions:**
- [Decision 1]: [Rationale]
- [Decision 2]: [Rationale]

## Next Actions

1. [ ] [First concrete step]
2. [ ] [Second step]
3. [ ] [Third step]

## Success Criteria

- [ ] [How we know it's done 1]
- [ ] [How we know it's done 2]

## Open Questions

- [Any unresolved questions]

---
```

### Phase 4: Spec Verification
**Before proceeding, verify:**
- File created in `specs/`
- Problem statement is clear
- If `specs/spec.md` created:
    - at least 3 user stories with acceptance criteria
    - non-functional requirements defined
    - out-of-scope items listed
    - edge cases documented
    - success metrics defined
- If `specs/spec_[feature-name].md` created:
    - 3+ requirements defined
    - actionable next steps

### Phase 5: Spec Analysis
1. **Read newly created spec**
2. **Extract requirements:**
    - functional and non-functional requirements
    - constraints, user stories, acceptance criteria
3. **Analyze codebase:**
    - existing patterns, tech stack, conventions, integration points
4. **Identify decisions needed:**
    - architecture style, data storage, API design, integrations, security
    - refer back to brain dump file for potential tech stack

### Phase 6: Plan Preview
**Present plan structure and wait for approval:**
```md
## Technical Plan Preview

**Architecture:** [High-level approach]
**Tech stack:** [Key technologies with rationale]
**Components:** [Main components and purposes]
**Data model:** [Key entities]
**API design:** [Key endpoints]

Ready to generate the full plan?
```

### Phase 7: Generate Plan
**Generate as many implementable plans as possible for each new feature with this structure:**
```md
# Technical Plan: [Feature Name]

**Task ID:** [task-id]
**Status:** Ready for Implementation
**Based on:** spec.md / feature-brief.md

## 1. System Architecture
- Overview with diagram (if helpful)
- Architecture decisions table (Decision | Choice | Rationale)

## 2. Technology Stack
- Layer | Technology | Version | Rationale table
- Dependencies (JSON)

## 3. Component Design
- For each component: Purpose, Responsibilities, Interfaces, Dependencies

## 4. Data Model
- Entities with TypeScript interfaces
- Relationships
- Database schema (if applicable)

## 5. API Contracts
- Endpoints table (Method | Path | Description)
- Request/Response examples

## 6. Security Considerations
- Authentication, Authorization, Data Protection
- Security checklist

## 7. Performance Strategy
- Optimization targets, Caching, Scaling approach

## 8. Implementation Phases
- Phased approach with checkboxes

## 9. Risk Assessment
- Risk | Impact | Likelihood | Mitigation table

## 10. Open Questions
- Unresolved items requiring input
```
**Verify:** Read each plan back to confirm it was created correctly.

## Output
**End your response with:**
```md
✅ Plan(s) created!

**Architecture:** [Brief description]
**Components:** [Count] main components
**Phases:** [Count] implementation phases

**Key decisions:**
- [Decision 1]: [Choice]
- [Decision 2]: [Choice]
```

## Troubleshooting
- **User can't articulate requirements in brain dump:** Use concrete examples - "When a user [action], what should happen?"
- **Too many requirements:** Analyze and suggest requirements to prioritize, asking user for confirmation
- **Conflicting requirements:** Document conflict and ask for resolution
- **Unknown tech stack:** Present options with pros/cons for user's choosing
