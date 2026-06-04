---
title: Context Budget Management
authors:
  - giodl73-repo
created: 2026-06-04
last_updated: 2026-06-04
rfc_pr: TBD
---

# Proposal: Context Budget Management

## Summary

Add a core OpenClaw context-budget surface that can measure static and runtime prompt inputs, explain where budget is spent, recommend concrete optimization actions, and expose read-only management evidence for policy, feed, and review workflows without automatically mutating user configuration or content.

## Motivation

OpenClaw already has several independent context-budget pressures: workspace bootstrap files, installed skills, system prompt additions, tool schemas, runtime conversation state, provider limits, and model-specific context caps. Users can see some of these effects in scattered command output, logs, Doctor checks, or provider metadata, but there is no single report that ties the recurring budget cost to a source, owner, freshness state, and optimization action.

That gap makes context work hard to review. A large `AGENTS.md`, a verbose skill, stale runtime usage, or an oversized tool surface can all look like generic model slowness or compaction pressure. Maintainers also need a durable artifact for PR review: a report that can say which sources grew, why the growth matters, and what a safe next action would be.

The context-budget work should live in core OpenClaw because it needs to understand static workspace injection, installed skills, CLI reporting, and runtime evidence from the same product boundary. Feeds and policy can consume evidence from the report, but the feature should not be built on the feeds stack.

## Goals

- Add a reusable `@openclaw/context-budget` package for context-budget reports.
- Include static contributions from workspace bootstrap files and installed skill instructions.
- Accept runtime evidence from system prompt reports, session usage, provider usage, and current-turn context.
- Preserve source provenance, token estimate status, timestamps, hashes when content is available, and stable contribution ids.
- Add CLI commands for `budget audit`, `budget report`, `budget explain`, `budget recommend`, and `budget manage`.
- Generate actionable recommendations such as trimming bootstrap files, optimizing skills, deferring tool schemas, refreshing stale evidence, and staying under a soft token limit.
- Add read-only management artifacts: patch previews, policy evidence, feed evidence, and recent-report deltas.
- Keep report schemas versioned and parseable across the planned PR stack.
- Make every stage reviewable as a small PR with focused tests and Codex review proof.

## Non-Goals

- Automatically editing `AGENTS.md`, skills, plugin manifests, provider config, or feed documents.
- Enforcing policy decisions inside the context-budget package.
- Making feeds a dependency for context-budget measurement or recommendations.
- Replacing model-specific context-window metadata, compaction, or provider usage accounting.
- Scoring answer quality or deciding whether a model should have used more or less context.
- Adding a background daemon, scheduled scanner, or telemetry upload path.
- Requiring live provider calls to produce a static context-budget report.

## Proposal

OpenClaw should add a core `@openclaw/context-budget` package and expose it through the native `openclaw budget` CLI. The report is a versioned JSON document with a generated timestamp, workspace root, summary totals, source contributions, recommendations, and optional management evidence.

The shortened PR series is:

1. Static audit foundation.
   Add the package, static source scanning, schema parsing, Markdown formatting, and `budget audit`, `budget report`, and `budget explain` commands.
2. Runtime evidence inputs.
   Extend reports with runtime provenance for system prompt reports, injected workspace files, current-turn context, tool schemas, session usage, and provider usage. Runtime rows can be estimated, exact, stale, or missing.
3. Actionable recommendations.
   Add limit-aware recommendations with stable ids, severity, source families, contribution ids, action text, and optional token savings. Add `budget recommend` and soft-limit CLI support.
4. Management integrations.
   Add read-only management evidence for review and governance consumers: patch previews, policy evidence, feed evidence, and recent-report deltas. Add `budget manage` and versioned schema compatibility for older reports in the stack.

A contribution represents one budget source:

```jsonc
{
  "id": "oc://workspace-file/AGENTS.md",
  "label": "AGENTS.md",
  "source": {
    "family": "workspace-file",
    "type": "workspace-bootstrap-file",
    "target": "oc://workspace-file/AGENTS.md",
    "path": "AGENTS.md",
    "sha256": "...",
    "collectedAt": "2026-06-04T00:00:00.000Z",
    "provenance": "static-scan"
  },
  "tokens": 420,
  "tokenEstimate": "estimated",
  "bytes": 1600,
  "characters": 1600
}
```

Runtime evidence uses the same contribution shape but sets provenance to `runtime-evidence`. Content-backed runtime rows can include hashes; aggregate rows such as provider usage can omit hashes and mark exact token totals. The parser must not materialize absent optional fields as `undefined`, because report round trips should preserve the wire shape.

Recommendations are built from contributions rather than from a separate scanner. That keeps the explanation path auditable: each recommendation points back to the exact contribution ids it is about. Recommendations can estimate savings when the estimate is meaningful; zero-token savings should be omitted from user-facing text.

Management evidence is deliberately non-mutating. Patch previews describe candidate edits or ownership actions, but `budget manage` does not apply them. Policy evidence reports whether the current budget posture is within the configured soft limit and whether stale or missing runtime evidence affects confidence. Feed evidence lets feed or catalog owners attach expected token ranges or ownership metadata to contribution ids, but the context-budget package only normalizes and reports that evidence.

## Rationale

Keeping the feature in core gives the package access to the same source families users already think of as OpenClaw context: workspace bootstrap files, skills, runtime prompt reports, tool schemas, and provider/session usage. A plugin-only design would be easier to ship independently, but it would either duplicate source discovery or depend on unstable internal behavior.

The report-first design also keeps the PR series small. Each stage expands one stable artifact instead of wiring a new enforcement path. That makes the work easier to review, easier to test, and easier to consume later from docs, Doctor, policy conformance, feeds, or release checks.

Feeds remain relevant as optional evidence consumers and producers, not as the foundation. A feed can say who owns a skill, what token range is expected, or whether a catalog entry is approved. It should not own measurement of OpenClaw's actual prompt budget.

Automatic edits are deferred because context-budget changes are high-trust changes. Trimming an instruction file or deferring a tool schema can alter agent behavior. The first contract should make costs visible and propose reviewable actions; applying changes can come later with explicit confirmation and separate safety design.

## Unresolved questions

- Which runtime command should be the canonical producer for system prompt evidence: `/context json`, Doctor, a dedicated `budget collect` command, or more than one source?
- How should management evidence appear in Doctor output once this package lands?
- Should policy conformance consume context-budget reports directly, or should it read a smaller normalized evidence document?
- What soft-limit defaults should OpenClaw use for different agent profiles and model families?
- Should a later RFC define explicit apply/fix behavior for safe recommendation classes?
