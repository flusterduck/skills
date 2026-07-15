---
name: flusterduck-triage
description: Use when the user asks what UX issues to fix, what is broken or hurting conversion, or wants a ranked plan built from their open Flusterduck issues.
---

# Flusterduck Triage

Flusterduck clusters behavioral friction signals into UX issues, each with a numeric confidence, severity, affected-user counts, evidence, and a recommended fix. This skill turns the live issue list into a ranked, decision-ready plan: what to fix now, what to watch, and why.

**When NOT to use this skill:** the user named one specific page ("why is /checkout bad") = `flusterduck-investigate-page`. The user asks whether a shipped fix worked = `flusterduck-verify-fix`. Flusterduck is not installed yet = `flusterduck-setup`.

## Data access

**MCP first.** If the Flusterduck MCP server is connected, these are the tools:

- `whoami`: confirm which site and scopes the key has. Call once if you have not already.
- `get_site_context`: one snapshot of scores, open issues, active alerts, deploys, and recommendations. Always start here.
- `get_issues` with `status: "open"` (statuses: `open`, `triaged`, `in_progress`, `ignored`, `resolved`, `regressed`).
- `get_issue` with `issue_id` (UUID): full evidence, session links, verification history, deploy correlation.
- `get_recommendations`: fixes ranked by estimated confusion reduction.
- `get_revenue_impact`: revenue at risk, when the site has conversion tracking configured.
- `get_conversion_insights`: confused-vs-calm cohort comparison; quote its numbers only where `confident` is true.

**REST fallback.** No MCP? The same data is served at `https://api.flusterduck.com/v1` with `Authorization: Bearer fd_sec_...` (an `fd_mcp_` key also reads). `site_id` can be omitted when the key is scoped to one site.

```
GET /query/mcp/context?site_id={id}
GET /query/issues?site_id={id}&status=open&limit=20
GET /query/issues/{issue_id}?site_id={id}
GET /query/recommendations?site_id={id}
GET /query/revenue?site_id={id}
GET /query/scores?site_id={id}
```

## Procedure

1. Call `get_site_context` (REST: `/query/mcp/context`). If it errors, jump to failure modes.
2. Call `get_issues` with `status: "open"`. Also pull `status: "regressed"`: an issue that came back after a supposed fix outranks a new one of equal size.
3. For the top candidates (up to 5, by `score_impact` or `affected_users`), call `get_issue` to read the actual evidence: which signals, how many sessions, which element, whether a `deploy_id` is attached.
4. Layer in `get_recommendations` and, if conversion data exists, `get_revenue_impact`.
5. Rank. Priority is a judgment across five inputs, roughly in this order:
   - **Confidence** (numeric, 0 to 1). Lead with what the evidence supports.
   - **Conversion proximity.** A checkout or signup issue affecting 20 users can outrank a blog issue affecting 200. Use `get_revenue_impact` numbers where they exist; otherwise reason from the page's role.
   - **Reach and trend.** `affected_users`, `affected_sessions`, and whether `last_seen_at` is recent and recurring.
   - **Severity.** Does it block the task or slow it?
   - **Regression status.** `regressed` issues and issues with a recent `deploy_id` are likely caused by a specific change; flag them as such.
6. Split into two buckets. High-confidence issues with clear evidence are **Fix this**. Low-confidence or low-volume issues are **Watching**: name them, but do not prescribe work yet.

## Output format

A ranked plan, not a data dump:

```
Fix this
1. <Issue title> on <page>
   Confidence <0.xx> · <N> users · <M> sessions · regressed after deploy <sha> (if applicable)
   Evidence: <dominant signal> x<count> on <element label/selector>
   Fix: <the issue's recommendation, made concrete>
   Why now: <conversion/revenue/severity reason>

Watching
2. <Issue title> on <page>
   Confidence <0.xx> · <N> users · trend <rising/flat>
   Not enough evidence to act. Recheck in a few days.
```

End with one sentence naming the starting point and why ("Start with #1: it is the only issue on a revenue page and it regressed after Tuesday's deploy").

## What DONE looks like

Every open issue is accounted for in one of the two buckets, each "Fix this" item cites real evidence (signal name, count, element) pulled from `get_issue`, and the user has a single recommended starting point. If they ask you to mark an issue triaged or assign it, that is `update_issue` (MCP, needs `manage:write`) or `PATCH /manage/issues/{id}`; do it only on explicit request.

## Failure modes

- **MCP not connected.** Fall back to the REST routes above with an `fd_sec_` key. No key either? Run `flusterduck-setup` Part 2 first; do not fabricate a plan from nothing.
- **No open issues.** That is a real answer. Check `get_scores` for pages with elevated scores that have not clustered into issues yet, report the site as healthy or early, and say when to look again. Never invent issues to seem useful.
- **Everything is low confidence.** Report a Watching-only plan. Say plainly that Flusterduck does not yet have the evidence to recommend engineering work, and name the one metric that would change that (more sessions on page X, another week of data).

## Guardrails

- Be opinionated in proportion to confidence. A 0.3-confidence issue is never a "fix now."
- Recommend concrete changes ("add a next-step CTA below the calculator result"), never "improve the UX."
- The data is behavioral only: clicks, scrolls, timing, focus, element context. Never claim to know what a user typed or who they are.
- Never invent issues, user counts, or revenue numbers.
- Never reference a free tier or open-source SDK; Flusterduck is a paid, proprietary product.
