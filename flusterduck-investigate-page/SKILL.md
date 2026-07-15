---
name: flusterduck-investigate-page
description: Use when the user names a specific page, route, or flow that is underperforming and wants to know where and why users get stuck on it.
---

# Flusterduck Investigate Page

Deep single-page diagnosis. Given one route, this skill finds the worst friction sources on it, decides whether the problem is acute (appeared after a change) or chronic (always been there), and recommends specific changes ranked by expected impact.

**When NOT to use this skill:** no page named, just "what should I fix" = `flusterduck-triage`. "Did my fix to this page work" = `flusterduck-verify-fix`. Flusterduck not installed = `flusterduck-setup`.

## Data access

**MCP first:** `get_page`, `get_elements`, `get_trends`, `get_deploys`, `diagnose_journey_friction`, `get_alerts`, `explore`, `get_session_detail`.

**REST fallback** at `https://api.flusterduck.com/v1` with `Authorization: Bearer fd_sec_...` (`site_id` optional for site-scoped keys):

```
GET /query/page?site_id={id}&page=/checkout
GET /query/elements?site_id={id}&page=/checkout
GET /query/trends?site_id={id}&page=/checkout&days=14
GET /query/deploys?site_id={id}
GET /query/journeys/friction?site_id={id}&min_friction_weight=5
GET /query/alerts?site_id={id}&status=fired
```

## Procedure

1. **Resolve the exact path.** Page lookups are exact-path. If the user said "the checkout" rather than a path, call `get_scores` (REST: `/query/scores`) and match against tracked paths. No match at all: see failure modes.
2. **Pull the page's full context.** `get_page` with `page: "/checkout"` returns score history, active issues, element friction, deploys, annotations, and confusion budgets in one call. This is the backbone; read it before forming any hypothesis.
3. **Rank the worst elements.** `get_elements` with the same `page` ranks buttons, forms, and links by friction signal volume. Keep the top three with signal type, element label, and count. These are the concrete nouns of the diagnosis.
4. **Acute or chronic.** `get_trends` with `page` and `days: 14`, plus `get_deploys`. A spike aligned with a deploy timestamp is acute (point at the change, suggest revert or patch). A flat-high line is chronic (a design problem; suggest a redesign of the failing interaction).
5. **Entry and exit friction.** `diagnose_journey_friction` (optionally with `min_friction_weight: 5`) and look for edges whose `from` or `to` is this page. Users looping between this page and pricing/FAQ signals a missing answer on this page, not a broken control.
6. **Cross-check the alert surface.** `get_alerts` with `status: "fired"`: an already-firing alert on this page changes urgency and should be named in the output.
7. **Optional depth.** For evidence-grade narrative, `explore` with filters like `[{field:"page",op:"is",value:"/checkout"},{field:"signal",op:"is",value:"rage_click"}]` and `output {mode:"list"}` returns real example sessions; `get_session_detail` on one of them gives the event-by-event story. Use `explore` with `{mode:"measure",metric:"conversion_rate",group_by:"cohort"}` to state what the friction costs in conversion terms.

## Reading the signals

Map the dominant signal pattern to an issue class and its standard fixes:

- **Dead or rage clicks on one element**: it looks interactive and is not, or does nothing visible. Fix the styling, wire the handler, explain the disabled state, add click feedback.
- **Idle plus scroll hesitation after a result or form step**: no clear next action. Add a primary CTA, explain the result, cut competing links.
- **Form field hesitation, repeated errors, abandonment**: label or validation problem. Add examples, validate earlier, state the required format.
- **Loops to FAQ or pricing and back**: the decision page is missing an answer. Surface it in place.
- **Tap misses concentrated on mobile**: hit target too small or too close to neighbors. Enlarge and space it.
- **Tab thrashing or focus traps**: broken focus order. Fix the order, restore focus on modal close, add escape behavior.
- **Deep-scroll abandonment after a key section**: comprehension failure. Rewrite the section, add a summary, fix hierarchy.

## Output format

```
Page diagnosis: <path>
Confusion score: <X>, <rising | stable | recovering>; <acute since deploy <sha> | chronic>
Top friction sources:
  1. <signal> on <element label>: <count> events, <N> sessions
  2. ...
Where users get stuck: <2-3 sentences grounded in the signals and, if pulled, an example session>
Recommended changes, most impactful first:
  1. <specific UI or copy change> addressing <element/signal>
  2. ...
Open alerts on this page: <list, or none>
```

## What DONE looks like

The user knows the top three friction sources by element and signal name with counts, whether the problem is acute or chronic (and which deploy if acute), and has 2 to 3 concrete changes they could put in a ticket verbatim. If they then ship one, hand off to `flusterduck-verify-fix`.

## Failure modes

- **Page not found or zero data.** Check `get_scores` for near-miss paths (trailing slash, locale prefix, dynamic segment). If the path is genuinely untracked, say so: either the page has no traffic yet or the SDK is not on it. Do not diagnose from imagination.
- **Low signal volume.** A handful of events is an anecdote, not a pattern. Give the read, label it tentative, and say what volume would firm it up. Skip the estimated-impact framing entirely.
- **MCP not connected.** Use the REST routes above with an `fd_sec_` key; no key means `flusterduck-setup` comes first.

## Guardrails

- Every claim cites a real signal name, element label, or count from the data. No invented user quotes, no imagined recordings; Flusterduck has no session replay.
- Keep acute and chronic distinct; they lead to different work.
- Recommend, do not mutate. Issue status changes and annotations are write actions (`update_issue`, `add_annotation`, needing `manage:write`) and run only on explicit request.
- Never reference a free or open-source tier; Flusterduck is a paid product.
