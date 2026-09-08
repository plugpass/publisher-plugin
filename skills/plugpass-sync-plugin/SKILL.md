---
name: plugpass-sync-plugin
description: >
  Registers a plugin with Plugpass, or syncs an existing one.

  The plugin's skills and MCP tools are automatically added, renamed, or removed
  in Plugpass, so they can be subsequently mapped to subscription plans in the
  Plugpass dashboard.
allowed-tools: mcp__plugin_plugpass_plugpass-publisher__plugpass_sync_plugin, mcp__plugin_plugpass_plugpass-publisher__plugpass_get_plugin_data, Read, Write, Edit, Glob, Grep, Bash(open:*), Bash(xdg-open:*), Bash(echo:*), Bash(git remote:*), Bash(git rev-parse:*), PowerShell(Start-Process:*), PowerShell(Write-Output:*), PowerShell(git remote:*), PowerShell(git rev-parse:*), AskUserQuestion, Skill
---

This skill works in the publisher's plugin repo: it reads and edits files in the current working directory and runs terminal commands there. If your environment cannot do both, tell the user "Plugpass needs to read and edit files in your plugin's repo and run terminal commands, which isn't possible here. Open your plugin's directory in an AI coding tool like Claude Code or Codex and run this skill again." and end the skill.

## What this skill does

One skill for both jobs: first-time registration of a plugin with Plugpass, and re-syncing an already-registered one. The full local snapshot — plugin metadata, skills, mcp_dependency entries, and publisher-confirmed MCP server tools — flows in a single `plugpass_sync_plugin` call; the server reconciles atomically (creating the plugin on first registration; diffing the snapshot against the current draft and applying adds / renames / removals on re-sync) and returns plugpass-ids for the plugin and every component. The skill writes those ids back to source files so renames are detectable across runs — a renamed skill or tool stays linked to the same Plugpass-side entitlement configuration — then opens the dashboard page the server chose as the follow-up surface.

Step 0 determines the run's MODE; later steps call out their mode-specific behavior:

- **REGISTER** — the plugin isn't linked to Plugpass in this working copy (first-time setup, a fresh checkout, or a wiped/regenerated manifest). No server-side state is consulted; every elicitation runs fresh.
- **SYNC** — the plugin carries a `plugpass-plugin-id`; server-side state pre-checks the elicitations so the publisher is never re-asked what Plugpass already knows.

