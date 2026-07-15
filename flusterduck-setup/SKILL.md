---
name: flusterduck-setup
description: Use when the user wants to install Flusterduck on a site, wire up the SDK or a framework wrapper, get a publishable key working, or connect an AI assistant (Claude Code, Claude Desktop, Cursor) to the Flusterduck MCP server.
---

# Flusterduck Setup

Flusterduck is automatic issue tracking for UX friction. Setup has two independent halves: (1) install the browser SDK so the site emits behavioral signals, and (2) connect an AI assistant to the Flusterduck MCP server so it can read the resulting data. Do the half the user asked for; offer the other when it makes sense.

**When NOT to use this skill:** the user wants to read or act on friction data that already exists. That is `flusterduck-triage` (what to fix), `flusterduck-investigate-page` (one page), or `flusterduck-verify-fix` (did a fix work).

## The package names

These are the published packages. Use exactly these names:

- `flusterduck`: the core browser SDK.
- `@flusterduck/next`, `@flusterduck/react`, `@flusterduck/vue`, `@flusterduck/svelte`, `@flusterduck/nuxt`: framework wrappers. Each depends on `flusterduck`.
- `flusterduck-cli` and `@flusterduck/cli`: the CLI, published under both names. They are the same tool at the same version; either works with `npx`.
- `@flusterduck/mcp-server`: the local stdio MCP server for AI assistants.

## The three key types

- `fd_pub_` publishable key: browser-safe, can only send events. Goes in the script tag or `init()`.
- `fd_sec_` secret key: server-side only, reads data through the REST API. Never in client code.
- `fd_mcp_` MCP key: server-side only, for the MCP server. Can carry `manage:write` scope for write tools.

All three are created in the dashboard under Settings > API Keys.

## Part 1: install the SDK

Pick one of three paths, in this order of preference.

### Path A: CLI (framework projects)

```bash
npx flusterduck-cli init
```

It detects the framework (Next.js, React, Vue, SvelteKit, Nuxt), installs the right packages, and injects the init call into the correct entry file. With no `--key` it opens a browser sign-in that returns the key to the terminal. Non-interactive: `npx flusterduck-cli init --key fd_pub_xxxx`. Add `--skip-install` to inject code without touching packages. `npx @flusterduck/cli init` runs the identical CLI.

### Path B: script tag (no build step)

Paste before the closing `</body>`:

```html
<script src="https://flusterduck.com/d.js" data-key="fd_pub_xxxxxxxxxxxx" data-dnt="false" async></script>
```

Two attributes are not optional:

- `async`, never `defer`: a slow or unreachable CDN must never delay the host page's `DOMContentLoaded`.
- `data-dnt="false"`: the SDK respects Do Not Track by default, which silently drops 30 to 50 percent of visitors (Firefox, Brave, privacy extensions). Flusterduck captures no PII, no form values, no keystrokes, so DNT filtering costs data for no privacy gain.

Dynamic injection (tag managers, `document.createElement('script')`) works too: the bootstrap reads `document.currentScript` and falls back to `script[data-key^="fd_pub_"]`.

### Path C: npm, manual wiring

```bash
npm install flusterduck
```

```ts
import { init } from 'flusterduck';

init({ key: 'fd_pub_xxxxxxxxxxxx', respectDoNotTrack: false });
```

The config option is `key`, and `init()` refuses `fd_sec_` and `fd_mcp_` keys. `respectDoNotTrack: false` matters the same way `data-dnt="false"` does on the script tag: the default honors DNT and silently drops 30 to 50 percent of visitors, and Flusterduck captures no PII to begin with. For a framework, install the matching wrapper alongside the SDK. Next.js example:

```tsx
// app/layout.tsx
import { FlusterduckScript } from '@flusterduck/next';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <FlusterduckScript apiKey="fd_pub_xxxxxxxxxxxx" respectDoNotTrack={false} />
      </body>
    </html>
  );
}
```

React uses `FlusterduckProvider` + the `useFlusterduck` hook, Vue a plugin + composable, Svelte and Nuxt a module. All take the `fd_pub_` key. Wrappers expose `setConsent` / `optOut` where the site needs consent gating.

### Verify the SDK install

Flusterduck drops events from `localhost` and automated browsers so dev traffic never pollutes scores. To watch events land during development, add `data-env="development"` to the script tag (or `environment: 'development'` to `init`), click around, and check the dashboard's Live signals feed on the home screen. Remove the attribute before shipping. If nothing arrives, see failure modes below.

## Part 2: connect an AI assistant to the MCP server

Needs two values: an `fd_mcp_` key (preferred; `fd_sec_` also works for reads) and the site ID (UUID). Never use an `fd_pub_` key here; it cannot read data.

Claude Code:

```bash
claude mcp add flusterduck \
  --env FLUSTERDUCK_MCP_KEY=fd_mcp_xxxxxxxxxxxx \
  --env FLUSTERDUCK_SITE_ID=your-site-uuid \
  -- npx -y @flusterduck/mcp-server
```

Claude Desktop (`claude_desktop_config.json`) and Cursor (`.cursor/mcp.json`) use the same shape:

```json
{
  "mcpServers": {
    "flusterduck": {
      "command": "npx",
      "args": ["-y", "@flusterduck/mcp-server"],
      "env": {
        "FLUSTERDUCK_MCP_KEY": "fd_mcp_xxxxxxxxxxxx",
        "FLUSTERDUCK_SITE_ID": "your-site-uuid"
      }
    }
  }
}
```

Optional env: `FLUSTERDUCK_ORG_ID` (unlocks `get_audit_log`, `get_degradation`, `get_webhook_deliveries`, `send_feedback`) and `FLUSTERDUCK_API_URL` (override the API base).

### Verify the MCP connection

Call `whoami` first: it returns the key type, the org and site it is scoped to, and its scopes (`mcp:read` for reads, `manage:write` for writes). Then `get_site_context` for a full data snapshot. Both succeeding means setup is done.

## What DONE looks like

- SDK: with `data-env="development"` set, an interaction shows up in Live signals within seconds. Scores appear as real traffic accumulates; first issues typically within 24 to 48 hours.
- MCP: `whoami` returns the expected site and scopes, and `get_site_context` returns JSON instead of an auth error.

## Failure modes

- **No events arriving.** In order: is the traffic from `localhost` without `data-env="development"`? (Dropped by design.) Is the key an `fd_pub_` key from the right site? Is the tag actually in the served HTML? Is an ad blocker eating the request in this one browser?
- **MCP tools all fail.** `whoami` errors usually mean the wrong key type (an `fd_pub_` key), a typo'd `FLUSTERDUCK_SITE_ID`, or a revoked key. Write tools failing while reads work means the key lacks `manage:write` scope: create a new key with that scope, do not hunt for a bug.
- **User has no key and no dashboard access.** Keys only come from Settings > API Keys, and only org members can mint them. Flusterduck is a paid product with a 3-day free trial (card required, not charged until day 3). Point them at flusterduck.com to create an account; do not invent a key or a free tier.

## Guardrails

- Right key in the right place: `fd_pub_` in the browser, `fd_sec_`/`fd_mcp_` on servers and in MCP config only.
- Flusterduck captures behavioral signals: clicks, scrolls, timing, focus, element labels with PII redacted in the browser. No session replay, no DOM recording, no form values, no keystrokes. Never tell a user to configure recording; the product does not do it.
- All reads and writes go through Flusterduck's APIs or the MCP server. Never suggest querying the database directly.
- Never describe Flusterduck as free or open source, and never suggest a $0 tier.
