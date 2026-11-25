---
name: playwright-isolated-auth-save
description: This skill should be used when a Playwright MCP browser lands on a Microsoft sign-in page, or the user says "save playwright auth", "re-up playwright auth", "playwright is logged out", "set up isolated playwright", or "playwright says the browser is already in use". Checks that the Playwright MCP server is installed and runs with --isolated, offers to add either, and saves an Entra ID sign-in to a storage-state file that every isolated browser loads at start.
compatibility: Requires Node.js 20 or later. Installs the Playwright MCP server (@playwright/mcp) under the name playwright when it is missing.
---

# Save the sign-in for isolated Playwright browsers

Make the Playwright MCP server run with `--isolated --storage-state <state-file>`, and save the user's Entra ID sign-in to `<state-file>`.

`<state-file>` is `<home>/.claude/playwright-storage-state.json`. Get `<home>` from `node -p "require('os').homedir()"`, replace every backslash with a forward slash, and use the absolute path everywhere below. Create `<home>/.claude` if it does not exist.

## Rules

- Never print cookie values or localStorage contents. Never copy `<state-file>` into a repository or send it anywhere.
- Never type credentials or approve MFA for the user.
- Never `kill` a Playwright MCP server process. Ask the user to do it.
- Never install the server or add `--isolated` without the user's yes.
- Never edit a project config file (`.mcp.json`, `.cursor/mcp.json`), and never add a second `playwright` entry.

## What `--isolated` does

- **Without `--isolated`:** sessions opened in the same project share one browser profile on disk. Sign-ins stay between runs in that project, but only one session can have a browser open. A second one gets "Browser is already in use".
- **With `--isolated`:** each server opens its browser on a temporary profile and deletes it on close. Any number of sessions can run at once, but every browser starts signed out unless `--storage-state` loads a saved sign-in.

## Where the server entry lives

- **Claude Code:** run `claude mcp get playwright`. `No MCP server named "playwright"` means there is no entry. Otherwise its `Scope` line names the file:
  - Local config: `~/.claude.json`, under `projects["<absolute project path>"].mcpServers.playwright`
  - User config: `~/.claude.json`, under the top-level `mcpServers.playwright`
  - Project config: `.mcp.json` in the project root, under `mcpServers.playwright`
- **Cursor:** `mcpServers.playwright` in `.cursor/mcp.json` in the project (project config), else in `~/.cursor/mcp.json` (user config). Neither means there is no entry.

## Changing `args`

Read `--storage-state <path>` and `--storage-state=<path>` as the same argument, in every step.

Steps 3 and 8 change the `args` array of the `playwright` entry. Each time:

1. Read the file named under "Where the server entry lives" again, immediately before editing.
2. Change only the `args` array of the `playwright` entry.
3. In Claude Code, run `claude mcp get playwright` and check its `Args` line shows the change.
4. Ask the user to close any Playwright browser window that is open, then reconnect the server: `/mcp` → `playwright` → **Reconnect** in Claude Code; toggle the server off and on in Cursor settings.
5. End the turn. Continue when the user says it is reconnected.

## Steps

1. **Find the server.** Look for the `playwright` entry as described under "Where the server entry lives".

   - The entry exists: go to step 2.
   - No entry, but Playwright browser tools such as `browser_navigate` are available in this session: tell the user the server is registered under another name or by a plugin, that this skill manages only an entry named `playwright`, and stop.
   - No entry and no Playwright browser tools: tell the user the Playwright MCP server is not installed and ask whether to install it. In the same message, give the explanation under "What `--isolated` does" and ask whether to install it with that flag. If the user declines the install, stop.

   To install:

   - Run `node --version`. If it fails or reports a version below 20, tell the user to install Node.js 20 or later, then stop.
   - Claude Code: run `claude mcp add --scope user playwright -- npx @playwright/mcp@latest`, with `--isolated` appended if the user agreed to it.
   - Cursor: add this under `mcpServers` in `~/.cursor/mcp.json`, with `"--isolated"` appended to `args` if the user agreed to it. If the file does not exist, create it as `{"mcpServers": {}}` first:

     ```json
     "playwright": {
       "command": "npx",
       "args": ["@playwright/mcp@latest"]
     }
     ```

   Then stop, with one of these messages:

   - Installed with `--isolated`: ask the user to restart Claude Code or Cursor and run this skill again to save the sign-in.
   - Installed without `--isolated`: ask the user to restart Claude Code or Cursor. Tell them the profile for each project keeps its own sign-in, so there is nothing to save.

