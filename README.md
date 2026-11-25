# playwright-isolated-auth-save

A Claude Code (and Cursor) plugin that lets several sessions use the [Playwright MCP server](https://github.com/microsoft/playwright-mcp) at the same time, and keeps every one of them signed in.

By default the Playwright MCP server uses one persistent browser profile per project. Open a second session and it fails with `Browser is already in use`. The `--isolated` flag fixes that by giving every server its own throwaway profile - but now every browser starts signed out.

This plugin's skill closes that gap:

1. Checks the Playwright MCP server is installed, and offers to install it if not.
2. Explains `--isolated` and asks before adding it.
3. Opens one Playwright window for you to sign in to Entra ID (and any other sites you want).
4. Saves that browser's cookies and localStorage to a storage-state file.
5. Adds `--storage-state <file>` to the server args, so every new isolated browser starts from that template.
6. Opens a fresh browser to verify the sign-in carried over.

## Install

Claude Code:

```shell
/plugin marketplace add KyMidd/PlaywrightIsolatedAuthSave
/plugin install playwright-isolated-auth-save@playwright-isolated-auth-save
```

Then run `/playwright-isolated-auth-save:playwright-isolated-auth-save`, or just tell Claude "save playwright auth".

## Requirements

- Node.js 20 or later
- The Playwright MCP server registered under the name `playwright` (the skill offers to install it)

## What it changes

The skill only changes the `args` of the `playwright` entry in your own user-level config (`~/.claude.json` or `~/.cursor/mcp.json`). If the entry lives in a shared project file (`.mcp.json`, `.cursor/mcp.json`), it stops without changing anything.

It also stops if your entry already uses `--config`, `--extension`, `--cdp-endpoint`, `--endpoint`, `--user-data-dir`, or `PLAYWRIGHT_MCP_*` environment variables, since those setups manage the browser some other way.

## The storage-state file

The sign-in is saved to `~/.claude/playwright-storage-state.json`, with `chmod 600` on macOS and Linux.

That file holds live session cookies. Treat it like a password - don't commit it, don't share it.

When a Playwright browser lands on a sign-in page again (cookies expire), run the skill again to refresh it.

## Using it with other identity providers

The skill checks sign-in against `https://myapps.microsoft.com` and looks for the Entra `ESTSAUTHPERSISTENT` cookie. If you use a different identity provider, fork it and swap the check page and cookie name in `SKILL.md`.

## License

MIT
