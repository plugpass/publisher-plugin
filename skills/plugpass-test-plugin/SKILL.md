---
name: plugpass-test-plugin
description: >
  Installs a test version of the publisher's plugin on one of their supported AI products
  and provides a link that puts their browser into test mode for the test user they specify,
  so they can test signing up, encountering paywalls, upgrading, using premium features, and
  the rest of the plugin's user experience without paying.
allowed-tools: mcp__plugin_plugpass_plugpass-publisher__plugpass_get_plugin_data, mcp__plugin_plugpass_plugpass-publisher__plugpass_list_test_users, mcp__plugin_plugpass_plugpass-publisher__plugpass_save_test_user, mcp__plugin_plugpass_plugpass-publisher__plugpass_reset_test_user, mcp__plugin_plugpass_plugpass-publisher__plugpass_test_as_user, Read, Write, Edit, Glob, Grep, Bash(ln:*), Bash(unlink:*), Bash(rsync:*), Bash(cp:*), Bash(zip:*), Bash(mkdir:*), Bash(mktemp:*), Bash(claude:*), Bash(codex:*), Bash(echo:*), Bash(open:*), Bash(xdg-open:*), PowerShell(New-Item:*), PowerShell(Remove-Item:*), PowerShell(Copy-Item:*), PowerShell(Compress-Archive:*), PowerShell(claude:*), PowerShell(codex:*), PowerShell(Write-Output:*), PowerShell(Start-Process:*), AskUserQuestion, Skill, WebSearch, WebFetch
---

This skill works in the publisher's plugin repo: it reads files in the current working directory and runs terminal commands there. If your environment cannot do both, tell the user "Plugpass needs to read files in your plugin's repo and run terminal commands, which isn't possible here. Open your plugin's directory in an AI coding tool like Claude Code or Codex and run this skill again." and end the skill.

## What this skill does

Gets the publisher into test mode: it confirms the plugin is ready to test, installs the working copy on the AI product they choose as a test version, chooses or creates a Plugpass test user, and posts the link that puts their browser into test mode for that test user. It never decides what to test, never writes tests, and never edits the plugin repo: a test build that differs from the working copy (a plugin with its own MCP servers, step 4) is a copy outside the repo.