2. **Check the entry is one this skill manages.** Tell the user which of these applies, and stop, when:

   - the entry is in a project config file;
   - `args` contains `--config`, `--extension`, `--cdp-endpoint`, `--endpoint` or `--user-data-dir`;
   - the entry's `env` sets a variable whose name starts with `PLAYWRIGHT_MCP_`.

   Otherwise go to step 3.

3. **Fix the arguments a browser needs.** Decide each of these:

   - `--isolated` is missing: give the user the explanation under "What `--isolated` does" and ask whether to add it. If no, tell them this project's profile already keeps their sign-in and there is nothing to save, then stop. If yes, add `--isolated`.
   - `--storage-state` points at a path other than `<state-file>`: tell the user it will be replaced by `<state-file>` and ask. If no, stop. If yes, remove the argument and its path.
   - `--storage-state` points at `<state-file>` and that file does not exist: remove the argument and its path.

   If nothing is added or removed, go to step 4. Otherwise apply the changes as described under "Changing `args`", then go to step 4.

4. **Open the check page.** Navigate the Playwright browser to `https://myapps.microsoft.com`. If the navigation fails, report the error and stop.

5. **Read the result.**

   - URL `https://myapps.microsoft.com/` and title "My Apps": the browser is signed in. Go to step 7. If the user asked to add another site's sign-in and has not been handed the window yet, go to step 6 first.
   - Any other URL, such as `login.microsoftonline.com` or a federated sign-in page: go to step 6.

6. **Hand the window to the user.** Ask them to:

   - sign in in the Playwright window and answer **Yes** to "Stay signed in?";
   - sign in to any other site to carry over, in tabs of that same window.

   End the turn. When the user says they are done, go back to step 4.

7. **Save the state.** Run this with the Playwright `browser_run_code_unsafe` tool, replacing `<state-file>`. If the server has no such tool, tell the user to update `@playwright/mcp` and stop.

   ```js
   async (page) => {
     const s = await page.context().storageState({ path: "<state-file>" });
     const domains = {};
     for (const c of s.cookies) domains[c.domain] = (domains[c.domain] || 0) + 1;
     const p = s.cookies.find(c => c.name === 'ESTSAUTHPERSISTENT');
     return {
       cookies: s.cookies.length,
       cookieDomains: domains,
       origins: s.origins.map(o => o.origin),
       persistentCookieExpires: p ? new Date(p.expires * 1000).toISOString() : null,
     };
   }
   ```

   On macOS and Linux, run `chmod 600 "<state-file>"` straight after.

   If `persistentCookieExpires` is null, tell the user the sign-in was saved without "Stay signed in" and is short-lived. Offer to redo it: have them sign out at `https://login.microsoftonline.com/logout.srf` in the Playwright window, then go to step 6.

8. **Point the server at the file.** If `args` already has a `--storage-state` argument for `<state-file>`, in either form, go to step 9. Otherwise add `--storage-state` and `<state-file>` as described under "Changing `args`".

9. **Verify.** Close the browser with `browser_close`, then navigate to `https://myapps.microsoft.com`.

   - It lands on "My Apps": close the browser and go to step 10.
   - It lands anywhere else: close the browser, report that the saved sign-in did not load, and stop.

10. **Report** in a few lines: whether `--isolated` was added, `persistentCookieExpires`, the extra sites captured, and anything that failed. Tell the user:

    - other sessions load the saved sign-in the next time their Playwright server opens a browser, and a session started before `args` changed needs its server reconnected first;
    - to run this skill again when a Playwright browser lands on a sign-in page.
