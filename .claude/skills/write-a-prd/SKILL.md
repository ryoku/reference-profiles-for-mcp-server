---
name: write-a-prd
description: Create a PRD through user interview, repo exploration, and content design, then submit as a GitHub issue. Use when user wants to write a PRD, create a product requirements document, or plan a new feature.
---

This skill will be invoked when the user wants to create a PRD. You may skip steps if you don't consider them necessary.

1. Ask the user for a long, detailed description of the problem they want to solve and any potential ideas for solutions.

2. Explore the repo to verify their assertions and understand the current state of the documentation — which stacks are already covered, what the existing architecture guides contain, and how `profile.json` and templates are currently structured.

3. Interview the user relentlessly about every aspect of this plan until you reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one.

4. Identify the documentation artifacts that need to be added or modified to complete the implementation. For this repo, those artifacts are:

- New or modified **stack folders** (each following the `node-typescript/` reference structure)
- Sections to add or revise in **`architecture.md`**
- New or updated **Handlebars templates** under `templates/`
- **`profile.json`** field changes (expected directories, naming rules, template paths)

Look for content that is genuinely reusable across stacks versus content that is stack-specific.

Check with the user which artifacts are in scope.

5. Once you have a complete understanding of the problem and solution, use the template below to write the PRD. The PRD should be submitted as a GitHub issue. PRDs should always be written in English.

<prd-template>

## Problem Statement

The problem that the user is facing, from the user's perspective.

## Solution

The solution to the problem, from the user's perspective.

## User Stories

A LONG, numbered list of user stories. Each user story should be in the format of:

1. As an <actor>, I want a <feature>, so that <benefit>

<user-story-example>
1. As a mobile bank customer, I want to see balance on my accounts, so that I can make better informed decisions about my spending
</user-story-example>

This list of user stories should be extremely extensive and cover all aspects of the feature.

## Implementation Decisions

A list of implementation decisions that were made. This can include:

- Which stack folders will be created or modified
- Which sections of `architecture.md` will be added, changed, or removed
- Which Handlebars templates will be created or updated, and what variables they expose
- Which fields in `profile.json` will change (expected directories, naming rules, template paths)
- Content decisions: conventions, naming rules, architectural patterns to document
- Decisions about what belongs in `architecture.md` vs templates vs `profile.json`

Focus on decisions and rationale. Avoid prose that is likely to go stale quickly.

## Review & Validation Decisions

A list of review and validation decisions. Include:

- Which artifacts need a consistency check (e.g. verify that `profile.json` reflects the directory layout described in `architecture.md`)
- Which code examples or templates need to be validated against the prose that describes them
- Completeness criteria for new stack folders (presence of `profile.json`, `architecture.md`, and at least one template)

## Out of Scope

A description of the things that are out of scope for this PRD.

## Further Notes

Any further notes about the feature.

</prd-template>