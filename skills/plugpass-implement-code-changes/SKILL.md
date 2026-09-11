---
name: plugpass-implement-code-changes
description: Writes or updates the premium feature access checks into a plugin after the premium features are configured in the dashboard
allowed-tools: mcp__plugin_plugpass_plugpass-publisher__plugpass_get_plugin_data, mcp__plugin_plugpass_plugpass-publisher__plugpass_log_implementation, Read, Write, Edit, Glob, Grep, Bash(git:*), Bash(open:*), Bash(xdg-open:*), Bash(echo:*), Bash(curl:*), PowerShell(git:*), PowerShell(Start-Process:*), PowerShell(Write-Output:*), PowerShell(curl:*), PowerShell(Invoke-WebRequest:*), AskUserQuestion, Skill, TaskCreate, TaskUpdate, WebSearch, WebFetch
---

This skill works in the publisher's plugin repo: it reads and edits files in the current working directory and runs terminal commands there. If your environment cannot do both, tell the user "Plugpass needs to read and edit files in your plugin's repo and run terminal commands, which isn't possible here. Open your plugin's directory in an AI coding tool like Claude Code or Codex and run this skill again." and end the skill.

## What this skill does

After the publisher configures plans & benefits in the Plugpass dashboard, this skill writes the corresponding premium feature access checks into the plugin's local source — and, for a publisher-hosted plugin, into the publisher's own MCP server source — and reports back so the dashboard's implementation page can detect completion and unlock publish.

It runs **entirely in this session — no agents**: you make the file edits yourself, ask the publisher questions when needed, and research external services when the work calls for it. The procedures live in four places:

- **A missing plugin manifest** — the "Missing manifest instructions" section below.
- **Paid skill bodies** — the "Skill body prepend instructions" section below.
- **Once-per-plugin core wiring** — the "Core wiring instructions" section below (including the connector directives).
- **Publisher MCP server scaffolding** — [TOOLS.md](TOOLS.md), read only when the run has publisher-server work: the plugin's connector is the publisher's own MCP server (`connector.hosting` is `publisher`), or the run's components include `tool` entries.

The verbatim templates (the premium feature access block, the per-plugin access-handler skill body, the per-language tool-wrapper/scaffolding templates) ship under this skill's `templates/` directory.

Unlike `plugpass-sync-plugin`, this skill has **no mode-check** — by the time monetization is being implemented the plugin is already registered, so it runs in any plugin state.

**Work silently.** The only text you post is what a step calls for. Skip the short preamble that normally precedes a tool call — including the first one — and post no mode or step announcements, no commentary on what you just did or are about to do, and no recap the steps didn't ask for.

**Presenting copy.** A `>` block is finished copy; the `>` characters delimit it here and are never part of it. Reproduce the text exactly — substituting each `{VARIABLE}` with its value — and never print the `>` characters, restyle the wording, or wrap it in a quote block. The surrounding step says where the copy goes: where it says to tell the publisher something, post it as your own message with nothing of your own before or after it, by whatever messaging method will be visible to them (especially if a tool call will follow it in the same turn). Copy given inline in double quotes is delivered the same way, without the quote marks.

- PUBLISHER_PLUGIN_VERSION = `0.0.6` (stamped by the release pipeline). Include it as `publisher_plugin_version` on every Publisher MCP tool call in this skill.
- USER_INPUT_TOOL = A tool that presents the user a question with selectable options and returns their choice (e.g. `AskUserQuestion`, `ask_user_input_v0`, etc.) that can be used in the default session state (not limited to a certain mode, e.g. plan mode). Where a prompt below calls for USER_INPUT_TOOL and no such tool is available, ask the question in chat and wait for the reply.
- PLATFORM = If your system instructions indicate an OpenAI product (Codex or ChatGPT), then `openai`; otherwise (an Anthropic / Claude product) `claude`.
- SKILL_PREFIX = If PLATFORM=`openai`, then `$`; otherwise `/`. (How the publisher types a skill invocation in their client — every typed command below renders through it.)
- OS = If your system instructions indicate the platform is `darwin`, then `mac`; if `linux`, then `linux`; if `win32`, then `windows`.
- If PLATFORM=`claude`: CODE_CLIENT = If (OS=`mac` || OS=`linux`), then Bash `echo "CLAUDE_CODE_ENTRYPOINT=$CLAUDE_CODE_ENTRYPOINT"`; if OS=`windows`, then PowerShell `Write-Output "CLAUDE_CODE_ENTRYPOINT=$env:CLAUDE_CODE_ENTRYPOINT"` (expected value: `cli` || `claude-desktop` || `remote`)
- OPEN_URL_TOOL = If OS=`mac`, then Bash `open "<url>"`; if OS=`linux`, then Bash `xdg-open "<url>"`; if OS=`windows`, then PowerShell `Start-Process "<url>"`. (OPEN_URL_TOOL is undefined when CODE_CLIENT=`remote` — no local browser to open; where a step says to open a URL, present it as a markdown link instead.)
- SHELL_TOOL = If (OS=`mac` || OS=`linux`), then the Bash tool; if OS=`windows`, then the PowerShell tool. Run every `git` command in this skill via SHELL_TOOL.
- PUBLISHER_TOOLS_MISSING = If a Publisher MCP tool this skill needs is not in your tool catalog under any connector prefix (if not loaded, attempt to load it via tool search), the Publisher MCP server isn't connected: invoke the `plugpass-access-handler` skill and follow its instructions. When it returns after a successful connection, retry the call that needed the tool.

## Step 1: Read the plugpass-plugin-id

