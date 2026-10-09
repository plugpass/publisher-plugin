---
name: plugpass-test-plugin
description: >
  Installs a test version of the publisher's plugin on one of their supported AI products
  and provides a link that puts their browser into test mode for the test user they specify,
  so they can test signing up, encountering paywalls, upgrading, using premium features, and
  the rest of the plugin's user experience without paying. When they're done testing, it
  switches them back to their real plugin.
allowed-tools: mcp__plugin_plugpass_plugpass-publisher__plugpass_get_plugin_data, mcp__plugin_plugpass_plugpass-publisher__plugpass_list_test_users, mcp__plugin_plugpass_plugpass-publisher__plugpass_save_test_user, mcp__plugin_plugpass_plugpass-publisher__plugpass_reset_test_user, mcp__plugin_plugpass_plugpass-publisher__plugpass_test_as_user, Read, Write, Edit, Glob, Grep, Bash(unlink:*), Bash(rsync:*), Bash(cp:*), Bash(zip:*), Bash(mkdir:*), Bash(mktemp:*), Bash(claude:*), Bash(codex:*), Bash(/Applications/ChatGPT.app/Contents/Resources/codex-cli/bin/codex:*), Bash(echo:*), Bash(open:*), Bash(xdg-open:*), PowerShell(New-Item:*), PowerShell(Remove-Item:*), PowerShell(Copy-Item:*), PowerShell(Compress-Archive:*), PowerShell(claude:*), PowerShell(codex:*), PowerShell(Write-Output:*), PowerShell(Start-Process:*), AskUserQuestion, Skill, WebSearch, WebFetch
---

This skill works in the publisher's plugin repo: it reads files in the current working directory and runs terminal commands there. If your environment cannot do both, tell the user "Plugpass needs to read files in your plugin's repo and run terminal commands, which isn't possible here. Open your plugin's directory in an AI coding tool like Claude Code or Codex and run this skill again." and end the skill.

## What this skill does

Gets the publisher into test mode: it confirms the plugin is ready to test, installs the working copy on the AI product they choose as a test version, chooses or creates a Plugpass test user, and posts the link that puts their browser into test mode for that test user. When they're done testing, it switches them back to their real plugin. It never decides what to test, never writes tests, and never edits the plugin repo: the test plugin (step 4) is a copy outside the repo.