The Plugpass Publisher MCP server (this plugin's `.mcp.json` `plugpass-publisher` entry) provides the `plugpass_get_plugin_data`, `plugpass_list_test_users`, `plugpass_save_test_user`, `plugpass_reset_test_user`, and `plugpass_test_as_user` tools this skill calls — reference them by those bare names under any connector prefix.

**Work silently.** The only text you post is what a step calls for. Skip the short preamble that normally precedes a tool call — including the first one — and post no step announcements, no commentary on what you just did or are about to do, and no recap the steps didn't ask for.

**Presenting copy.** A `>` block is finished copy; the `>` characters delimit it here and are never part of it. Reproduce the text exactly — substituting each `{VARIABLE}` with its value — and never print the `>` characters, restyle the wording, or wrap it in a quote block. Where a step says to tell the publisher something, post it as your own message with nothing of your own before or after it. Copy given inline in double quotes is delivered the same way, without the quote marks.

- PUBLISHER_PLUGIN_VERSION = `0.0.17` (stamped by the release pipeline). Include it as `publisher_plugin_version` on every Publisher MCP tool call in this skill.
- USER_INPUT_TOOL = A tool that presents the user a question with selectable options and returns their choice (e.g. `AskUserQuestion`, `ask_user_input_v0`, etc.) that can be used in the default session state. Where a prompt below calls for USER_INPUT_TOOL and no such tool is available, ask the question in chat and wait for the reply.
- OS = If your system instructions indicate the platform is `darwin`, then `mac`; if `linux`, then `linux`; if `win32`, then `windows`.
- OPEN_URL_TOOL = A tool that opens a URL in a browser for the user: a dedicated one (e.g. `open_in_codex`) if present, else a shell command (e.g. Bash `open "<url>"`, Bash `xdg-open "<url>"`, PowerShell `Start-Process "<url>"`). Never a web search or page fetch.
- PUBLISHER_TOOLS_MISSING = If a Publisher MCP tool this skill needs is not in your tool catalog under any connector prefix (if not loaded, attempt to load it via tool search), the Publisher MCP server isn't connected: invoke the `plugpass-access-handler` skill and follow its instructions. When it returns after a successful connection, retry the call that needed the tool.
- ARGUMENTS = The text after the command, if any (the plugin's install page passes them as `{product name} {test user email}`): the last whitespace-separated token containing `@` is the test user's email, and whatever precedes it is the product, matched case-insensitively against `{products}` by value or label. An argument that matches nothing is ignored and its question asked as usual.

## Step 1: Confirm the plugin is ready to test

Read `.claude-plugin/plugin.json` (else `.codex-plugin/plugin.json`). `{manifest name}` is its `name`; `{plugin-plugpass-id}` is `metadata.plugpass-plugin-id`. If the id is absent or empty, tell the user "This plugin isn't registered with Plugpass yet. Run the `plugpass-sync-plugin` skill first." and end the skill.

Call `plugpass_get_plugin_data` with `plugin_id: {plugin-plugpass-id}`; keep its `plugin_slug` as `{plugin slug}`, its `plugin_display_name` as `{plugin display name}`, its `plugin_origin` as `{plugin origin}`, its `supported_products` as `{products}`, and its `owned_servers` as `{owned servers}` (the publisher's own MCP servers, each with its `.mcp.json` key `server_name` and its `url`).

Call `plugpass_list_test_users` with `plugin_id: {plugin-plugpass-id}` and keep the response as `{test users}`. If `versions.draft` is present and its `outstanding` has an entry whose `code` starts with `impl_`, tell the user "This plugin's premium features aren't implemented yet. Run the `plugpass-implement-code-changes` skill first." and end the skill. If `any_version_ready` is false, tell the user "No version of this plugin is ready to test yet. Finish the plugin's setup in the Plugpass dashboard first; the Publish page lists what's outstanding." and end the skill. `{version}` is `draft` when `versions.draft` is present and `ready`, else `production`; `versions.{version}.plans` are the plans step 3 offers, and `users` the existing test users.

## Step 2: Ask which AI product to test on

If the product was given in ARGUMENTS, has already been specified, or is otherwise known, skip this question. Otherwise ask with USER_INPUT_TOOL: "Which AI product do you want to test on?", offering each value of `{products}` by its label, in this order: `claude` Claude, `claude_code` Claude Code, `chatgpt_chat` ChatGPT Chat, `chatgpt_work` ChatGPT Work, `codex` Codex. Capture the answer's value as `{product}` and its label as `{product label}`.

## Step 3: Choose or create the test user

If the test user's email was given in ARGUMENTS, has already been specified, or is otherwise known, skip this question: an email among `{test users}` is that user, and any other email is a new one to create. Otherwise, if `{test users}` has any `users` whose `user_type` is `internal`, ask with USER_INPUT_TOOL: "Which test user do you want to test as?", offering each such user's email with its current state, plus "A new test user". Otherwise skip to creating one.

To create one, ask with USER_INPUT_TOOL for the email, offering the publisher's own email first when you know it, and "Another email". Then ask "What state should the test user start in?", offering "Not signed up", "Signed up" (only when no plan of `versions.{version}` has `is_free` true), and each plan of `versions.{version}` (a paid plan once per `intervals` entry, e.g. "Pro, monthly"). Call `plugpass_save_test_user` with `plugin_id`, `email`, `version: {version}`, `starting_status` (`not_signed_up`, `signed_up`, or `plan`), and for a plan `starting_plan_id` and `starting_term` (`monthly` or `annual`; omit for a free plan). When the user asks to test production while the draft is ready, `{version}` is `production` from here on.

If the user chose an existing test user and wants to start over, call `plugpass_reset_test_user` with `plugin_id` and `email`.

`{tested version}` is the chosen user's `resolved.kind`, or `{version}` for a new one. If the chosen user's `resolved.ready` is false, skip to step 5: the link call is refused, and its message names what to do.

## Step 4: Load the test build

When `{tested version}` is `production`, the published plugin is what is tested. Tell the user, then skip to step 5:

> If you already have the {plugin display name} plugin installed on {product label} then you can begin testing now. Or follow the [install instructions]({plugin origin}/install?product={product}) if you don't.

Otherwise follow the steps for `{product}`. These are the steps as last verified (2026-09-25); if one no longer matches what you see, find the current path rather than declaring it impossible. The real install of this plugin in that product is turned off for the duration and turned back on afterward.

**The test build.** When `{owned servers}` is empty, the test build is the working copy itself. When it is not, the test build is a **copy** of the working copy, made outside the repo, in which each owned server's `.mcp.json` entry (the key named by its `server_name`) has its `url` suffixed with `/test`, the server's test path; the repo is never written, and the copy is refreshed on every run. Make the copy with `rsync -a --delete --exclude .git --exclude node_modules --exclude .plugpass "$PWD/" "{copy path}/"` (PowerShell: `Copy-Item -Recurse -Force` with those folders excluded), removing a symlink already at the destination first (`unlink`), then `Edit` the copy's `.mcp.json`. `{build dir}` is the copy for a plugin with owned servers and the plugin directory otherwise.

Where a product takes an archive, build it from `{build dir}` yourself, leaving out `.git`, `node_modules`, and `.plugpass` directories (Bash: `zip -r <name>.zip . -x '.git/*' 'node_modules/*' '.plugpass/*'` from that directory; PowerShell: `Compress-Archive` over the same directory with those folders excluded); a copy for an archive goes in a temporary directory (`mktemp -d`).

- **`claude_code` (terminal and desktop app):** with no owned servers, run `mkdir -p ~/.claude/skills` then `ln -s "$PWD" ~/.claude/skills/{manifest name}` (PowerShell: `New-Item -ItemType SymbolicLink -Path "$HOME\.claude\skills\{manifest name}" -Target (Get-Location)`); with owned servers, the copy goes at `~/.claude/skills/{manifest name}` instead. If the plugin is also installed from a marketplace, run `claude plugin disable {manifest name}@<marketplace>` first and tell the user to run `claude plugin enable {manifest name}@<marketplace>` when they are done testing. Tell the user "Start a new Claude Code session to load the test build. Remove the {symlink | copy} at `~/.claude/skills/{manifest name}` when you're done testing."
- **`claude` (web and desktop, Cowork included):** build the archive, then tell the user "In Claude, open Customize, then Plugins, then Add, then Upload plugin, and upload `{archive path}`. It lists under Created by you. If the plugin is also installed from a marketplace, turn that copy off from its row menu while you test. To update the test build later, upload the same file again and confirm Replace existing plugin."
- **`chatgpt_chat` (ChatGPT on the web):** build the archive, then tell the user "In ChatGPT, open Plugins, click the plus button, choose Upload plugin, and upload `{archive path}`. The upload creates a record under Personal; install it from the plugin's page. If the real copy is installed, uninstall it from its page menu while you test. To update the test build later, use Upload new version from the plugin page's menu."
- **`chatgpt_work`:** on the web, as for `chatgpt_chat`; in the ChatGPT desktop app, as for `codex`.
- **`codex` (the Codex CLI and the ChatGPT desktop app):** with owned servers, put the copy at `~/.codex/plugins/{manifest name}`. Tell the user "Set up a local marketplace per OpenAI's manual install guide: put the plugin at `~/.codex/plugins/{manifest name}` with `~/.agents/plugins/marketplace.json` pointing at it, or run `codex plugin marketplace add` for the CLI, then restart the app. After edits, update the directory the entry points to and restart."

## Step 5: Open the Test as user link

Call `plugpass_test_as_user` with `plugin_id` and `email`; `{link}` is its `url`, single-use and valid for ten minutes. If the call is refused, relay its message to the user verbatim and end the skill.

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

> - Keep the test build current before each session.

When `{tested version}` is `draft` and `{owned servers}` is not empty, add:

> - The test build points your own MCP {server | servers} at {its | their} test path, which is what test mode uses. If you add {its | a server's} connector by URL, use the test path ({url}/test for each server), turn the real connector off while you test since the two carry the same tool names, and remove the test connector when you're done.

If `{test users}` has `has_unpublished_changes` true, add:

> When you're done testing, publish your changes in Plugpass to make them live.
>
> [Publish plugin](https://plugpass.ai/dashboard/plugin/{plugin slug}/publish)
