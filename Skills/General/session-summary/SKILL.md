---
name: session-summary
description: Summarize the current Claude Code session and save the useful results to the user's SecondBrain.
disable-model-invocation: true
---

# Session Summary

When explicitly invoked, summarize the current programming session and save the result to the SecondBrain vault.

## Goal

Capture useful information from the session without copying the conversation verbatim.

Focus on:

- What was accomplished.
- Important technical decisions.
- Problems encountered.
- Solutions discovered.
- Important concepts learned.
- Coding practices discovered or reinforced.
- Remaining work.
- Useful project context.

## Session location

Save the summary in:

Sessions/

Do not hardcode the physical location of the SecondBrain vault.

Resolve `Sessions/` relative to the SecondBrain vault.

## Filename

Use:

YYYY-MM-DD - <short descriptive title>.md

## Format

Use:

---
type: coding-session
date: YYYY-MM-DD
project: <project name>
---

# <Title>

## Summary

Short summary of the session.

## What We Did

Describe the important work completed.

## Decisions

Record important technical or architectural decisions and why they were made.

## Problems & Solutions

Record meaningful problems and their solutions.

## Lessons

Record genuinely useful lessons learned.

## Best Practices

Record practices that are worth remembering.

Do not automatically modify Coding Standards from this section.

If a best practice appears important enough to become a permanent personal standard, mention it as a recommendation at the end instead.

## Remaining Work

List unfinished work or useful next steps.

## Related

Link to relevant existing SecondBrain notes using Obsidian wikilinks when appropriate.

## Rules

- Never copy the conversation verbatim.
- Do not include irrelevant conversation.
- Do not invent information.
- Prefer concise, high-value summaries.
- Search existing notes before creating duplicate knowledge.
- Do not modify existing notes unless explicitly requested.
- Do not modify Coding Standards automatically.
- Do not create permanent knowledge notes automatically.
- The session summary itself is the only automatic write.