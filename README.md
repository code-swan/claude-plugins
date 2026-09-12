# CodeSwan for Claude Code and Codex

Ask your organisation's software catalog from your coding assistant: which services
exist, what they do, who owns them, what depends on what, and what breaks if you change
it.

## Install

```bash
/plugin marketplace add code-swan/claude-plugins
/plugin install codeswan@codeswan
```

The first call opens a browser to sign in. No token goes in any config file.

> **Already added the server by hand?** If you have a `codeswan` entry from
> `claude mcp add` or a claude.ai connector, remove it first — two entries pointing at
> the same URL make the sign-in fail with a `redirect_uri` mismatch. Check with
> `claude mcp list`.

## For a whole team

Commit this to your repository's `.claude/settings.json`. Everyone who trusts the
project gets the catalog with no setup of their own:

```json
{
  "extraKnownMarketplaces": {
    "codeswan": {
      "source": { "source": "github", "repo": "code-swan/claude-plugins" }
    }
  },
  "enabledPlugins": { "codeswan@codeswan": true }
}
```

## What you get

A set of read-only tools over your organisation's catalog, and a skill that tells the
assistant when to reach for them.

Ask things like:

- "Is there already a service that handles refunds?"
- "What calls the payment service, and what breaks if we change its API?"
- "Who owns the components that publish to `order-events`?"

The catalog is built from a scan of your repositories, so it covers services whose source
you do not have checked out, and connections that never appear as a matching string in
any file — a gRPC call, or an event published to a topic another service subscribes to.

## Self-hosted installs

This plugin points at the hosted service. An on-prem install has its own URL and is
configured directly — ask your CodeSwan contact for the setup guide that ships with your
deployment.

## Links

- [code-swan.com](https://code-swan.com)
- Questions or problems: contact your CodeSwan representative.
