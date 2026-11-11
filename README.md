# Easy Spec — Cloud (Claude Code plugin)

Registers the hosted Easy Spec MCP server in Claude Code. That is all it does: one HTTP endpoint,
no credentials, no source.

With it installed you get the Easy Spec tools — suites, flows, page objects, utils, datasets,
environments, secrets, runs — and, when a Chrome side panel is connected, live control of a real
browser (`live_snapshot`, `live_run_step`, `live_replay_flow`, `live_pick_element`).

> **Replaces the old `easy-spec` plugin**, which registered the local, file-based CLI server
> (`npx easy-spec-mcp`). If you still have that installed, remove it — see *Upgrading* below.

## Before you install: ask an admin for an account

**Signing in never creates an account.** An administrator has to create one for you and grant you
a project, using the `create_user` tool or the admin panel's Members screen. Without that you will
reach a sign-in page you cannot get past, and nothing on it will explain why.

Ask for:

- an account, created against **the email address you sign in to Microsoft with**
- access to the project(s) you need

## Install

```
/plugin marketplace add easyhealthinc/easy-spec-claude
/plugin install easy-spec-cloud@easy-spec
```

Then **restart Claude Code** — MCP servers are loaded at startup, so a newly installed one is not
visible until you do.

The first tool call opens a browser to sign in. After that the session refreshes itself and in
practice does not expire again.

Check it worked by asking for your projects — `list_accessible_projects` should return something.
An empty list means the account exists but has not been granted a project yet; go back to your
admin.

## Upgrading from the old local plugin

The marketplace is the same (`easy-spec`), so it updates in place. The plugin inside it is a
different one, so the old install has to go:

```
/plugin marketplace update easy-spec
/plugin uninstall easy-spec@easy-spec
/plugin install easy-spec-cloud@easy-spec
```

If you edited `~/.claude/settings.json` by hand, replace `"easy-spec@easy-spec"` with
`"easy-spec-cloud@easy-spec"` in `enabledPlugins`, and check that the marketplace source points at
`easyhealthinc/easy-spec-claude` rather than a personal fork.

## If you want to drive a real browser

The data tools work on their own. The `live_*` tools need the Chrome extension connected:

1. Sign in to the admin panel and use its **Download extension** button. That is the supported
   source — it serves the bundle built from the same commit as the server, so the panel and the
   server can never disagree about what they are.
2. Unzip it.
3. `chrome://extensions` → turn on **Developer mode** → **Load unpacked** → choose the unzipped
   folder.
4. Open the side panel, sign in with the same account, and pick your project.

`live_status` should then list a connected session. If it does not, the `live_*` tools have
nothing to drive — open the panel rather than retrying the call.

## Configuring it by hand instead

If you would rather edit settings than run the commands, add this to `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "easy-spec": {
      "source": { "source": "github", "repo": "easyhealthinc/easy-spec-claude" }
    }
  },
  "enabledPlugins": { "easy-spec-cloud@easy-spec": true }
}
```

The doubled `source.source` is correct, not a typo: the outer key names the setting, the inner one
names the kind of source.

## Updating

Updates follow the **commit** of this repository, not the `version` string in `plugin.json`. A
`/plugin marketplace update easy-spec` picks up whatever has been pushed here.

## What this deliberately does not ship

Only the OAuth-backed `easy-spec-cloud` endpoint. No API keys or tokens — you authenticate as
yourself — and none of the other MCP servers used when developing Easy Spec itself, one of which
takes a personal GitHub token that has no business being distributed.

## The local plugin

The previous contents of this repository — a plugin for the local, file-based Easy Spec CLI — are
in the history, at the commit before this one was introduced.