The Plugpass Publisher MCP server (this plugin's `.mcp.json` `plugpass-publisher` entry) provides the `plugpass_get_plugin_data`, `plugpass_list_test_users`, `plugpass_save_test_user`, `plugpass_reset_test_user`, and `plugpass_test_as_user` tools this skill calls — reference them by those bare names under any connector prefix.

**Work silently.** The only text you post is what a step calls for. Skip the short preamble that normally precedes a tool call — including the first one — and post no step announcements, no commentary on what you just did or are about to do, and no recap the steps didn't ask for.

**Presenting copy.** A `>` block is finished copy; the `>` characters delimit it here and are never part of it. Reproduce the text exactly — substituting each `{VARIABLE}` with its value — and never print the `>` characters, restyle the wording, or wrap it in a quote block. Where a step says to tell the publisher something, post it as your own message with nothing of your own before or after it. Copy given inline in double quotes is delivered the same way, without the quote marks.

- PUBLISHER_PLUGIN_VERSION = `0.0.23` (stamped by the release pipeline). Include it as `publisher_plugin_version` on every Publisher MCP tool call in this skill.
- USER_INPUT_TOOL = A tool that presents the user a question with selectable options and returns their choice (e.g. `AskUserQuestion`, `ask_user_input_v0`, etc.) that can be used in the default session state. Where a prompt below calls for USER_INPUT_TOOL and no such tool is available, ask the question in chat and wait for the reply.
- OS = If your system instructions indicate the platform is `darwin`, then `mac`; if `linux`, then `linux`; if `win32`, then `windows`.
- OPEN_URL_TOOL = A tool that opens a URL in a browser for the user: a dedicated one (e.g. `open_in_codex`) if present, else a shell command (e.g. Bash `open "<url>"`, Bash `xdg-open "<url>"`, PowerShell `Start-Process "<url>"`). Never a web search or page fetch.
- PUBLISHER_TOOLS_MISSING = If a Publisher MCP tool this skill needs is not in your tool catalog under any connector prefix (if not loaded, attempt to load it via tool search), the Publisher MCP server isn't connected: invoke the `plugpass-access-handler` skill and follow its instructions. When it returns after a successful connection, retry the call that needed the tool.
- ARGUMENTS = The text after the command, if any (the plugin's install page passes them as `{product name} {test user email}`): the last whitespace-separated token containing `@` is the test user's email, and whatever precedes it is the product, matched case-insensitively against `{products}` by value or label. An argument that matches nothing is ignored and its question asked as usual. When the first token is `done`, the user is done testing, and whatever follows it is the product.

## Step 1: Confirm the plugin is ready to test

Read `.claude-plugin/plugin.json` (else `.codex-plugin/plugin.json`). `{manifest name}` is its `name`; `{plugin-plugpass-id}` is `metadata.plugpass-plugin-id`. If the id is absent or empty, tell the user "This plugin isn't registered with Plugpass yet. Run the `plugpass-sync-plugin` skill first." and end the skill.

Call `plugpass_get_plugin_data` with `plugin_id: {plugin-plugpass-id}`; keep its `plugin_slug` as `{plugin slug}`, its `plugin_display_name` as `{plugin display name}`, its `plugin_origin` as `{plugin origin}`, its `supported_products` as `{products}`, its `owned_servers` as `{owned servers}` (the publisher's own MCP servers, each with its `.mcp.json` key `server_name` and its `url`), its `connector` as `{connector}` (the plugin's connector, with its `.mcp.json` key `server_key` and its `url`), and its `chatgpt_install_url` as `{listing url}` (the plugin's ChatGPT Plugin Directory listing, or null). When the user is done testing, skip to Switching back.

Call `plugpass_list_test_users` with `plugin_id: {plugin-plugpass-id}` and keep the response as `{test users}`. If `versions.draft` is present and its `outstanding` has an entry whose `code` starts with `impl_`, tell the user "This plugin's premium features aren't implemented yet. Run the `plugpass-implement-code-changes` skill first." and end the skill. If `any_version_ready` is false, tell the user "No version of this plugin is ready to test yet. Finish the plugin's setup in the Plugpass dashboard first." and end the skill. `{version}` is `draft` when `versions.draft` is present and `ready`, else `production`; `versions.{version}.plans` are the plans step 3 offers, and `users` the existing test users.

## Step 2: Ask which AI product to test on

If the product was given in ARGUMENTS, has already been specified, or is otherwise known, skip this question. Otherwise ask with USER_INPUT_TOOL: "Which AI product do you want to test on?", offering each value of `{products}` by its label, in this order: `claude` Claude, `claude_code` Claude Code, `chatgpt` ChatGPT, `codex` Codex. Capture the answer's value as `{product}` and its label as `{product label}`.

## Step 3: Choose or create the test user

If the test user's email was given in ARGUMENTS, has already been specified, or is otherwise known, skip this question: an email among `{test users}` is that user, and any other email is a new one to create. Otherwise, if `{test users}` has any `users` whose `user_type` is `internal`, ask with USER_INPUT_TOOL: "Which test user do you want to test as?", offering each such user's email with its current state, plus "A new test user". Otherwise skip to creating one.

To create one, ask with USER_INPUT_TOOL for the email, offering the publisher's own email first when you know it, and "Another email". Then ask "What state should the test user start in?", offering "Not signed up", "Signed up" (only when no plan of `versions.{version}` has `is_free` true), and each plan of `versions.{version}` (a paid plan once per `intervals` entry, e.g. "Pro, monthly"). Call `plugpass_save_test_user` with `plugin_id`, `email`, `version: {version}`, `starting_status` (`not_signed_up`, `signed_up`, or `plan`), and for a plan `starting_plan_id` and `starting_term` (`monthly` or `annual`; omit for a free plan). When the user asks to test production while the draft is ready, `{version}` is `production` from here on.

If the user chose an existing test user and wants to start over, call `plugpass_reset_test_user` with `plugin_id` and `email`.

`{tested version}` is the chosen user's `resolved.kind`, or `{version}` for a new one. If the chosen user's `resolved.ready` is false, skip to step 5: the link call is refused, and its message names what to do.

## Step 4: Load the test plugin

When `{tested version}` is `production`, the published plugin is what is tested. Tell the user, then skip to step 5:

> If you already have the {plugin display name} plugin installed on {product label} then you can begin testing now. Or follow the [install instructions]({plugin origin}/install?product={product}) if you don't.

Otherwise follow the steps for `{product}`. These are the steps as last verified (2026-10-02); if one no longer matches what you see, find the current path rather than declaring it impossible. The real install of this plugin in that product is turned off for the duration and turned back on afterward.

**The test plugin** is a **copy** of the working copy, made outside the repo, in which each owned server's `.mcp.json` entry (the key named by its `server_name`), or when `{owned servers}` is empty the entry named by `{connector}`'s `server_key`, has its `url` suffixed with `/test`, the connector's test path; the repo is never written, and the copy is refreshed on every run. Make the copy with `rsync -a --delete --exclude .git --exclude node_modules --exclude .plugpass "$PWD/" "{copy path}/"` (PowerShell: `Copy-Item -Recurse -Force` with those folders excluded), creating its parent directory first (`mkdir -p`) and removing a symlink already at the destination (`unlink`), then `Edit` the copy's `.mcp.json`. `{build dir}` is the copy.

Where a product takes an archive, build it from `{build dir}` yourself, leaving out `.git`, `node_modules`, and `.plugpass` directories (Bash: `zip -r "{test plugin path}" . -x '.git/*' 'node_modules/*' '.plugpass/*'` from that directory; PowerShell: `Compress-Archive` over the same directory with those folders excluded). The archive, and a copy made for it, go in your session's scratchpad or outputs folder when it has one, else a temporary directory (`mktemp -d`); `{test plugin path}` is the archive's absolute path.

**Local state** lives in `.plugpass/` in the repo: create it when absent, and add a `.plugpass/` line to the repo's `.gitignore` when it's missing (skip when the plugin directory isn't in a git repo). `.plugpass/test-plugin.json` lists, under each product's value, the installs this skill turned off there, such as `{"codex": ["{manifest name}@<marketplace>"]}`; add to a product's list, keeping what's already on it. `{done command}` is this skill's command in your client followed by `done`: `/plugpass-test-plugin done` in Claude Code, `$plugpass-test-plugin done` in Codex.

- **`claude_code` (terminal and desktop app):** put the copy at `$HOME/.claude/skills/{manifest name}`. If the plugin is also installed from a marketplace, run `claude plugin disable {manifest name}@<marketplace>` first and add `{manifest name}@<marketplace>` to `claude_code`'s list in `.plugpass/test-plugin.json`. Tell the user, leaving out " & enable the real {plugin display name} one again" when `claude_code`'s list is empty:

  > Start a new Claude Code session to load the test plugin.
  >
  > Let me know when you're done testing (or run `{done command}`), and I'll remove the test plugin & enable the real {plugin display name} one again.
- **`claude` (web and desktop, Cowork included):** build the archive, then post the steps:

  > **Install the test plugin in Claude**
  > 1. Open [Claude plugin settings](https://claude.ai/customize/plugins)
  > 2. If the real {plugin display name} is installed, turn it off from its menu
  > 3. Click `Add` → `Upload plugin` and upload `{test plugin path}`
  > 4. If Claude asks to replace the existing plugin, confirm

  Then tell the user:

  > When you're done testing, remove the test plugin (under `Created by you`) and turn the real {plugin display name} back on, both from their menus.
- **`chatgpt` (ChatGPT on the web, and the desktop app's Work and Codex):** run step 5 first and come back here, since creating or connecting ChatGPT's test app signs in as whoever is signed in on the plugin's site. ChatGPT reaches the connector through a developer-mode app the publisher creates once per plugin. `Read` `.plugpass/chatgpt-test-app.json`; `{app hex}` is its `app_id` after `asdk_app_`. When the file or the id is missing, post:

  > **Set up ChatGPT to test {plugin display name}**
  > 1. If you haven't already enabled Developer Mode in ChatGPT, open [ChatGPT's plugin settings](https://chatgpt.com/#settings/Plugins), click `Developer mode`, and turn it on
  > 2. [Create an app](https://chatgpt.com/plugins#settings/Connectors?create-connector=true) named `{plugin display name} test` with the server URL `{connector url}/test`, check `I understand and want to continue`, and click `Create`
  > 3. Open `{plugin display name} test` under [Personal](https://chatgpt.com/plugins?view=personal) and paste its URL here

  with `{connector url}` as `{connector}`'s `url`. The pasted URL ends in `plugin_asdk_app_` and the app's hex: `Write` `{"app_id": "asdk_app_{app hex}"}` to `.plugpass/chatgpt-test-app.json`. If the publisher's account has no developer mode, follow the `codex` steps instead.

  Call `plugpass_save_test_user` with `plugin_id`, `email`, and `chatgpt_app_id: asdk_app_{app hex}`. Then make the test plugin, except that the connector's `.mcp.json` entry (named by `{connector}`'s `server_key`) is removed rather than pointed at its test path, and in the copy `Write` `.app.json` as `{"apps": {"{connector server key}": {"id": "asdk_app_{app hex}"}}}`, set `"apps": "./.app.json"` in `.codex-plugin/plugin.json`, and set `name` to `dev-{app hex}` in `.codex-plugin/plugin.json` and `.claude-plugin/plugin.json`. Build the archive, then post the steps, leaving out the first and numbering the rest from 1 when `{listing url}` is null:

  > **Install the test plugin in ChatGPT**
  > 1. If {plugin display name} is installed in ChatGPT, [uninstall it]({listing url})
  > 2. Open [your test plugin](https://chatgpt.com/plugins/plugin_asdk_app_{app hex}), choose `Upload new version` from its menu, and upload `{test plugin path}`
  > 3. If it shows `Install plugin`, install it, then open [its settings](https://chatgpt.com/#settings/Plugins/plugin_asdk_app_{app hex}) and click `Connect another account`

  Then tell the user, when `{listing url}` is present:

  > When you're done testing, uninstall [your test plugin](https://chatgpt.com/plugins/plugin_asdk_app_{app hex}) from its menu, [end your test session]({plugin origin}/test-session/end), and [reinstall {plugin display name}]({listing url}).

  and otherwise:

  > When you're done testing, uninstall [your test plugin](https://chatgpt.com/plugins/plugin_asdk_app_{app hex}) from its menu and [end your test session]({plugin origin}/test-session/end).
- **`codex` (the Codex CLI and the ChatGPT desktop app):** `{codex}` is the `codex` command, or when it isn't found, the ChatGPT desktop app's own copy (on a Mac, `/Applications/ChatGPT.app/Contents/Resources/codex-cli/bin/codex`). If neither is found, tell the user "I couldn't find Codex on this computer. Run this skill in Codex, or [install the Codex CLI](https://learn.chatgpt.com/docs/codex/cli) and run it again." and end the skill.

  Put the copy at `$HOME/.codex/plugpass-test/plugins/{manifest name}`. Create `~/.codex/plugpass-test/.agents/plugins/marketplace.json` as `{"name": "plugpass-test", "interface": {"displayName": "Plugpass test plugins"}, "plugins": []}` when it's absent, and add `{"name": "{manifest name}", "source": {"source": "local", "path": "./plugins/{manifest name}"}}` to its `plugins` when no entry has that name. Run `{codex} plugin marketplace add "$HOME/.codex/plugpass-test"`, then `{codex} plugin add {manifest name}@plugpass-test`, even when it's already installed. In `~/.codex/config.toml`, set `enabled = false` in each `[plugins."{manifest name}@<marketplace>"]` table other than `plugpass-test`'s that has `enabled = true`, unless `<marketplace>` ends in `-remote`, and add each `{manifest name}@<marketplace>` you turn off to `codex`'s list in `.plugpass/test-plugin.json`.

  Then post the following, leaving out " & enable the real {plugin display name} one again" when `codex`'s list is empty. When `{listing url}` is present:

  > **Load the test plugin in Codex**
  > 1. If {plugin display name} is installed from the ChatGPT Plugin Directory, [uninstall it]({listing url})
  > 2. Start a new Codex session. If you're testing in the ChatGPT desktop app, restart the app first
  >
  > Let me know when you're done testing (or run `{done command}`), and I'll uninstall the test plugin & enable the real {plugin display name} one again.

  and otherwise:

  > Start a new Codex session to load the test plugin. If you're testing in the ChatGPT desktop app, restart the app first.
  >
  > Let me know when you're done testing (or run `{done command}`), and I'll uninstall the test plugin & enable the real {plugin display name} one again.

## Step 5: Open the Test as user link

Call `plugpass_test_as_user` with `plugin_id` and `email`; `{link}` is its `url`, single-use and valid for ten minutes. If the call is refused, relay its message to the user verbatim and end the skill. When `{product}` is `chatgpt` and step 4's ChatGPT steps are still to run, return to them once the link is open.

When you will drive the testing yourself in a browser you control (a browser tool such as the Chrome extension on Claude, Chrome DevTools, or the in-app browser on the ChatGPT desktop app), open {link} there. When the user has said where to open it, do that. Otherwise post the following and ask with USER_INPUT_TOOL, offering "Yes" and "No, I'll paste the link somewhere else":

> I've generated the link that initiates test mode for {email}:
> [{link}]({link})
>
> Do you want me to open the test mode link in your browser?

On "Yes", open {link} with the OPEN_URL_TOOL. If that fails for any reason (error, approval declined, user says it didn't open, etc.), tell the user:

> I couldn't open the page. Use the link above.

## Step 6: Hand back

Tell the user:

> Test mode as {email} lets you test signing up, encountering paywalls, upgrading, using premium features, and the rest of {manifest name}'s user experience. A few things worth knowing:
> - To try the true new-user experience, disconnect or remove the plugin's connector in {product label} first; a connected client skips sign-up.
> - The first subscription counts as an upgrade. Premium features, upgrades, and billing changes can be tested in any order; payments are simulated.
> - Reset returns the test user to its starting state, from the plugin's Test page in the dashboard or by asking me.

When `{tested version}` is `draft`, add:

> - Keep the test plugin current before each session.

When `{tested version}` is `draft`, `{product}` isn't `chatgpt`, and `{owned servers}` is not empty, add:

> - The test plugin points your own MCP {server | servers} at {its | their} test path, which is what test mode uses. If you add {its | a server's} connector by URL, use the test path ({url}/test for each server), turn the real connector off while you test since the two carry the same tool names, and remove the test connector when you're done.

When `{tested version}` is `draft`, `{product}` isn't `chatgpt`, and `{owned servers}` is empty, add, with `{url}` as `{connector}`'s `url`:

> - The test plugin points the plugin's connector at its test path, which is what test mode uses. If you add the connector by URL, use its test path ({url}/test), turn the real connector off while you test since the two carry the same tool names, and remove the test connector when you're done.

If `{test users}` has `has_unpublished_changes` true, add:

> When you're done testing, publish your changes in Plugpass to make them live.
>
> [Publish plugin](https://plugpass.ai/dashboard/plugin/{plugin slug}/publish)

## Switching back

When the user says they're done testing, or ARGUMENTS starts with `done`, switch back on `{product}`: the product named after `done`, else the one tested in this chat, else the one they choose when asked with USER_INPUT_TOOL "Which AI product are you done testing on?", offering `{products}` as step 2 does.

- **`codex`:** with `{codex}` as step 4 defines it, run `{codex} plugin remove {manifest name}@plugpass-test`, remove the `{manifest name}` entry from `~/.codex/plugpass-test/.agents/plugins/marketplace.json`, and run `{codex} plugin marketplace remove plugpass-test` when no entries remain. In `~/.codex/config.toml`, set `enabled = true` in the table of each install on `codex`'s list in `.plugpass/test-plugin.json`, then empty the list. Tell the user "Start a new Codex session to load the real {plugin display name}. If you're using the ChatGPT desktop app, restart the app first.", or when the list was empty, "Start a new Codex session to finish removing the test plugin. If you're using the ChatGPT desktop app, restart the app first." When `{listing url}` is present, add:

  > If you uninstalled {plugin display name} from the ChatGPT Plugin Directory, [end your test session]({plugin origin}/test-session/end) and [reinstall it]({listing url}).
- **`claude_code`:** delete `$HOME/.claude/skills/{manifest name}`, run `claude plugin enable` on each install on `claude_code`'s list in `.plugpass/test-plugin.json`, then empty the list. Tell the user "Start a new Claude Code session to load the real {plugin display name}.", or when the list was empty, "Start a new Claude Code session to finish removing the test plugin."
- **`claude` and `chatgpt`:** post that product's step 4 lines that begin "When you're done testing", with their values as step 4 defines them. When ChatGPT was tested through the `codex` steps, follow `codex` instead.