The Plugpass Publisher MCP server (this plugin's `.mcp.json` `plugpass-publisher` entry) provides the `plugpass_sync_plugin` and `plugpass_get_plugin_data` tools this skill calls — reference them by those bare names under any connector prefix.

**Work silently.** The only text you post is what a step calls for. Skip the short preamble that normally precedes a tool call — including the first one — and post no mode or step announcements, no commentary on what you just did or are about to do, and no recap the steps didn't ask for.

**Presenting copy.** A `>` block is finished copy; the `>` characters delimit it here and are never part of it. Reproduce the text exactly — substituting each `{VARIABLE}` with its value — and never print the `>` characters, restyle the wording, or wrap it in a quote block. The surrounding step says where the copy goes: where it says to tell the publisher something, post it as your own normal assistant message with nothing of your own before or after it. Copy given inline in double quotes is delivered the same way, without the quote marks.

- PUBLISHER_PLUGIN_VERSION = `0.0.3` (stamped by the release pipeline). Include it as `publisher_plugin_version` on every Publisher MCP tool call in this skill.
- USER_INPUT_TOOL = A tool that presents the user a question with selectable options and returns their choice (e.g. `AskUserQuestion`, `ask_user_input_v0`, etc.) that can be used in the default session state (not limited to a certain mode, e.g. plan mode). Where a prompt below calls for USER_INPUT_TOOL and no such tool is available, ask the question in chat and wait for the reply.
- PLATFORM = If your system instructions indicate an OpenAI product (Codex or ChatGPT), then `openai`; otherwise (an Anthropic / Claude product) `claude`.
- OS = If your system instructions indicate the platform is `darwin`, then `mac`; if `linux`, then `linux`; if `win32`, then `windows`.
- If PLATFORM=`claude`: CODE_CLIENT = If (OS=`mac` || OS=`linux`), then Bash `echo "CLAUDE_CODE_ENTRYPOINT=$CLAUDE_CODE_ENTRYPOINT"`; if OS=`windows`, then PowerShell `Write-Output "CLAUDE_CODE_ENTRYPOINT=$env:CLAUDE_CODE_ENTRYPOINT"` (expected value: `cli` || `claude-desktop` || `remote`)
- OPEN_URL_TOOL = If OS=`mac`, then Bash `open "<url>"`; if OS=`linux`, then Bash `xdg-open "<url>"`; if OS=`windows`, then PowerShell `Start-Process "<url>"`. (OPEN_URL_TOOL is undefined when CODE_CLIENT=`remote` — no local browser to open; where a step says to open a URL, present it as a markdown link instead.)
- PUBLISHER_TOOLS_MISSING = If a Publisher MCP tool this skill needs is not in your tool catalog under any connector prefix (if not loaded, attempt to load it via tool search), the Publisher MCP server isn't connected: invoke the `plugpass-access-handler` skill and follow its instructions. When it returns after a successful connection, retry the call that needed the tool.

## Step 0: Mode check

Check the current working directory (the publisher's plugin repo root) for both plugin manifests: `.claude-plugin/plugin.json` (family `claude`) and `.codex-plugin/plugin.json` (family `codex`). A plugin repo may carry either or both. `{manifests}` is the list of families present, and step 7 writes the plugpass-plugin-id back to every one of them. If neither exists, tell the user "No Claude or Codex plugin manifest found. Run this skill again from inside your plugin repo." and end the skill.

`{manifest}` — where this skill reads the plugin's metadata — is `.claude-plugin/plugin.json` when present, else `.codex-plugin/plugin.json`. Read it.

Check `{manifest}`'s `metadata.plugpass-plugin-id` field:

- Set to a non-empty value → **SYNC mode**. Capture the value as `{plugin-plugpass-id}`.
- Absent or empty → **REGISTER mode**. (If the plugin is actually registered server-side under this manifest `name` — e.g., the manifest was wiped or regenerated — `plugpass_sync_plugin` relinks to the existing registration automatically; REGISTER mode needs no special handling for that case, and you must not announce or guess whether the plugin is already registered — you don't know.)

## Step 1: Extract plugin metadata

From `{manifest}`, extract:

- `name` — required.
- `displayName` — the manifest's human-readable display name, if present and non-empty: the top-level `displayName` in `.claude-plugin/plugin.json`, or `interface.displayName` in `.codex-plugin/plugin.json`. When the manifest carries it, the manifest is authoritative: it seeds the plugin's Plugpass display name on create and is re-written on every re-sync. When the manifest omits it, the display name is title-cased from `name` on create and is publisher-owned on the dashboard afterward — re-sync never touches it.
- `description` — may be absent.
- `version` — informational only.
- `license` — the SPDX license id (e.g. `MIT`), if present; may be absent. Pass it through to `plugpass_sync_plugin` so Plugpass can offer "keep your existing license" in the dashboard.

Derive `slug` from `name`: lowercase, ASCII alphanumeric with `-` as the only separator, strip leading/trailing separators, collapse runs of separators into one (e.g. `My Plugin!` → `my-plugin`). The slug seeds the plugin's public URL on create; server-side it is **locked** for already-registered plugins — on SYNC runs `plugpass_sync_plugin` ignores the value and preserves the stored slug. Slug edits happen on the dashboard only.

## Step 1b: Detect the plugin repository

Propose the plugin's public repository and its directory in it. It is a proposal: the publisher confirms and verifies it on the dashboard's Distribution page.

- `repository` — `{manifest}`'s `repository` field when present; otherwise the `origin` remote from `git remote get-url origin` (Bash, or PowerShell on Windows). Omit the whole `source` object when neither exists.
- `subdirectory` — the plugin directory's path from the repository root, from `git rev-parse --show-prefix` run in the plugin directory, without its trailing slash (`""` when the plugin is the repository root). Omit the `source` object when the directory isn't in a git repository.

## Step 2: Discover local skill components

Use `Glob` to find every component file in the plugin repo:

- Skills: `skills/*/SKILL.md`

**Drop the plugin's access-handler skill** (`skills/{plugin-name}-access-handler/SKILL.md` — its name is the manifest `name` + `-access-handler` — or a legacy `skills/premium_check_cases/`) from the discovered skill list before anything else — it's Plugpass infrastructure that `plugpass-implement-code-changes` writes into the plugin, never a publisher component, so it must never be registered as a `skill` (it would otherwise surface in `/plans`, the pitch flow, etc.). Same recognize-and-filter shape as the Plugpass-platform-owned MCP-server filter in step 3. On an already-monetized plugin this skill is normally present, so the filter matters on every SYNC run; on a true first-time REGISTER it's normally absent, and the filter is defensive against re-runs and already-monetized sources.

For each file, use `Read` to load it and parse the YAML frontmatter (the block between the leading `---` and the next `---`). Extract:

- `name` — required; if absent, fall back to the directory or file name without the extension.
- `metadata.plugpass-component-id` — **SYNC mode only**: the existing Plugpass id from a prior run; may be absent if the component was added since. The plugpass_id (when present) is what makes rename detection work — `plugpass_sync_plugin` matches by id first and the canonical row's name gets updated to the current local name. **In REGISTER mode, ignore this field even if present** — a stale or copied id must not ride the payload; the server's name-fallback matching handles any relink.

Track the file path next to each component so step 8 can write back to it, and keep each file's `description` + body content on hand — step 5 drafts pitch suggestions from them.

## Step 3: Discover local mcp_dependency components

Read `.mcp.json` at the plugin root (when `{manifest}` is `.codex-plugin/plugin.json` and its `mcpServers` field names a different path, read that file instead; an inline `mcpServers` object there is the servers map itself). If the file is absent or its `mcpServers` object is empty, the discovered tool list and mcp_dependency list are both empty — skip to step 5 (pitch drafting still runs for the skills) with no MCP-server content in the payload.

Otherwise, parse `mcpServers`. **Before doing anything else with the parsed entries, filter out Plugpass-served MCP servers** — the platform's own hosts are never publisher-owned `mcp_dependencies` (connector audiences derive from plugin state, not from per-plugin source). Drop any `mcpServers` entry whose URL host (host:port for localhost entries) matches one of:

- `plugpass.ai` or ANY `*.plugpass.ai` subdomain — every Plugpass-hosted plugin connector lives at `{slug}.plugpass.ai` (typically under the plugin's own manifest-name key, placed by `plugpass-implement-code-changes`), and the Publisher MCP at `publisher.plugpass.ai` (included only in the `plugpass` publisher plugin)
- `localhost:3000` or any `*.localhost:3000` subdomain (dev-mode apex + plugin-connector host forms)
- `localhost:8788` (the retired dev-mode shared end-user MCP — older monetized sources may still carry it)
- `localhost:8789` (dev-mode Publisher MCP)

**SYNC mode only, once step 4's pre-check response is in hand:** additionally drop any parsed entry whose URL host equals the host of the pre-check's `plugin_origin` — the plugin's connector served on its active custom domain, which host shape alone can't recognize. (In both modes `plugpass_sync_plugin` rejects these server-side as a backstop — if a sync fails naming a domain as "a plugin's active custom domain", drop that entry and re-send.)

These are platform-served hosts the publisher cannot monetize; they never surface in the step 4 elicitation and never appear as `mcp_dependency` components in the `plugpass_sync_plugin` payload. If filtering leaves zero remaining `mcpServers` entries, step 4 is skipped just as it would be for a plugin with no `.mcp.json`.

For each remaining entry (after filtering), emit an `mcp_dependency` component:

- `name` — the server key (the property name in `mcpServers`).
- `domain` — the bare hostname extracted from the server's `url` field. Take the substring between `://` and the next `/`, then strip any trailing `:{port}`. Examples: `"url": "https://mcp.example.com/mcp"` → `mcp.example.com`; `"url": "http://localhost:9000/mcp"` → `localhost`. Required for `http` and `sse` transport entries (any entry with a `url` field). For stdio entries (`command` set, no `url`), omit the `domain` field.
- `url` — the entry's `url` field verbatim (the same value `domain` was extracted from). Required alongside `domain`; omitted for stdio entries. The server cross-checks it against `domain` and enforces the owned-server URL shape below.

`mcp_dependency` entries have no source-file slot for plugpass-id (`.mcp.json` has no per-server metadata block). On SYNC runs `plugpass_sync_plugin` matches them by `(plugin_id, type, name)` — i.e., server_name.

## Step 4: Ownership + premium tool elicitation

If step 3 produced no `mcp_dependency` entries (no `.mcp.json` or empty `mcpServers`, post-Plugpass-filter), skip this entire step — including the SYNC-mode pre-check below. The tool list is empty.

### Pre-check: fetch current entitlement state (SYNC mode only)

In SYNC mode, call the `plugpass_get_plugin_data` tool (under any connector prefix) with the plugin id from step 0 (PUBLISHER_TOOLS_MISSING applies if the tool isn't in your catalog):

```json
{ "plugin_id": "{plugin-plugpass-id}", "publisher_plugin_version": "{PUBLISHER_PLUGIN_VERSION}" }
```

Capture the response. From the `components` array:

- **`mcp_dependency` entries** carry per-server `ownership` (`unknown` / `owned` / `not_owned`) and `has_ungated_tools` (true when the server has at least one solo canonical tool that isn't already paid, OR when the server has no canonical tools at all yet). Build a `server_name → { plugpass_id, ownership, has_ungated_tools }` map — Q1 and Q2 below consume it.
- **`tool` entries** carry the existing per-tool `database_record` subfield + `operation`. Build a `(server_name, tool_name) → { plugpass_id, database_record, operation }` map for tool enumeration's re-run identity match and paired-tool detection's pre-confirmation.

**In REGISTER mode, skip the call** — there is no plugin id to query. Treat every server's effective state as empty: ownership `unknown`, `has_ungated_tools` true, no tool in any tier, no pre-confirmed pairs, and an empty plugpass-id map.

### Question 1 — ownership

Asked for every server whose **effective** ownership is `unknown`: in REGISTER mode that's every entry from step 3; in SYNC mode only servers whose server-side `ownership` is `unknown` (or which don't exist server-side at all — newly-discovered). Servers already `owned` or `not_owned` server-side are NOT asked again — the existing value is preserved (omit the `ownership` field in their payload entry). If the set is empty, skip Q1 entirely.

**Format by count:**

- **2 or more servers** — multi-select via USER_INPUT_TOOL (multi-select mode). **Use this exact question text verbatim**:

  > Do you own & control any of the following MCP servers? If any are provided by a third-party (e.g. Gmail, etc), don't include them.

  Each server is one option, labeled by the server name (the `mcpServers` key verbatim).

- **Exactly 1 server** — yes/no via USER_INPUT_TOOL. **Use this exact question text verbatim**, interpolating the server name:

  > Do you own & control the {server-name} MCP server? If it's provided by a third-party (e.g. Gmail, etc), answer 'No'.

  Options: "Yes" and "No". Treat "Yes" as owned, "No" as not_owned.

- **5 or more servers** — same multi-select copy as the 2+ case, split into multiple questions of up to 4 servers each. Aggregate the checkbox results.

Record each answered server's value for the `plugpass_sync_plugin` payload's `mcp_dependency.ownership` field (`owned` or `not_owned`).

### Shared-hosting domain check

Run immediately after Q1, before the URL shape check. For every server whose **effective** ownership is `owned`, skipping local-development hosts (`localhost`, `*.localhost`, `*.local`, or a private-range IP): its domain must not sit under a shared-hosting root.

**Shared-hosting roots:**

- `workers.dev`
- `vercel.app`
- `netlify.app`
- `pages.dev`
- `github.io`
- `herokuapp.com`

If one does, tell the publisher:

> Your {server-name} MCP server is hosted on a shared-hosting domain ({root-domain}). You need to move your MCP server to a domain you own to continue. Let me know what domain you want to use and I'll help you update your MCP server and plugin code.

Then help them fix it: point the server at the domain they name, deploy it, and update that server's `.mcp.json` `url`. Do not stop the run or ask them to re-run this skill. Continue with the remaining instructions once the server is serving on their own domain.

### Owned-server https check

Validate the same set of owned servers (effective ownership `owned`, local-development hosts skipped): the `.mcp.json` `url` must be `https`. Any path and port are fine.

If one isn't https, tell the publisher:

> The {server-name} MCP server is registered at {url}. A server you own has to be served over https. Update it to https, update `.mcp.json` to match, and I'll continue.

Then help them fix it in place, as above.

### Question 2 — tool restriction

Compute the eligible set: a server qualifies for Q2 iff its **effective** ownership is `owned` (just answered Yes in Q1, or — SYNC mode — already `owned` server-side) AND its effective `has_ungated_tools` is true. In REGISTER mode every owned server qualifies. Servers marked `not_owned` are skipped — neither their tools nor any restriction question apply. If the eligible set is empty, skip Q2 entirely.

An effectively-owned server whose `has_ungated_tools` is false — every tool it has is already restricted — is **not asked and counts as checked**: the publisher restricted it on a prior run, and its tools are enumerated below exactly as a checked server's are. (Leaving it out would send a payload with none of its tools, which reconciles as removing every one of them.)

**Format by count of eligible servers:**

- **2 or more eligible servers** — multi-select via USER_INPUT_TOOL (multi-select mode). **Use this exact question text verbatim**:

  > Do you want to limit the usage of any of the tools provided by the following MCP servers to paid plans? E.g. free users can only use certain tools, or only can use certain tools a limited number of times.

  Each eligible server is one option, labeled by the server name.

- **Exactly 1 eligible server** — yes/no via USER_INPUT_TOOL. **Use this exact question text verbatim**, interpolating the server name:

  > Do you want to limit the usage of any of the tools provided by the {server-name} MCP server to paid plans? E.g. free users can only use certain tools, or only can use certain tools a limited number of times.

  Options: "Yes" and "No". Treat "Yes" as the server being checked, "No" as unchecked.

- **5 or more eligible servers** — same multi-select copy, split into multiple questions of up to 4 servers each.

**Pre-check state.** In REGISTER mode, no server is pre-checked — the publisher's selection drives which servers participate in tool enumeration. In SYNC mode, a server option is pre-checked (or, for the single-server case, the Yes/No default leans toward "Yes") if the server currently has paid tools — i.e., any of its tools appear with `in_any_tier: true` in the pre-check response; the publisher can uncheck to remove all of that server's tools from the new sync.

Servers marked `not_owned` in Q1 still get an `mcp_dependency` entry in the payload (with `ownership: 'not_owned'`) — they're persisted so we know not to re-ask on the next run — but they contribute no tools. Servers whose effective ownership is `owned` but that are unchecked in Q2 also contribute no tools; they stay free on a first run, and on a re-run their previously-registered tools are dropped from Plugpass.

The **enumerated set** is every server checked in Q2 plus every effectively-owned server not asked because all its tools are already restricted. Steps below run over that set.

### For each enumerated server: locate the source

Tools are enumerated from the publisher's own MCP server source, so its location is resolved first — and Step 8 stamps each tool's plugpass id back into that same source. Per-server paths are stored in the plugin repo's gitignored `.plugpass/` directory (per-machine, never shared — each collaborator keeps their own):

```
.plugpass/mcp-server-paths.json
```

(relative to the plugin repo root) — a JSON object mapping `server-name → absolute path to the server's source directory`.

`Read` it (a missing file is an empty map `{}`). For each enumerated server: if the map already has a path that still resolves (the directory exists), keep it — a prior run usually recorded it; the server's source living inside the plugin repo resolves it too. Otherwise — a newly-monetized server, or the source moved — elicit the **absolute** path with USER_INPUT_TOOL, one server at a time, then `Write` the merged map back (creating `.plugpass/` if absent). Absolute paths only, since this file is per-publisher and never shared (a relative path would be meaningless on a collaborator's machine). After writing, ensure the plugin repo's `.gitignore` has a `.plugpass/` line — append one if it's missing (skip when the plugin directory isn't in a git repo).

Prompt: "I'm having trouble locating the source directory for the {server-name} MCP server. Please provide its absolute path."

The path is **required**. If the publisher can't or won't provide one that resolves, stop: make no `plugpass_sync_plugin` call at all, tell them

> I can't read the tools on your {server-name} MCP server without its source. Tell me where that server's source directory is to continue.

and end the skill. Syncing the rest and leaving that server's tools out would silently un-register them.

### For each enumerated server: enumerate tools from its source

Read the server's source and collect every tool it registers. Start from the entry file and follow its setup path, and `Grep` the source tree for the SDK's registration idiom to catch tools defined elsewhere — `registerTool` / `server.tool` / `add_tool`, a `@mcp.tool` decorator or `#[tool]` attribute, a tool struct or hash carrying a `name` field, a builder. Every SDK carries the tool's registered name, description, and metadata block together in one registration. A tool registered behind a condition (only when the client declares UI support, say) is still one of the server's tools — collect it.

For each tool, capture:

- `tool_name` — the name the server registers the tool under (the string clients call), never the function or handler name.
- `description` — the registered description (paired-tool detection and pitch drafting below read it, alongside the handler's own code).
- `plugpass_id` — the registration's `_meta.plugpass_component_id`, when it carries one. Absent for a tool that has never been synced; in REGISTER mode ignore any value found, exactly as step 2 ignores stale skill ids.
- `visibility` — from the registration's `_meta.ui.visibility`: `model` when it lists only `"model"`, `app` when it lists only `"app"`, and `both` when it lists both or the field is absent (absent is the MCP Apps default, which is both).
- `ui_backed` — `true` when the registration carries `_meta.ui.resourceUri` (the tool renders a UI resource), `false` otherwise.

The metadata block's field name follows the SDK (`_meta`, `meta`, `Meta`); the keys inside it are the wire names above.

If the same `tool_name` is registered more than once (per-client variants), it is one tool: `ui_backed` is true if any variant declares a resource, and `visibility` is the union of the variants'.

**Drop the plugin's Plugpass-owned tools** — `{plugin_name}_check_access`, where `{plugin_name}` is the manifest `name` with every `-` replaced by `_` — from the collected list. That tool is Plugpass infrastructure `plugpass-implement-code-changes` registers on the publisher's server, never a publisher component, so it must never be registered as a `tool` (it would otherwise surface in `/plans`, the pitch flow, etc.). Same recognize-and-filter shape as the platform-owned MCP-server filter in step 3.

If you can't find the server's tool registrations at all, treat it exactly as a missing source path above: stop, say so naming that server, and end the skill.

This reads source, not your tool catalog: a tool the server exposes only to a UI is absent from the catalog by design, and the server does not need to be connected in this session.

### For each enumerated server: paired-tool detection

For each enumerated server's tool set, identify candidate **paired tools** — two tools on the same server where one adds a database record to some collection in the publisher's database and the other removes a database record of that same kind. Common examples: `connect_brokerage` ↔ `disconnect_brokerage`, `add_seat` ↔ `remove_seat`, `task_add` ↔ `task_remove`, `subscribe` ↔ `unsubscribe`, `lock_door` ↔ `unlock_door`.

There is no fixed verb table or required name shape — the publisher may have named their tools in any natural style (`verb_noun`, `noun_verb`, `verbNoun`, single-word inverses, etc.). Judge from the tool names, their descriptions, and what their handlers actually do in the source you just read. A pair's two sides need not share a visibility — a widget-driven add and a model-facing remove act on the same records. When in doubt, propose the pair to the publisher in prompt 1 below — they confirm or reject it; one-off false positives are cheap to reject.

**Pre-confirmation (SYNC mode only).** A candidate pair is **pre-confirmed** if both tools in the pre-check response carry a `database_record` subfield referencing the same canonical database_record plugpass_id (with opposite `operation` values). Skip both prompts for pre-confirmed pairs — the publisher already confirmed them on a prior run. Pass the existing `database_record`'s `{ plugpass_id, name, title }` through unchanged in the `plugpass_sync_plugin` payload so server reconciliation updates the existing row in place rather than creating a duplicate.

For candidates that are NOT pre-confirmed (all candidates in REGISTER mode; newly detected pairs, or pairs the publisher rejected last time, in SYNC mode), surface **two** USER_INPUT_TOOL prompts in sequence (one pair = up to two prompts):

**Prompt 1 — pair confirmation:**

> Do the {add_tool_name} & {remove_tool_name} tools both act on the same type of database record? E.g. one adds something & the other removes the same type of thing.

Options: "Yes" / "No, treat as independent tools".

If the publisher answers "No", drop the candidate; both tools are treated as solo (no `database_record` entry, no paired routing on either `tool` entry). Skip to the next candidate.

If "Yes", proceed to prompt 2.

**Prompt 2 — database record name:**

> What do you want to name the database records?

Offer the AI's proposed sentence-case plural name as the recommended option, plus one alternate phrasing the AI thought of. Example for `add_brainstorm_topic` ↔ `remove_brainstorm_topic`: "Brainstorm topics" (preferred) and "Topics" (alternate). The auto-appended "Other" option (where the tool provides one) handles full free-text override.

The publisher's chosen string becomes the database record's `title`. The AI then derives `name` by:

1. Lowercase the title.
2. Replace runs of non-alphanumeric characters with a single underscore.
3. Strip leading and trailing underscores.

Example: "Brainstorm topics" → `brainstorm_topics`. "Saved chart views!" → `saved_chart_views`. The server re-derives `name` from `title` using the same rule and rejects the call if the AI's `name` doesn't match — clients shouldn't trust their own derivation when ours is canonical.

For each confirmed pair (pre-confirmed or newly confirmed), emit:

- One `database_record` component entry carrying `name` + `title` (`plugpass_id` present for pre-confirmed pairs, omitted for newly-confirmed ones).
- The two paired `tool` entries each carry `database_record_name` (matching the `database_record` entry's `name`) and `operation` (`add` for the tool that adds a database record, `remove` for the tool that removes a database record).

**SYNC mode:** for previously-confirmed pairs that are no longer detected (e.g., the publisher renamed one of the tools so the pattern no longer matches) or that the publisher un-confirms this run, omit the `database_record` entry and omit `database_record_name` + `operation` on both partner tool entries. `plugpass_sync_plugin` reconciles by clearing `database_record_id` + `operation` on both partner `mcp_tools` rows and deleting the orphaned `database_records` canonical row (cascading through any `custom_entitlements` row that referenced it).

### Assemble the tool component list

For each tool on each enumerated server, emit a `tool` component:

- `server_name` — the server key (`mcpServers` key verbatim).
- `tool_name` — the name the server registers the tool under.
- `visibility` — `model`, `app`, or `both`, as read from the registration. Always sent.
- `ui_backed` — `true` or `false`, as read from the registration. Always sent.
- `database_record_name` — set only when this tool is part of a confirmed pair (references the `database_record` entry in the same call by name); omit otherwise.
- `operation` — set only when this tool is part of a confirmed pair (`add` for the tool that adds a database record, `remove` for the tool that removes a database record); omit otherwise. `database_record_name` and `operation` move together — either both present (paired) or both omitted (solo).
- `plugpass_id` — SYNC mode: from the registration's `_meta.plugpass_component_id`, else the pre-check map, if either has one; omitted otherwise. Always absent in REGISTER mode.

**All discovered `mcp_dependency` entries from step 3 are included in the payload regardless of Q2 outcome** — they're how Plugpass persists per-server `ownership` and tracks dependency identity. Entries for `not_owned` servers carry their ownership and never contribute tools. Entries for effectively-owned servers contribute tools only when they're in the enumerated set.

## Step 5: Draft suggested pitches

Draft one suggested pitch per component going into the payload — skills, tools on enumerated servers, and database records. The drafts ride the `plugpass_sync_plugin` payload as `suggested_pitch` fields; they are suggestions only, pre-filling the dashboard's features page where the publisher reviews, edits, and confirms them. **Don't show the drafts to the publisher or ask for confirmation here** — the features page is the review surface.

**Always draft and send `suggested_pitch` for every component, on every run.** The server's reconcile rule makes this safe and keeps this skill stateless about pitch status: a publisher-confirmed pitch is never overwritten (the suggestion isn't even stored for it), while an unconfirmed component's suggestion gets refreshed — so a component whose body changed picks up a fresher draft automatically.

Each pitch is the predicate completing the component's fixed sentence form (the same sentence end users see in the premium feature access messages and on the public plans page):

- Skill: "The {name} skill ___."
- Tool: "The {tool_name} tool ___."
- Database record: "{title} ___." (the title carries the whole noun phrase — no "The", no type word — and is **plural**, so the predicate takes the plural verb form: "Brainstorm topics **track** the topics you have saved", never "tracks")

Drafting rules:

- Source material: a skill's frontmatter `description` plus body (kept from Step 2); a tool's registered `description` and handler (read in Step 4); a database record's two partner-tool descriptions plus its publisher-chosen `title`.
- Start with a lowercase present-tense verb that agrees with the sentence's subject — singular for skills/tools (e.g. "generates", "refines"), plural for database records (e.g. "track", "store").
- One clause, roughly 5–15 words.
- No trailing period (the rendering surfaces append it), and don't restate the component name.
- Paired tools never carry their own `suggested_pitch` — the pair's pitch goes on its `database_record` entry (the server rejects a paired tool entry that carries one).
- A component you have no real material for can omit `suggested_pitch`; the publisher authors it from scratch on the dashboard.

## Step 6: Sync the plugin and all components in one call

Call the `plugpass_sync_plugin` tool (under any connector prefix) with the full payload (PUBLISHER_TOOLS_MISSING applies if the tool isn't in your catalog — in SYNC mode the pre-check already established it):

```json
{
  "plugin": {
    "name": "{name}",
    "displayName": "{manifest displayName, omit if absent}",
    "slug": "{derived-slug}",
    "description": "{description, omit if absent}",
    "license": "{plugin.json license id, omit if absent}",
    "plugpass_id": "{plugin-plugpass-id from step 0 — SYNC mode only; omit in REGISTER mode}",
    "manifests": ["{each family from {manifests} — \"claude\", \"codex\", or both}"],
    "source": { "repository": "{repository from step 1b}", "subdirectory": "{subdirectory from step 1b}" }
  },
  "components": [
    { "type": "skill",          "name": "{name}", "plugpass_id": "{from frontmatter, omit if absent}", "suggested_pitch": "{drafted pitch, omit if none}" },
    { "type": "mcp_dependency",
      "name": "{server-key}",
      "domain": "{hostname; omit for stdio}",
      "url": "{full .mcp.json url, verbatim; omit for stdio}",
      "plugpass_id": "{from the pre-check, omit if newly-discovered or in REGISTER mode}",
      "ownership": "{owned | not_owned, ONLY when Q1 just collected the answer for this server; omit when server-side ownership is already known}"
    },
    { "type": "database_record",
      "name": "{derived snake_case identifier}",
      "title": "{publisher-chosen sentence-case plural display label}",
      "plugpass_id": "{from the pre-check if pre-confirmed, omit on first confirmation}",
      "suggested_pitch": "{drafted pitch, omit if none}"
    },
    { "type": "tool",
      "server_name": "{server-key}",
      "tool_name": "{name}",
      "visibility": "{model | app | both, from the registration}",
      "ui_backed": "{true | false, from the registration}",
      "plugpass_id": "{from _meta or the pre-check map, omit if absent}",
      "database_record_name": "{matching database_record entry's name, omit for solo}",
      "operation": "{add | remove, omit for solo}",
      "suggested_pitch": "{drafted pitch — solo tools only, never on paired entries; omit if none}"
    }
  ],
  "publisher_plugin_version": "{PUBLISHER_PLUGIN_VERSION}"
}
```

(Every `plugpass_id` field is omitted throughout in REGISTER mode.)

Capture from the successful response:

- `plugin_id` — the plugin's public short id (a bare base58 string, no prefix). In SYNC mode it should match `{plugin-plugpass-id}`; in REGISTER mode it's newly minted (or the relinked existing registration's id).
- `components` — array of `{ type, plugpass_id, ...type-specific fields }` for every component the server now knows about.
- `continuation_url` — points at the dashboard page the server chose as this sync's follow-up surface: the plugin settings (confirmation) page normally, the plans page when an already-published plugin gained new components (which need assigning to plans there), or the page that fixes an outstanding connector-setup gap.
- `continuation_message` — optional. Present when the plugin's connector state needs the publisher's attention (e.g. connector setup to complete, which the message lists as errors to fix, or the connector switching back to the Plugpass-hosted one). Relay it VERBATIM in Step 10 — never paraphrase or omit it.

## Step 7: Write plugpass-plugin-id back to every manifest

For each manifest in `{manifests}`: if its `metadata.plugpass-plugin-id` doesn't match the returned `plugin_id`, use `Edit` to set it. Add a `metadata` block if one doesn't exist yet. Preserve all other manifest fields exactly; leave a manifest already carrying the right id alone. (In SYNC mode the ids should already match — this is a verify; in REGISTER mode this is the write that links the working copy. A repo that has gained a second manifest since the last run picks the id up here.)

## Step 8: Write plugpass-component-ids back to source

For every `skill` component in the response, find the matching source file (by component `name` matched against the file paths tracked in step 2). If the source file's `metadata.plugpass-component-id` doesn't match the returned `plugpass_id`, use `Edit` to update the frontmatter:

- If `metadata.plugpass-component-id` is present with a different value, replace it with the returned `plugpass_id`.
- If `metadata.plugpass-component-id` is absent, add it. If the frontmatter has no `metadata:` block yet, add one above the existing fields.

Preserve indentation, comments, and unrelated frontmatter fields. Do not modify the skill body or anything below the closing `---`.

For every `tool` component in the response, stamp its plugpass id into the publisher's MCP server source so renames stay detectable across runs. Go back to the registration you read in Step 4 and add (or correct) a `_meta` field carrying `plugpass_component_id: "{plugpass_id}"`. Placement follows the server's MCP SDK — e.g. the tool-config object passed to `registerTool` in the TS SDK (`{ title, description, inputSchema, _meta: { plugpass_component_id } }`); adapt to the server's language (requires MCP protocol revision 2025-06-18+). Leave every other `_meta` field alone — `_meta.ui` is the publisher's. This is **identity only** — not the premium feature access check wrapper, which `plugpass-implement-code-changes` adds later once plans are configured. Idempotent: if `_meta.plugpass_component_id` already matches, leave it — this is also how renames stay linked across runs.

`mcp_dependency` components skip this step — `.mcp.json` has no metadata slot to write an id into.

## Step 9: Open the dashboard

Open {continuation_url} with the OPEN_URL_TOOL. (The server chose the destination: the plugin settings/confirmation page normally; the plans page when a published plugin gained new components; the page that fixes an outstanding connector-setup gap when there is one.)

## Step 10: Report

Tell the publisher what happened, by mode:

- **REGISTER mode:** "The {plugin-name} plugin was successfully registered with Plugpass. Continue setup in the Plugpass dashboard in your browser."
- **SYNC mode:** "The {plugin-name} plugin was successfully updated in Plugpass. Continue in the Plugpass dashboard to confirm the changes, assign any new features to plans, & publish the plugin updates."

If the response carried `continuation_message`, include it verbatim as its own paragraph. Then end the skill.
