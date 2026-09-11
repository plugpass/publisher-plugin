---
name: plugpass-access-handler
description: Use this skill 1) when you are instructed to invoke it directly, 2) whenever any tool result includes PLUGPASS_PUBLISHER=true && UPDATE=recommended, or PLUGPASS_PUBLISHER=true && UPDATE=required.
user-invocable: false
allowed-tools: mcp__plugin_plugpass_plugpass-publisher__authenticate, AskUserQuestion, Bash(open:*), Bash(xdg-open:*), Bash(echo:*), Bash(claude:*), Bash(codex:*), PowerShell(Start-Process:*), PowerShell(Write-Output:*), Read, Skill(update-config)
---

USER_INPUT_TOOL = A tool that presents the user a question with selectable options and returns their choice (e.g. `AskUserQuestion`, `ask_user_input_v0`, etc.) that can be used in the default session state (not limited to a certain mode, e.g. plan mode).

TRIGGER_RESULT = the tool result already in context that triggered this skill, when one did — the Publisher MCP tool result carrying `PLUGPASS_PUBLISHER=true`.

Present each USER_INPUT_TOOL prompt exactly as specified, setting each option's description to an empty string. For a prompt posted as a `>` block, a reply of an option's number or label selects that option. The names of the prompts (e.g. `ConnectChoice`) are used for guiding your logic flow only, and should not be communicated to the user.

The `>` blocks and USER_INPUT_TOOL prompt copy are messages for the user; everything else here is an instruction for you (or the main agent) to follow. Replace each {VARIABLE} in them with its value; never state a variable in its token form. Post each `>` block verbatim as your own message — nothing rephrased, summarized, or added before, within, or after it; no quote wrapping or added formatting; never the `>` characters themselves. Use whatever messaging method will be visible to the user (especially if a tool call will follow it in the same turn).

Any variables defined by tool presence should be assessed purely from its presence in your tool list (if not loaded, attempt to load it via tool search); never attempt to call a tool if not present.

- NOT_CONNECTED = a `plugpass_sync_plugin` tool is present in your tool list (under any connector prefix) ? `false` : `true`
- PLATFORM = If your system instructions indicate an OpenAI product (Codex or ChatGPT), then `openai`; otherwise (an Anthropic / Claude product) `claude`.
- If PLATFORM=`claude`:
  - CLAUDE_PRODUCT = If your system instructions indicate your environment is Cowork, then `cowork`; if they indicate your environment is Claude Code, then `code`; otherwise `chat`.
  - If CLAUDE_PRODUCT=`code`:
    - OS = If your system instructions indicate the platform is `darwin`, then `mac`; if `linux`, then `linux`; if `win32`, then `windows`.
    - CODE_CLIENT = If (OS=`mac` || OS=`linux`), then Bash `echo "CLAUDE_CODE_ENTRYPOINT=$CLAUDE_CODE_ENTRYPOINT"`; if OS=`windows`, then PowerShell `Write-Output "CLAUDE_CODE_ENTRYPOINT=$env:CLAUDE_CODE_ENTRYPOINT"` (expected value: `cli` || `claude-desktop` || `remote`)
- If PLATFORM=`openai`:
  - OPENAI_CLIENT = If your system instructions include `# Codex desktop context`, then `desktop`; otherwise `codex-cli`.
- OPEN_URL_TOOL = A tool that opens a URL in a browser for the user: a dedicated one (e.g. `open_in_codex`) if present, else a shell command (e.g. Bash `open "<url>"`, Bash `xdg-open "<url>"`, PowerShell `Start-Process "<url>"`). Never a web search, page fetch, remote browser, or other context where the user can't access the URL's page.

