---
name: flusterduck-verify-fix
description: Use after a deploy when the user asks whether a fix worked, a change helped, or an issue is resolved; compares confusion before and after and returns HELD, REGRESSED, or PENDING.
---

# Flusterduck Verify Fix

Flusterduck records every deploy with a `confusion_before` snapshot, back-fills `confusion_after` once post-deploy traffic arrives (starting about 5 minutes in, firming up as traffic accumulates), and opens a verification record on each open issue. This skill reads that machinery and delivers one of three verdicts: HELD, REGRESSED, or PENDING.

**When NOT to use this skill:** the user wants to know *what* to fix = `flusterduck-triage`. They want a diagnosis of one page's problems = `flusterduck-investigate-page`. Flusterduck or deploy tracking is not set up = `flusterduck-setup`.

## Data access

**MCP first:** `get_deploys`, `get_issue`, `get_issues`, `get_trends`, `compare_pages`, `get_page`.

**REST fallback** at `https://api.flusterduck.com/v1` with `Authorization: Bearer fd_sec_...` (`site_id` optional for site-scoped keys):

```
GET /query/deploys?site_id={id}
GET /query/issues/{issue_id}?site_id={id}
GET /query/issues?site_id={id}&status=regressed
GET /query/trends?site_id={id}&page=/checkout&days=3
GET /query/compare?site_id={id}&a=/checkout&b=/checkout-v2
```

## Procedure

1. **Pin down the deploy.** Call `get_deploys`. Match the one the user means (most recent, or by SHA/time if they named it). Record its timestamp, `confusion_before`, and `confusion_after`. If `confusion_after` is null the deploy is too fresh; the verdict will be PENDING unless trends already tell a clear story.
2. **Pin down the issue.** If the user named an issue or you can identify it from context, call `get_issue` with its UUID. Read its verification history (a verification record is created for every open issue when a deploy lands), its defining signals from `signals[]` (signal type, count, sessions), the offending element from the issue's own `element_selector`, and its current status. If they only named a page, call `get_page` and pick the issue whose element matches the fix; if several could match, see failure modes.
3. **Compare before vs after on the affected page.** Call `get_trends` with `page` set to the affected path and `days: 3` (widen to 7 if the deploy is older). Ask two questions of the trend: did the page score drop, and did the issue's defining signals drop? A rage-click issue is only fixed if rage clicks on that element fell.
4. **Look for collateral damage.** Call `get_issues` with `status: "regressed"` (auto-flagged returns) and scan for any new issue whose `first_seen_at` is after the deploy, especially one step downstream of the fixed page. A fix that moves friction to the next screen did not work.

## Verdict rules

- **HELD**: `confusion_after` is materially below `confusion_before` on the affected page, the issue's defining signals dropped, the verification record does not show a regression, and no new downstream issue appeared. Quantify it: "dead clicks on Apply Coupon fell from 41 to 3; page confusion 42 to 14."
- **REGRESSED**: the score is flat or worse, the defining signals persist at similar counts, or the issue carries `status: "regressed"`. Name the element that is still firing.
- **PENDING**: `confusion_after` is not yet filled in, or post-deploy traffic is too thin to separate signal from noise. Say what would firm it up (typically more sessions on the page or a few more hours) and offer to re-check.

When torn between HELD and PENDING, say PENDING. A premature "fixed" that regresses costs more trust than a cautious "wait."

## Output format

```
Fix verification: <issue title or page>
Deploy: <sha/label> at <time>
Verdict: HELD | REGRESSED | PENDING
Page confusion: <before> to <after> on <page>
Defining signals: <signal> on <element>: <before-count> to <after-count>
New/downstream issues: <list, or none>
Next action: <close it out | here is what is still firing | re-check after N more hours of traffic>
```

## What DONE looks like

One clear verdict tied to the issue's own defining signals, with before/after numbers the user could re-derive from the dashboard, plus a single next action. If the verdict is HELD and the user explicitly asks, mark the issue done via `update_issue` with `status: "resolved"` (MCP, needs `manage:write`) or `PATCH /manage/issues/{issue_id}` with `{"status": "resolved"}`. Never mutate without being asked.

## Failure modes

- **No deploys recorded.** Deploy correlation is not wired up, so there is no before/after anchor. Fall back to `get_trends` on the affected page around the time the user says they shipped, present it as a trend read rather than a verification, and suggest setting up deploy tracking so the next fix gets a real verdict.
- **Ambiguous issue.** Several open issues touch the fixed page. List them with one line each (title, element, dominant signal) and ask which one the fix targeted, or verify the one whose element matches the diff if the user shared it. Do not average across unrelated issues.
- **MCP not connected.** Use the REST routes above with an `fd_sec_` key; if there is no key, run `flusterduck-setup` first.

## Guardrails

- Tie the verdict to the issue's defining signals, never to the overall page score alone; scores move for unrelated reasons.
- Behavioral data only: signal counts, scores, verification records. No session replay exists, so never claim to have "watched" a user.
- Never declare victory on thin data, and never reference a free or open-source tier; Flusterduck is a paid product.