Read the plugin manifest from the current working directory (the publisher's plugin repo root): `.claude-plugin/plugin.json` when present, else `.codex-plugin/plugin.json` (a Codex-only plugin — same one-repo format). The file you read is `{manifest}` for every later manifest read in this skill; the other manifest, when it exists alongside `{manifest}` or this run creates it, is `{other-manifest}` (the core wiring keeps its `license` + `version` in step with `{manifest}`). If neither exists, tell the user "No Claude or Codex plugin manifest found. Run this skill again from inside your plugin repo." and end the skill.

Extract `metadata.plugpass-plugin-id`. If it is absent, the plugin isn't linked to Plugpass in this working copy (a fresh checkout, or a wiped/regenerated manifest) — silently invoke the `plugpass-sync-plugin` skill. Do not announce the relink or state whether the plugin is registered — you don't know yet, and the sync skill reconciles it either way: it relinks to the existing registration when the plugin is already registered under this manifest `name`, or registers it fresh otherwise, and writes the id back to the manifest. Then re-read `{manifest}` and re-extract `metadata.plugpass-plugin-id`. If it is still absent (the publisher cancelled, or registration didn't complete), tell the user "I couldn't link this plugin to Plugpass. Run `{SKILL_PREFIX}plugpass-sync-plugin`, then re-run `{SKILL_PREFIX}plugpass-implement-code-changes`." and end the skill. Capture the resulting value as `{plugin-plugpass-id}`. Also capture the manifest's current `version` and its `name` — the latter is `{PluginName}`, the plugin's lowercase manifest identifier, used verbatim as the plugin's tool-ID namespace. The display name is **not** read from the manifest: `{PluginDisplayName}` comes from the `plugpass_get_plugin_data` response (Step 2), the dashboard-authoritative value.

Determine the run mode: if the user invoked the skill with `--reconcile` (or asked for a full re-check), this is a **reconcile run**; otherwise it is a normal **changeset run**. A changeset run rewrites an artifact only when the response says something changed — its gating, or the Plugpass template it was written from (Step 3) — and otherwise verifies it is present and structurally intact without rendering it; a reconcile run additionally re-audits every `unchanged` tool's wrapper and each in-set server's scaffolding probe by probe, per TOOLS.md's Reconcile runs section — the escape hatch for drift the server's record cannot see (a reverted or restored repo).

## Step 2: Fetch the components needing implementation

Call the `plugpass_get_plugin_data` tool (under any connector prefix) with `{ "plugin_id": "{plugin-plugpass-id}", "implementation_needed_only": true, "publisher_plugin_version": "{PUBLISHER_PLUGIN_VERSION}" }`. The server returns every component needing implementation attention: each component whose premium feature access code must be **removed** (it went paid→free), plus every currently-gated, plan-assigned skill / tool — including `unchanged` ones, each carrying `template_stale` so this run knows which of them to rewrite (the template that wrote it has since changed) and which to merely verify.

**If the call errors**, the dashboard configuration isn't ready for an implement run — the server hard-gates one state, with a message naming the dashboard page that resolves it:

- **"Publisher-hosted connector setup is incomplete: …"** — the plugin runs its own MCP server(s) and the connector's prerequisites (pages domain, check-host selection, domain verification) are outstanding; the error names the dashboard page that fixes them and links to it.

In that case, relay the server's message to the publisher verbatim, then end the skill. The message specifies the error and links to where they can fix it, and instructs them to re-run the skill after resolving the issue.

**If the tool is not in your catalog**, PUBLISHER_TOOLS_MISSING applies.

Capture from the response:

- `components` — each entry carries `type`, `plugpass_id`, `gating_change_since_last_implement` (`added` / `removed` / `changed` / `unchanged`), and `template_stale` (`true` when the Plugpass template this component's code was written from changed in a later publisher-plugin version than the one that wrote it, or when no run has recorded it yet — the server computes it from what this skill reported on earlier runs; nothing is read locally). Each **skill** entry additionally carries the block-input pair — `FeatureId` (the component's stable plugpass-component-id, identical to its `plugpass_id`) and `FeatureType` (`skill`, the recital's word). The block bakes no copy or plan-structure facts — all state copy is composed server-side at check time. Each **tool** entry carries `server_name`, `tool_name`, `is_free`, and — for paired tools — `operation` + `entitlement` (the pair's shared entitlement, identity only; every gated tool bakes its own `plugpass_id` as its `feature_id`) — the inputs TOOLS.md's wrapper procedure consumes. (Tool entries appear only for a publisher-hosted plugin — gating a tool requires owning its server.)
- `connector` — `{ hosting, server_key, type, url, issuer, identity_kind }`, the plugin's connector entry (always non-null in this mode — the setup gate above guarantees it). `hosting` is the branch discriminator the connector directives key on: `native` (no owned MCP server — the Plugpass-hosted connector: `server_key` is the plugin's manifest name, `url` derives from the plugin's own domain) or `publisher` (the publisher's own MCP server IS the connector: `server_key` is the check host's `server_name` from their own `.mcp.json`, `url` is that server's canonical `https://{host}/mcp`). Its `server_key` is `{ConnectorKey}` — the key the connector is referenced under in `.mcp.json` and hence the middle segment of every `mcp__plugin_{PluginName}_{ConnectorKey}__…` tool namespace baked below. `issuer` is the connector's OAuth authorization server — the plugin's own pages-host origin (identical to `plugpass_issuer`); it is what a publisher server bakes as `ISSUER` and advertises in its PRM.
- `access_handler_template_stale` / `server_scaffolding_template_stale` — the plugin-level counterparts of `template_stale` for the access-handler skill (consumed by the core wiring) and the publisher-server scaffolding (consumed by TOOLS.md).
- `connector_change_since_last_implement` — `unchanged` | `added` (no run has recorded a connector yet) | `changed` — with `previous_connector` (`{ hosting, server_key, url, issuer, identity_kind }`; `issuer` null on a record from before it was tracked) alongside when `changed`. Any field differing reads as `changed`, including an issuer change alone (the plugin's pages host moved — the subdomain form ↔ a custom domain — so a publisher server's baked `ISSUER` / PRM is stale even though its own URL isn't). The connector-directive key: the core wiring's connector work is mechanical off this value + `connector.hosting`, no inference.
- `mcp_server_url` — the plugin's connector URL (identical to `connector.url`), surfaced top-level as one of the two values this skill bakes.
- `entitlement_api_origin`, `paywall_script_url`, `plugpass_issuer`, `plugpass_jwks_url`, `owned_servers` — the publisher-server scaffolding's baked-default inputs (TOOLS.md's runtime constants). `plugpass_issuer` is the plugin's own pages-host origin (its authorization server) and `plugpass_jwks_url` that host's key-set document — per plugin, so a plugin whose pages host changes gets new values on its next run. `paywall_script_url` is where the in-widget paywall script loads from (the plugin's own host). `owned_servers` maps each owned server's `server_name` to its registered connector-shape URL (its `RESOURCE_URL`) and its **scaffolding directive** — `scaffolding.layers`, the layers TOOLS.md writes and verifies for that server (`resource_server`, `check_proxy`, `tool_wrappers`, and `ui_paywall` on a server that renders MCP Apps widgets); execute exactly the layers named. Ignore all five when the run has no publisher-server work.
- `required_manifests` — the plugin manifests to check, as `{ family, path, manifest_label, products_label }`. `Read` each `path` at the plugin repo root; `{missing-manifests}` is the entries whose file is absent. Check exactly these entries, whatever else the repo carries.
- `plugin_slug` — the plugin's Plugpass slug (platform-assigned — never guess it from the manifest), baked as `{PluginSlug}` into the access-handler skill by the core wiring.
- `plugin_origin` — the plugin's pages ORIGIN (scheme + host, no path; https on every real deployment, http on dev's `*.localhost` hosts — the server composes the scheme, you never do) — the `{PluginOrigin}` slot the core wiring bakes the access-handler NOT_CONNECTED links on. Server-supplied because only the platform knows the active custom hostname and the environment's scheme.
- `plugin_display_name` — the plugin's dashboard-authoritative display name, baked as `{PluginDisplayName}` into the access-handler skill and each paid body's block (the core wiring and body prepend). Always from here, never the manifest — the manifest's `displayName` is optional and can be stale once the publisher edits it on the dashboard.
- `license` — `{ mode, eula }` (EULA-adoption state), consumed by the core wiring and the block's recital. When `mode` is `platform_license`, `eula` carries the rendered `LICENSE.md` + manifest value to write; otherwise the publisher's license is left untouched.
- `last_published_plugin_version` — the plugin.json version stamped at the most recent publish (null when never published); the core wiring's version-bump anchor.

On a run with an empty `components` array, `connector_change_since_last_implement` of `unchanged`, **and** an empty `{missing-manifests}`, there is nothing to implement: tell the publisher "Your plugin's code is already up to date with your plan configuration." and skip to Step 7 (still report, so the page reflects current state). A run with an empty `components` array but a connector change (`added` / `changed`), or a non-empty `{missing-manifests}`, **proceeds** — the connector directives still have real work (re-pointing or retiring a stale `.mcp.json` entry after a domain or hosting change), a missing manifest is this run's to resolve, and Step 7's report records both. (On a run with components, a pure connector flip never even looks empty — the gated set always returns, `unchanged` entries included — and proceeds through the full sequence: verified bodies, connector directives, scaffolding.)

## Step 3: Partition the work and locate local files

Partition the returned components:

- **Changeset** — every non-`unchanged` entry. These are the semantic edits: `added` → write the premium feature access code (a body block for a skill, a wrapper for a tool), `removed` → strip it.
- **Template refresh set** — every `unchanged` entry with `template_stale` `true`. Its gating didn't change, but the template its code was written from did, so its code is rewritten from the current template (a body block for a skill, a wrapper for a tool) exactly as an `added` entry's would be.
- **Verify set** — every other returned entry (`unchanged`, not stale). Each is verified present and structurally intact and otherwise left alone — never rendered or rewritten. One exception, for skills only: a block whose embedded bundle version is behind `{PluginVersion}` (Core wiring item 4) is re-reproduced so the version it reports stays current.
- **The server set** — the publisher's own MCP servers this run touches: every server with tool work (the distinct `server_name` values over the returned `tool` entries), plus the **check host** (the server named by `connector.server_key`) when `connector.hosting` is `publisher` and any returned component is not `removed` (a plugin with nothing left gated needs no check scaffolding). Empty for a native plugin with no tool entries — every subsequent server-related step then no-ops.

For each returned `skill` component, locate its source file by matching `metadata.plugpass-component-id` (in `skills/*/SKILL.md` frontmatter) against the component's `plugpass_id`. Use `Glob` + `Read` to build the path map. If an entry has **no matching local file** (a component dropped from the repo but still in the draft config), do not try to edit a file that isn't there — surface it to the publisher (it usually means `{SKILL_PREFIX}plugpass-sync-plugin` needs re-running to drop the component) and exclude it from the run's work.

**Locate each server's source.** For every server in the server set, resolve its absolute source-directory path. Per-server paths live in the plugin repo's gitignored `.plugpass/` directory (per-machine, never shared — each collaborator keeps their own):

```
.plugpass/mcp-server-paths.json
```

(relative to the plugin repo root) — a JSON object mapping `server-name → absolute path to the server's source directory`. `Read` it (a missing file is an empty map `{}`). For each server in the set: a recorded path that still resolves is kept (a prior sync/implement run usually recorded it); when the server source lives in the current working directory tree, that resolves it too. Otherwise elicit the **absolute** path with USER_INPUT_TOOL, one server at a time, then `Write` the merged map back (creating `.plugpass/` if absent; absolute paths only — this file is per-publisher and never shared). After writing, ensure the plugin repo's `.gitignore` has a `.plugpass/` line — append one if it's missing (skip when the plugin directory isn't in a git repo). Prompt: "I'm having trouble locating the source directory for the {server-name} MCP server. Please provide its absolute path." If the publisher can't provide a path, exclude that server's work from the run (its tasks stay open and it's excluded from the implementation record — Step 7); when the unlocatable server is the **check host**, the connector's server-side half can't be implemented this run, so say so plainly in the closing summary and leave the scaffolding task open — the dashboard's publish gate holds until a re-run completes it.

Also identify each located server's **repo**: run `git -C {server source dir} rev-parse --show-toplevel` via SHELL_TOOL; a server outside any git repo is tracked by its source directory instead. The repo set (the plugin repo + each distinct server repo) feeds Step 4's uncommitted-changes check and Step 6's `edited_repos` recording. A server repo that is the same repo as the plugin (a monorepo) is one entry playing both roles.

## Step 4: Check for uncommitted changes in the repos the run writes

Before any edits, check whether any repo this run writes — the plugin directory (the current working directory) plus every located server repo from Step 3 — has uncommitted work, so the publisher can commit first and keep the changes easy to review or revert. Not every publisher keeps their code in a repo, so this step stays silent unless there is actually something to commit.

For each distinct repo: run `git -C {dir} rev-parse --show-toplevel` via SHELL_TOOL. A directory that isn't inside a git repo exits non-zero — skip it. Otherwise run `git -C {toplevel} status --porcelain`; non-empty output means uncommitted changes.

If no checked repo has uncommitted changes, continue to Step 5 without mentioning it.

Otherwise, identify each dirty repo by its toplevel's directory basename and tell the publisher (naming one repo, or listing several):

> It looks like there are some uncommitted changes in the {reponame} git repo. Before I implement the plugin monetization changes, I recommend committing those so the changes I make will be easy to review or revert. Let me know once you've committed, or tell me if you'd like to continue without committing.

Wait for the publisher's reply, then continue to Step 5 — whether they committed or chose to proceed without committing.

## Step 5: Build the task list

Create one tracked task per item of work so the publisher sees per-component progress as the run advances. One task per **changeset** item and per **template refresh** item (skip any the Step 3 checks dropped for a missing local file or an unlocatable server — those are surfaced to the publisher, not tracked here); the verify set gets no tasks of its own (a version-only block refresh is incidental). Additionally:

- One task per `{missing-manifests}` entry — `subject` "Add the {manifest_label} to the plugin repo", `activeForm` "Adding the {manifest_label} to the plugin repo".
- One task for the once-per-plugin **core wiring**, when the run includes it — there is at least one gated skill or tool among the returned components, **or** the EULA is adopted (`license.mode` is `platform_license`), **or** `connector_change_since_last_implement` is not `unchanged` (the connector directives always have work then).
- One task per located **server** in the server set, for its scaffolding work (the layers its `scaffolding.layers` directive names — the resource-server layer, the check proxy on the check host, and the in-widget paywall on a server that renders widgets).

Phrase each task in the publisher's language — "premium feature access check", never internal terms like "gating". Pick the verb from `gating_change_since_last_implement`, against the dashboard-style component label (`{label}` — e.g. "deep-brainstorm skill", "add_brainstorm_topic tool"):

- `added` → `subject` "Add premium feature access check to {label}", `activeForm` "Adding premium feature access check to {label}";
- `removed` → `subject` "Remove premium feature access check from {label}", `activeForm` "Removing premium feature access check from {label}";
- a template refresh entry → `subject` "Update premium feature access check in {label}", `activeForm` "Updating premium feature access check in {label}";
- the core wiring task → `subject` "Implement monetization infrastructure", `activeForm` "Implementing monetization infrastructure";
- each server task → `subject` "Set up premium access check on the {server_name} MCP server", `activeForm` "Setting up premium access check on the {server_name} MCP server".

All tasks start `pending`. Execution is sequential in this one session, so statuses genuinely advance item by item: flip each task to `in_progress` as you pick it up and `completed` as it finishes. A task that fails — or ends in a needs-publisher-resolution state — goes back to `pending`, stays visible in the list and on the next run, and its component is excluded from the Step 7 record.

## Step 6: Implement

Work through the items sequentially, transitioning task statuses as you go:

1. **Missing manifests** — when `{missing-manifests}` is non-empty, follow the "Missing manifest instructions" below.
2. **Resolve the version bump** (Core wiring instructions, item 4) — before any body writes, so every block written or refreshed this run embeds the post-bump version.
3. **Skill bodies** — follow the "Skill body prepend instructions" below: write the block for the changeset's `added` skills and the template refresh set's skills, strip it for `removed` skills, and verify every verify-set skill (rewriting only a block whose embedded version is behind).
4. **Core wiring** — when the run includes it (Step 5's condition), follow the "Core wiring instructions" below (items 1–3, including the connector directives; item 4 was resolved up front).
5. **Publisher-server work** — when the server set is non-empty, follow [TOOLS.md](TOOLS.md) per server, executing the layers its `scaffolding.layers` directive names: the resource-server layer (`resource_server`), the check proxy (`check_proxy`, check host), and the in-widget paywall (`ui_paywall`, a server that renders widgets) — each regenerated when `server_scaffolding_template_stale` or missing, verified otherwise — and the tool wrappers (`tool_wrappers`) (the changeset's add / strip, the template refresh set's rewrite, the verify set's presence check), finishing each server with TOOLS.md's per-server verification.

Track every repository the run edits — the plugin repo, plus each server repo TOOLS.md's work touched — identified by its git toplevel's directory basename, for the report. Record per repo:

- The **set** of deploy-action roles it plays — `plugin` (the plugin bundle repo), `mcp_server` (an MCP-server source repo), or both (a monorepo holding the plugin and its server). For an `mcp_server`-role repo, also record `mcp_server_names` — the server(s) edited in it.
- Its **default branch**: `git -C {repo toplevel} symbolic-ref --short refs/remotes/origin/HEAD`, stripping the leading `origin/` (e.g. `origin/main` → `main`); omit when the command fails (a local-only repo with no `origin` remote).
- Its **current branch**: `git -C {repo toplevel} rev-parse --abbrev-ref HEAD`; omit when it returns `HEAD` (detached) or the command fails. The checklist compares this to the default branch to decide between a **push** (they match — the work already landed on the default branch) and a **merge** of `{current_branch}` into `{default_branch}` (they differ).

Omit any fact that can't be resolved; the checklist falls back to more generic wording.

## Step 7: Report back

Call the `plugpass_log_implementation` tool (under any connector prefix) with:

```json
{
  "plugin_id": "{plugin-plugpass-id}",
  "implemented_component_ids": ["{plugpass_id of each successfully-implemented skill / tool}"],
  "implemented_connector": { "hosting": "{connector.hosting}", "server_key": "{connector.server_key}", "url": "{connector.url}", "issuer": "{connector.issuer}", "identity_kind": "{connector.identity_kind}" },
  "connector_reference_present": "{true when the run confirmed the connector's .mcp.json entry is present by URL; omit otherwise}",
  "edited_repos": [{ "name": "{repo basename}", "roles": ["{plugin | mcp_server | other}", "…"], "default_branch": "{default branch — omit when unresolvable}", "current_branch": "{branch HEAD was on — omit when detached/unresolvable}", "mcp_server_names": ["{each MCP server the run edited in this repo — include for an mcp_server-role repo}"] }],
  "manifest_families_present": ["{the `family` of each required_manifests entry whose file is present at the end of this run}"],
  "plugin_version": "{the plugin.json version after Step 6 — post-bump when one applied}",
  "publisher_plugin_version": "{PUBLISHER_PLUGIN_VERSION}"
}
```

Report **only successes** — a half-done or partially-blocked run still records the parts that landed and a re-run picks up the rest. A component that couldn't be brought to its end state is never reported (its task stays open). Include every returned component the run brought to **or confirmed in** its end state — the rewritten changeset and template-refresh entries and the verify set's confirmed entries alike: the server records this run's `publisher_plugin_version` against each as the template version its code now stands at, which is what clears `template_stale`. `implemented_connector` is **required on every run** and is exactly the Step 2 response's `connector` (hosting, server key, URL, issuer) — the connector this run actually wired for; the platform records it verbatim and compares the current target against it, which is how a connector change after (or mid-) run reads back as outstanding. **Never send the report when the core wiring didn't reach its end state** — the connector's `.mcp.json` work, the access-handler skill, and the license files alike (the server records the handler's template version off this report, and the connector off `implemented_connector`). If any of that failed (e.g. the check host's scaffolding couldn't be completed, or the publisher-hosted `.mcp.json` entry is missing), stop before Step 7: make no `plugpass_log_implementation` call at all if nothing else succeeded, or, when other components did land, report them on a follow-up run after the core wiring completes — recording a connector or handler the code doesn't actually implement would falsely satisfy the publish gate or hide a stale handler. `connector_reference_present` is add-only server-side; send `true` only after the by-URL presence check actually passed (the native entry the core wiring ensured, or the publisher's own existing entry it verified). `manifest_families_present` is **required on every run**: the `family` of every `required_manifests` entry whose file exists at the end of the run — those Step 2 found plus any this run created, never one still missing. The server replaces the recorded set with it, so list every present family each run. Include `edited_repos` only when the run edited at least one repo; each entry carries the repo's basename `name`, its **non-empty `roles` set**, `mcp_server_names` for an `mcp_server`-role repo, and its `default_branch` / `current_branch` when resolved (omit any field that couldn't be resolved). The platform **union-merges** each run's `edited_repos` into the recorded set, so reporting only this run's repos is correct even across multiple runs: a later partial run adds to the deploy checklist rather than replacing it. Always include `plugin_version`. The server stamps each reported component's current draft gating fingerprint, which is what the dashboard's implementation page reads back to detect completion.

## Step 8: Hand off to the dashboard

Run this **only when the implementation is fully complete** — every targeted component successfully implemented, nothing failed or left pending.

If anything is incomplete (a component with no local file, any component you couldn't bring to its end state, or a `{missing-manifests}` entry still missing), do **not** open the URL or post the message below. Surface what's blocking and work through it with the publisher until every component is implemented; this handoff fires only once a run reaches full completion (possibly a later re-run, after they've resolved the blocker).

Once everything is implemented, open https://plugpass.ai/dashboard/plugin/{plugin_slug}/publish with the OPEN_URL_TOOL, then give the publisher this closing message verbatim (don't add a deploy checklist — the publish page owns it):

> The monetization changes have been implemented.
>
> I'd recommend reviewing them now, but you should **wait to merge/deploy them** until after you've published the monetization changes in Plugpass.
>
> Next, return to your browser to publish your changes in Plugpass.

Then end the skill.

## Missing manifest instructions

Work through `{missing-manifests}` one entry at a time.

Tell the publisher, interpolating that entry's fields:

> This plugin supports the {products_label}, and they need a {manifest_label} (`{path}`) that this repo doesn't have.
>
> Two ways forward: create the manifest, or deselect the {products_label} on the plugin's Distribution page (https://plugpass.ai/dashboard/plugin/{plugin_slug}/distribution).

Then ask with USER_INPUT_TOOL "Want help creating it?", offering "Create it now" and "I'll take care of it later".

If they choose "I'll take care of it later": leave the task pending, leave that entry's `family` out of Step 7's report, and continue the run.

If they choose "Create it now":

1. Find the current specification for that manifest with WebSearch and read it with WebFetch (the platform's own documentation or schema). Write the file from that specification, not from memory.
2. Copy every field the specification shares with `{manifest}` from `{manifest}`, `name` verbatim.
3. Set `metadata.plugpass-plugin-id` to `{plugin-plugpass-id}`.
4. For each remaining required field, ask the publisher with USER_INPUT_TOOL, or in chat when the answer is free text. A Codex manifest's `interface` block is what users see at install: ask the publisher for its values rather than inferring them. Use only values the publisher gives and files that exist in the repo; omit an optional field you have no value for.
5. If a required field still has no value, tell the publisher which one, leave the task pending, leave the entry's `family` out of Step 7's report, and continue the run.
6. Write `{path}` with `Write`, then mark the task completed and count the entry's `family` as present in Step 7's report.

## Skill body prepend instructions

Each component arrives with its `plugpass_id`, `FeatureId` (the same id, surfaced under the block-input slot name), `FeatureType`, `gating_change_since_last_implement`, located file path, and `{plugin-plugpass-id}`. Skills live in `skills/*/SKILL.md`. You are **idempotent**: read each file before editing and only `Edit` what differs. **Never modify the publisher's own instructions** — the only changes are prepending/updating/removing the access block and the frontmatter grants below.

### Write the premium feature access block (`added`, the template refresh set, and a verified block behind on its version)

Two changes only:

**1. Frontmatter grants.** Additively merge (never drop the publisher's own) the grants the skill needs. `{PluginName}` is the plugin's `name` from `{manifest}` — its lowercase manifest identifier, used verbatim as the plugin's tool-ID namespace (Claude Code and Codex namespace plugin MCP tools identically); `{ConnectorKey}` is `connector.server_key` from the Step 2 response — the key the plugin's connector is referenced under in `.mcp.json` (ensured present by the core wiring: the native entry it writes, or the publisher's own server entry it verifies).

- **Skills** (`allowed-tools`):
  - `mcp__plugin_{PluginName}_{ConnectorKey}__{CheckToolName}`
  - `Skill`

  A skill always runs in the **main agent**, whose universe already carries the flow tools the access-handler uses (and `AskUserQuestion` is a skill default), so a paid skill only needs the check plus `Skill` to hand off — an invoked skill is bounded by its caller's universe but reaches those tools through it. Additively merge as always; leave any generic grants the publisher already uses alone.

**2. The premium feature access block, prepended as the very first content of the body.** Read [templates/premium-feature-access-block.md](templates/premium-feature-access-block.md) (a bundled resource of this skill) and reproduce its content verbatim — changing **only** the `{...}` slots below. The block carries **no markers**: it runs from the top of the body down to and including its closing `CORE_INSTRUCTIONS:` + `---` line, which is the boundary between your region (everything above) and the publisher's body (everything below). The block goes at the very top; you never renumber, reorder, or edit the publisher's existing instructions. There is no `${CLAUDE_PLUGIN_ROOT}` reference and no in-body fallback: the access-handler skill owns every non-authorized flow.

Slot values (all from this component's `plugpass_get_plugin_data` entry unless noted):

- `{PluginName}` → as captured in Step 1; `{PluginPublicId}` → the plugin's public short id (the `{plugin-plugpass-id}` captured in Step 1 — a bare base58 id, baked as the check call's `plugin_id`); `{CheckToolName}` → the plugin data's `check_tool_name`; `{AccessHandlerSkillName}` → the plugin data's `access_handler_skill_name`; `{ConnectorKey}` → `connector.server_key` from Step 2.
- `{FeatureType}` → the component's `FeatureType` (`skill`) — a **recital-only** slot (its "this {FeatureType}" wording), not sent on the check call and not passed on the invoke. `{FeatureId}` → its `FeatureId` (the component's public short id, prefixed `skill_` — the prefix carries the feature type everywhere it's needed at runtime; the block dual-bakes it into both the check call's `feature_id` and the invoke's single `FEATURE_ID` argument).
- `{PluginVersion}` → the plugin's current `plugin.json` version (after Step 6's bump resolution). A verified block whose embedded value is behind this is re-reproduced (the verify procedure below) so the version it reports stays current; template changes reach a block through its `template_stale` flag, never through a blanket re-render.
- `{PluginDisplayName}` → the plugin's display name, from `plugin_display_name` on the `plugpass_get_plugin_data` response (dashboard-authoritative — never the manifest). Used only in the legal recital.

The block bakes **zero merchandising or plan-structure facts** — no pitch, no name copy, no free-plan hint. All state copy arrives server-composed in the check result at runtime, so copy edits and free↔paid flips never require a re-implement run.

**The final paragraph of the block — the legal recital (§ 1201 / § 106 / AUP) — is included only when the publisher has adopted the Plugpass EULA** (`license.mode` is `platform_license`). For a plugin under any other license, omit that paragraph (everything from "The premium feature access check above is a technological protection measure…" down to just above `CORE_INSTRUCTIONS:`): the license affirmatively permits copying and modification, so its assertions would be false. The functional check, the skill hand-off, and the `CORE_INSTRUCTIONS:` + `---` boundary stay in both cases.

### Verify the premium feature access block (the verify set)

Read the top of the body and check, without rendering the template: the body begins with the block and its first `CORE_INSTRUCTIONS:` + `---` boundary follows it; the check call names `{CheckToolName}` with this component's `{FeatureId}` and `{PluginPublicId}`; the frontmatter carries the two grants above; and the recital paragraph is present exactly when `license.mode` is `platform_license`. A block that is missing or fails any of those checks is treated as `added` — write it. A block that passes but embeds a `plugin_version` other than `{PluginVersion}` is re-reproduced (the write procedure above) so the version it reports stays current. Otherwise leave the file untouched.

### Strip the premium feature access block (`removed`)

Remove the prepended block — everything from the top of the body down to and including the first `CORE_INSTRUCTIONS:` + `---` boundary line — restoring the publisher's own body (everything below that boundary) as the start of the file. Also remove the frontmatter entries you would have added (leave any the publisher's own body legitimately uses — when unsure, leave the grant; an unused grant is harmless, a missing one breaks the component). Leave the rest of the body untouched. If no block is present (the body has no leading premium feature access block ending in a `CORE_INSTRUCTIONS:` + `---` boundary), it's already stripped — no-op.

### Sweep stale blocks the server didn't order (every run)

The server's changeset can only order work against implementations it has recorded — a block written but never reported (a hand-baked body, a bake predating fingerprint recording, a lost record) is invisible to it, so a component that later went paid→free surfaces **no** `removed` entry and its stale block would survive a by-the-book run. The local sweep is the only possible detector, so run it on every run, after the changeset, template-refresh, and verify work: Glob every `skills/*/SKILL.md` (skipping the access-handler skill's directory — `skills/{AccessHandlerSkillName}/`, or a legacy `skills/premium_check_cases/` — Plugpass infrastructure, never a component), and for each body that begins with a premium feature access block (content above a `CORE_INSTRUCTIONS:` + `---` boundary) whose component this run's response did NOT order gated — it isn't in the returned set at all, or is `removed` — apply the strip procedure above. Include each swept component's plugpass_id (its frontmatter `metadata.plugpass-component-id`) in Step 7's `implemented_component_ids` so the server stamps its current (free) fingerprint and the baseline finally reflects reality (a later re-gate then reads `added`). If a swept file's frontmatter id isn't one the server knows (Step 7 would reject it), leave it out of the report and note the sweep in the closing summary instead.

## Core wiring instructions

The once-per-plugin Plugpass wiring — up to four items, each gated independently, so the run does only the relevant subset. Items 1–2 apply when the plugin has at least one gated skill or tool among the returned components (item 1 additionally whenever `connector_change_since_last_implement` is not `unchanged` — a connector change always has `.mcp.json` work, even on an otherwise-empty run); item 3 when the EULA is adopted; item 4 whenever the run changes any file in the plugin repo.

### 1. The `.mcp.json` connector reference — the connector directives

The `.mcp.json` work is **mechanical, keyed entirely off `connector_change_since_last_implement` + the hosting values — no inference.** Read `.mcp.json` at the plugin repo root first (if it is missing, create it with an empty `{ "mcpServers": {} }` object), then apply the matching directive:

- **`unchanged` / `added`, hosting `native`** — ensure the Plugpass-hosted entry (the **native ensure**, below).
- **`unchanged` / `added`, hosting `publisher`** — **no new entry.** The connector is the publisher's own server, whose entry is how sync registered it — verify an entry whose `url` equals `connector.url` is present (match by URL, not key). If none is, the plugin repo and the registration disagree (the publisher removed or changed their server's entry locally without re-syncing) — tell them "Plugpass has {connector.url} registered as this plugin's premium access check server, but your `.mcp.json` no longer has an entry for it. Run `{SKILL_PREFIX}plugpass-sync-plugin` to re-sync, then re-run `{SKILL_PREFIX}plugpass-implement-code-changes`." and end the skill.
- **`changed`, native→publisher** (`previous_connector.hosting` `native`, `connector.hosting` `publisher`) — remove the entry whose `url` equals `previous_connector.url` (the plugin's own retired native connector — the generalized retire-stale exception), then proceed as `added` + `publisher` above.
- **`changed`, publisher→native** — run the native ensure; remove **nothing** (the publisher already deleted their server's entry — that's how sync knew — or it legitimately remains for a server that still serves tools).
- **`changed`, publisher→publisher** (a check-host reselect) — verify the NEW check host's entry is present by URL, exactly as `added` + `publisher`; remove nothing here (both entries are the publisher's own servers). The server-side half — scaffolding the new check host, removing the old one's proxy tool — is TOOLS.md's work, per the connector directive it receives.
- **`changed`, publisher→publisher with the same `server_key` and `url`** (issuer drift — the plugin's pages host moved) — verify the entry by URL as `unchanged` + `publisher`; `.mcp.json` needs nothing else. The server-side half — re-baking the check host's `ISSUER` / `JWKS_URL` (and with them its PRM) — is TOOLS.md's work.
- **`changed`, native→native** (URL drift from a domain change) — re-point the existing native entry: find the entry whose `url` equals `previous_connector.url` and set its `url` to `connector.url` (the key — the manifest name — is unchanged); if no entry matches the previous URL, fall back to the native ensure.

**The native ensure:**

1. **Match by URL, not by key.** If any entry under `mcpServers` has `url` equal to `connector.url`, the reference is already present — do nothing further (idempotent).
2. Otherwise, add one entry under the key `{ConnectorKey}` (the `connector.server_key` — the plugin's own manifest name, so the connect moment reads as the plugin's own):

   ```json
   "{ConnectorKey}": { "type": "http", "url": "<connector.url>" }
   ```

   Preserve every existing `mcpServers` entry and all other `.mcp.json` content exactly; only add the one key. Use the `url` verbatim — never hardcode a platform URL yourself; it is derived per-plugin (the plugin's own domain + `/mcp`), differs between dev and production, and is always supplied by `plugpass_get_plugin_data`.

3. **Retire stale platform entries.** If `mcpServers` still carries an entry for the RETIRED shared Plugpass connector — the key `premium-check`, or any entry whose URL host is `auth.plugpass.ai` or `localhost:8788` — remove that one entry (it points at a decommissioned host; leaving it gives every user of the plugin a dead, un-connectable server).

This reference is required whenever the plugin has a gated skill: their premium blocks call `{CheckToolName}` on this connector. (A publisher-hosted plugin's reference is the publisher's own server entry — verified, never written, per the directives above.)

**Add-only.** Never remove or rewrite any existing reference beyond the two retire exceptions above (the legacy shared-connector entries, and the plugin's own previous native connector on a native→publisher flip) — both are the plugin's own dead platform entries, never the publisher's servers. A listed-but-unused connector entry is inert — mere presence triggers no auth; its tools run only when actually called — while removing an entry a user has already connected severs that connection. **Never register the native connector as an `mcp_dependency`** and never send it to `plugpass_sync_plugin`: it is platform-served and its audience derives from plugin state; this is a pure local-source write with no server-side row (the exact complement of the platform-MCP filter in `plugpass-sync-plugin`).

### 2. The access-handler skill

The per-plugin skill that owns **all** user-facing premium feature access behavior — the not-connected / sign-up / limit-reached / out-of-date flows — inline in one `SKILL.md`. Every paid skill's block hands off to it (by invoking the `{AccessHandlerSkillName}` skill), and a paid MCP tool's non-authorized `result_text` auto-triggers it by description. It is **fully Plugpass-owned**. Regenerate it wholesale from the bundled template when `access_handler_template_stale` is `true` (the template changed since the run that last wrote it), when the file is missing, or when it is **divergent** — read `skills/{AccessHandlerSkillName}/SKILL.md` and check, without rendering the template, that its frontmatter `name` is `{AccessHandlerSkillName}`, its `description` carries `PLUGPASS_PLUGIN={PluginSlug}`, and its body carries every current baked value: the `mcp__plugin_{PluginName}_{ConnectorKey}__` namespace, `{PluginDisplayName}`, the `{PluginOrigin}` links, `{MarketplaceName}`, `{MarketplaceCliName}`, and `{MarketplaceRepo}`. Any failed check is divergence. Otherwise leave the file untouched. Those checks are what carry a display-name, domain, connector, or marketplace change into the handler; the staleness flag carries template changes.

When regenerating, write `skills/{AccessHandlerSkillName}/SKILL.md` (create the directory if absent; delete a legacy `skills/premium_check_cases/` directory when present — the access-handler replaces it) by reproducing the bundled template [templates/access-handler.md](templates/access-handler.md) (a bundled resource of this skill) **verbatim** — it is the complete skill, **frontmatter and body**, so you author no frontmatter yourself — baking **only** the plugin-constant slots below and leaving every other token literal:

- **Slots to bake (write time):** `{PluginName}` (manifest identifier — the first segment of the `mcp__plugin_{PluginName}_{ConnectorKey}__…` tool namespaces), `{ConnectorKey}` (`connector.server_key` — the namespaces' second segment, in the frontmatter `allowed-tools` and the body's tool references), `{PluginDisplayName}` (`plugin_display_name`), `{PluginSlug}` (`plugin_slug` — the `PLUGPASS_PLUGIN={PluginSlug}` auto-trigger condition in the frontmatter `description`), `{PluginOrigin}` (`plugin_origin` — the NOT_CONNECTED flows' baked `{PluginOrigin}/signup…` / `/login…` links, which carry `connect=web` + `platform=anthropic`; baked because NOT_CONNECTED is the one case with no server response in hand to read a runtime base from, and a re-implement re-bakes it after a domain change; the value already carries its scheme — never prepend one), and `{MarketplaceName}` / `{MarketplaceCliName}` / `{MarketplaceRepo}` (`marketplace_name` / `marketplace_cli_name` / `marketplace_repo` — the update nudge's marketplace facts: the marketplace the plugin distributes through, its `extraKnownMarketplaces` settings key, and the `owner/repo` that entry sources from; a marketplace change on the dashboard re-bakes on the next implement run).
- **Leave literal (resolved at the end user's runtime by the skill itself):** every ALL-CAPS token (`USE_AUTHORIZED`, `UPDATE`, `PLUGPASS_PLUGIN`, `FEATURE_ID`, `CTAS`, `MESSAGE`, `PLUGIN_ORIGIN`, and the skill's own detection variables — `NOT_CONNECTED`, `PLATFORM`, `CLAUDE_PRODUCT`, `OPENAI_CLIENT`, `OS`, `CODE_CLIENT`, `OPEN_URL_TOOL`, `CAN_OPEN_URL`, `USER_INPUT_TOOL`, `PAYWALL_UI`, `AUTH_URL`, `CHECK_RESULT`, `CTA`) and every other non-plugin-constant `{…}` token. These are filled by the invoke's `FEATURE_ID` argument, the check result already in context, and the skill's own runtime detection — never at write time. `PLUGIN_ORIGIN` in particular is **never baked** even though the baked `{PluginOrigin}` slot carries the same origin: it is read from the update-nudge response at runtime, which is what makes a custom-domain change take effect on the next check with no re-implement and no republish (every other page URL arrives fully built inside the server-composed `MESSAGE`/`CTAS`).

Write nothing else into the publisher's plugin for the flows — there is no `.check_premium_access/` directory, no in-body fallback, no per-surface instruction file. The access-handler skill is the single home for all of it.

### 3. The `LICENSE.md` and manifest license

_Only when `license.mode` is `platform_license` (then `license.eula` is non-null)._

1. Write `license.eula.license_md` **verbatim** to `LICENSE.md` at the plugin repo root, overwriting any existing `LICENSE.md`. The content is the fully rendered EULA (cover page + Bonterms base), composed server-side with all tokens already filled — never edit, re-wrap, summarize, or regenerate it yourself. (Idempotent — if the file already matches, it's a no-op.)
2. Set the manifest `license` field in `{manifest}` to `license.eula.manifest_license`. Use `Edit`, preserving every other field exactly; add the `license` key if absent. When `{other-manifest}` exists too, set its `license` field the same way — the same Edit-preserving-every-other-field rule; the two manifests describe one plugin and must agree.

Never touch `LICENSE.md` or the manifest `license` field when `license.mode` is anything other than `platform_license` — the publisher keeps their own license.

### 4. The manifest version bump

Resolved **first**, before any body writes (Step 6), so every refreshed block embeds the post-bump version. The bump is what makes rolling plugin updates actually deliver the new bodies to users — without it, pinned-version installs never see the update.

1. Determine whether this run will change any plugin-repo file **aside from the bump itself** — by the read-only checks above, never by rendering: a changeset skill edit, a template refresh set skill, a verify-set block that is missing, broken, or behind on its embedded version, an access-handler regenerate (item 2's conditions), a manifest this run creates (Step 6's item 1), or a core wiring write (items 1 or 3) that isn't already current. If nothing else would change, the run is a no-op for the plugin repo — no bump, and every verified body is left untouched.
2. The **current version** is `{manifest}`'s `version` — or, when `{other-manifest}` exists too and the two disagree, the **higher** of the two (semver order): the two manifests describe one plugin and must report one version, and unifying upward never rolls back a version that is already out there on either platform.
3. When the run will change the plugin repo: bump the current version (patch — e.g. `1.2.3` → `1.2.4`) **only if** it still equals `last_published_plugin_version` from Step 2. A never-published plugin (null anchor), or one whose version the publisher already bumped themselves, is left alone — re-runs never inflate the version (idempotent: the first bump makes current ≠ anchor, so subsequent runs skip it until the next publish re-anchors).
4. Either way, the version the bodies embed as `{PluginVersion}` is the current version **after** this resolution, and it is written to `{manifest}` — and to `{other-manifest}` when it exists — wherever it differs (Edit, preserving every other field). Codex reads `.codex-plugin/plugin.json` first, so the two must agree.