If the OPEN_URL_TOOL fails to open the page for any reason (error, approval declined, user says it didn't open, etc.), then post the following message in the same turn:

> I couldn't open the page. [Open it here]({URL})

STANDING RULES (they govern every section below):

- Never proceed with the invoking skill on any non-authorized outcome (including NOT_CONNECTED — no tool could run), under any circumstances.
- **Retry:** If the user indicates they've completed the action that was blocking the work — by answering the confirmation prompt or in their own words — resume the invoking skill from the step that was blocked; the blocked tool call runs again.
- **Decline:** If the user declines — by choosing the negative option at any prompt below, or in their own words — Do not proceed with the invoking skill! Tell the user that they can run it again later if they change their mind.
- **Anything else:** If the user answers anything other than the presented options — Do not proceed with the invoking skill! Respond to the user's message as appropriate.

---

# If NOT_CONNECTED=`true`

## If OPENAI_CLIENT=`codex-cli`

Post the `ConnectOffer` prompt:

> Sign up or log in to Plugpass to continue.
>
> 1. Sign in
> 2. Not now

### If user answers `Sign in` to `ConnectOffer`

Run `codex mcp login plugpass-publisher`.

### If login succeeds

> End and resume the session with `codex resume` to continue.

Then post the `ResumeConfirm` prompt in the same turn:

>
> Have you resumed?
>
> 1. Yes
> 2. Never mind

### If user answers `Yes` to `ResumeConfirm`

Apply the Retry standing rule.

### If login fails

Inform the user and offer to try again or help them troubleshoot.

## If OPENAI_CLIENT=`desktop`

> Connect Plugpass to continue:
>
> 1. Open a terminal in the app with the shortcut: `` control ` `` (control + backtick)
> 2. Paste `codex mcp login plugpass-publisher` & press `Enter`
> 3. Sign up or log in to Plugpass

Then post the `ConnectConfirm` prompt in the same turn:

>
> Have you connected?
>
> 1. Yes
> 2. Not now

### If user answers `Yes` to `ConnectConfirm`

Apply the Retry standing rule.

## If CLAUDE_PRODUCT=`cowork`

> Sign up or log in to Plugpass to continue.
>
> [Sign up](https://plugpass.ai/signup?connect=web&platform=anthropic)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Log in](https://plugpass.ai/login?connect=web&platform=anthropic)

Then present the `ConnectConfirm` prompt with USER_INPUT_TOOL in the same turn:

- Prompt: Have you signed in?
- Options:
  - Yes
  - Not now

### If user answers `Yes` to `ConnectConfirm`

> Press `cmd-R` (`ctrl-R` on Windows) to refresh the session to continue.

Then present the `RefreshConfirm` prompt with USER_INPUT_TOOL in the same turn:

- Prompt: Have you refreshed?
- Options:
  - Yes
  - Cancel setup

### If user answers `Yes` to `RefreshConfirm`

Apply the Retry standing rule.

## If CODE_CLIENT=`cli`

Present the `ConnectChoice` prompt with USER_INPUT_TOOL:

- Prompt: Sign up or log in to Plugpass to continue.
- Options:
  - Sign up
  - Log in
  - Not now

### If user answers `Sign up` to `ConnectChoice`

AUTH_URL = the URL returned by `mcp__plugin_plugpass_plugpass-publisher__authenticate`, with `&mode=signup` appended.
Open {AUTH_URL} with the OPEN_URL_TOOL.

Then present the `SignInConfirm` prompt with USER_INPUT_TOOL in the same turn:

- Prompt: Have you signed up?
- Options:
  - Yes
  - Never mind

### If user answers `Log in` to `ConnectChoice`

AUTH_URL = the URL returned by `mcp__plugin_plugpass_plugpass-publisher__authenticate`, with `&mode=login` appended.
Open {AUTH_URL} with the OPEN_URL_TOOL.

Then present the `SignInConfirm` prompt with USER_INPUT_TOOL in the same turn:

- Prompt: Have you logged in?
- Options:
  - Yes
  - Never mind

### If user answers `Yes` to `SignInConfirm`

Apply the Retry standing rule.

## If CODE_CLIENT=`claude-desktop`

> Sign up or log in to Plugpass to continue.
>
> Enter `/mcp`, then connect `plugpass-publisher` to sign in.

Then post the `ConnectConfirm` prompt in the same turn (not via USER_INPUT_TOOL):

>
> Have you signed in?
>
> 1. Yes
> 2. Not now

### If user answers `Yes` to `ConnectConfirm`

Apply the Retry standing rule.

## If CODE_CLIENT=`remote`

> Sign up or log in to Plugpass to continue.
>
> [Sign up](https://plugpass.ai/signup?connect=web&platform=anthropic)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Log in](https://plugpass.ai/login?connect=web&platform=anthropic)
>
> Once you've signed in, start a new session to continue.

Do not proceed with the invoking skill!

---

# If UPDATE=`required` in TRIGGER_RESULT

The Publisher MCP refused the call: the installed plugin is below the minimum supported version, so the invoking skill cannot proceed until the plugin is updated. INSTALLED_VERSION and MINIMUM_VERSION are the corresponding lines of TRIGGER_RESULT.

## If CODE_CLIENT=`cli`

> The installed Plugpass plugin version ({INSTALLED_VERSION}) is no longer supported.

Then present the `UpdateChoice` prompt with USER_INPUT_TOOL in the same turn:

- Prompt: Update it now?
- Options:
  - Update
  - Not now

### If user answers `Update` to `UpdateChoice`

Run `claude plugin marketplace update plugpass-marketplace`.

> Run `/reload-plugins` to finish updating.

Then post the `ReloadConfirm` prompt in the same turn (not via USER_INPUT_TOOL):

>
> Have you reloaded?
>
> 1. Yes
> 2. Never mind

### If user answers `Yes` to `ReloadConfirm`

> Turn on auto-update for the Plugpass Marketplace to keep the plugin up to date with the latest features & fixes.

Then present the `AutoUpdateChoice` prompt with USER_INPUT_TOOL in the same turn:

- Prompt: Turn on auto-update?
- Options:
  - Turn on
  - Not now

### If user answers `Turn on` to `AutoUpdateChoice`

Invoke the `update-config` skill to set `extraKnownMarketplaces.plugpass-marketplace` in `~/.claude/settings.json` to `{ "source": { "source": "github", "repo": "plugpass/plugpass-marketplace" }, "autoUpdate": true }`.

### If user answers (`Turn on` || `Not now`) to `AutoUpdateChoice`

Apply the Retry standing rule.

## If CODE_CLIENT=`claude-desktop`

Read `~/.claude/settings.json`.

### If the `extraKnownMarketplaces.plugpass-marketplace` entry (or the file) is absent

> The installed Plugpass plugin version ({INSTALLED_VERSION}) is no longer supported.
>
> [View update instructions](https://plugpass.ai/update)

Then present the `UpdateConfirm` prompt with USER_INPUT_TOOL in the same turn:

- Prompt: Have you updated?
- Options:
  - Yes
  - Not now

#### If user answers `Yes` to `UpdateConfirm`

> Press `cmd-R` (`ctrl-R` on Windows) to refresh the session to finish updating.

Then present the `ReloadConfirm` prompt with USER_INPUT_TOOL in the same turn:

- Prompt: Have you refreshed?
- Options:
  - Yes
  - Never mind

#### If user answers `Yes` to `ReloadConfirm`

Apply the Retry standing rule.

### If the entry is present

> The installed Plugpass plugin version ({INSTALLED_VERSION}) is no longer supported.

Then present the `UpdateChoice` prompt with USER_INPUT_TOOL in the same turn:

- Prompt: Update it now?
- Options:
  - Update
  - Not now

#### If user answers `Update` to `UpdateChoice`

Run `claude plugin marketplace update plugpass-marketplace`.

> Press `cmd-R` (`ctrl-R` on Windows) to refresh the session to finish updating.

Then present the `ReloadConfirm` prompt with USER_INPUT_TOOL in the same turn:

- Prompt: Have you refreshed?
- Options:
  - Yes
  - Never mind

#### If user answers `Yes` to `ReloadConfirm`

> Turn on auto-update for the Plugpass Marketplace to keep the plugin up to date with the latest features & fixes.

Then present the `AutoUpdateChoice` prompt with USER_INPUT_TOOL in the same turn:

- Prompt: Turn on auto-update?
- Options:
  - Turn on
  - Not now

#### If user answers `Turn on` to `AutoUpdateChoice`

Invoke the `update-config` skill to set `extraKnownMarketplaces.plugpass-marketplace` in `~/.claude/settings.json` to `{ "source": { "source": "github", "repo": "plugpass/plugpass-marketplace" }, "autoUpdate": true }`.

#### If user answers (`Turn on` || `Not now`) to `AutoUpdateChoice`

Apply the Retry standing rule.

## If CODE_CLIENT=`remote`

> The installed Plugpass plugin version ({INSTALLED_VERSION}) is no longer supported.
>
> [View update instructions](https://plugpass.ai/update)
>
> Once you've updated, start a new session and try again.

Do not proceed with the invoking skill!

## If OPENAI_CLIENT=`codex-cli`

> The installed Plugpass plugin version ({INSTALLED_VERSION}) is no longer supported.
>
> End and resume the session with `codex resume` to update the Plugpass plugin.

Then post the `UpdateConfirm` prompt in the same turn:

>
> Have you updated?
>
> 1. Yes
> 2. Not now

### If user answers `Yes` to `UpdateConfirm`

Apply the Retry standing rule.

## If OPENAI_CLIENT=`desktop`

> The installed Plugpass plugin version ({INSTALLED_VERSION}) is no longer supported.
>
> Restart the Codex app to update the Plugpass plugin.

Then post the `UpdateConfirm` prompt in the same turn:

>
> Have you updated?
>
> 1. Yes
> 2. Not now

### If user answers `Yes` to `UpdateConfirm`

Apply the Retry standing rule.

---

# If UPDATE=`recommended` in TRIGGER_RESULT

`UPDATE=recommended` only ever rides a successful tool result, so the invoking skill's work is never blocked. Complete the invoking skill's work first, then run this nudge — at most once per session. Unlike every other case, never say "Do not proceed with the invoking skill" here; the work is already done.

## If CODE_CLIENT=`cli`

> The Plugpass plugin is out of date.
>
> Turn on auto-update for the Plugpass Marketplace to keep the plugin up to date with the latest features & fixes.

Then present the `AutoUpdateChoice` prompt with USER_INPUT_TOOL in the same turn:

- Prompt: Turn on auto-update?
- Options:
  - Turn on
  - Not now

### If user answers `Turn on` to `AutoUpdateChoice`

Invoke the `update-config` skill to set `extraKnownMarketplaces.plugpass-marketplace` in `~/.claude/settings.json` to `{ "source": { "source": "github", "repo": "plugpass/plugpass-marketplace" }, "autoUpdate": true }`.

### If user answers `Not now` to `AutoUpdateChoice`

Do nothing further.

### If user answers anything other than (`Turn on` || `Not now`) to `AutoUpdateChoice`

Respond to the user's message as appropriate.

## If CODE_CLIENT=`claude-desktop`

Read `~/.claude/settings.json`. If it contains an `extraKnownMarketplaces.plugpass-marketplace` entry, follow the `If CODE_CLIENT=cli` section's instructions; if the entry or file is absent, follow the `If CODE_CLIENT=remote` section's instructions.

## If CODE_CLIENT=`remote`

> The Plugpass plugin is out of date.
>
> [View update instructions](https://plugpass.ai/update)

## If PLATFORM=`openai`

Do nothing — Codex updates configured marketplaces at every app start, so a stale bundle heals on the next launch without any action.
