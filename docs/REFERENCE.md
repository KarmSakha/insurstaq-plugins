# InsurStaq plugins: full reference

These packages let the coding agent you already use ask InsurStaq, on your Mac, whether its
changes made the repository less secure. Each package is a thin launcher: it starts the local
InsurStaq MCP server (`insurstaq mcp`) and explains how to use it. All behavior lives in
InsurStaq for Mac. The InsurStaq app can also connect each agent for you with one click:
Settings → Coding Agents ([Install without a marketplace](#install-without-a-marketplace)).

| Folder | For | Contents |
|---|---|---|
| [`claude/`](../claude/) | Claude Code | `.claude-plugin/plugin.json` + `.mcp.json` + `scripts/insurstaq-mcp` launcher |
| [`cursor/`](../cursor/) | Cursor | `.cursor-plugin/plugin.json` + `mcp.json` |
| [`codex/`](../codex/) | Codex | portable `plugin.json` + `mcp.json`, and a `config.toml` snippet |
| [`copilot/`](../copilot/) | VS Code with GitHub Copilot (agent mode) | `.vscode/mcp.json` |

Product names are used only to describe compatibility. No affiliation or endorsement implied.

## Quick start

1. Install **InsurStaq for Mac 1.0.6 or later** from [insurstaq.ai](https://insurstaq.ai), add your repository and
   run one audit. Guard needs a licence that includes it (Solo and above).
2. Connect your coding agent, either way:
   * In InsurStaq, open **Settings → Coding Agents** and choose **Connect…** next to the agent; or
   * Claude Code, from this repository's marketplace:

     ```text
     /plugin marketplace add KarmSakha/insurstaq-plugins
     /plugin install insurstaq@insurstaq
     ```

   Manual commands and install links for Claude Code, Codex, Cursor and VS Code are in
   [Install without a marketplace](#install-without-a-marketplace).
3. Ask your agent to verify its changes with InsurStaq before it finishes.

## Architecture

Plugins are distribution; MCP is local plumbing. Users install a plugin; they are not asked to
hand-configure MCP (the snippets here exist for tools without a plugin format).

```text
Cursor plugin  ─┐
Codex plugin   ─┤
Copilot config ─┼── local MCP (stdio) ──> InsurStaq Core (same engine, same data directory)
Claude plugin  ─┘
```

* Every package launches the same command, `insurstaq mcp` (the tool bundled in InsurStaq.app, see
  [below](#where-the-insurstaq-command-comes-from)), speaking MCP over stdio.
* The server never listens on a network socket, never makes network requests (its network
  gateway is blocked for the whole process), and never runs your repository's code.
* It is a separate process that opens the same encrypted local stores as the InsurStaq app
  (`~/Library/Application Support/ai.insurstaq.app`). The app window does not need to be open.
  It does not start file watchers or background update/license checks; the app does those.

## Two privacy boundaries

**Guard boundary** — the agent writes files, InsurStaq reads them locally, and nothing is sent
back to the agent:

```text
external agent writes files → InsurStaq reads them locally → nothing is returned to the agent
```

**Plugin boundary** — the agent asks the local server, and gets security metadata, not source
code:

```text
external agent  ↔  local InsurStaq MCP
```

```text
BLOCKED
severity: high
category: authorization
finding: SEC-143
summary: Ownership validation removed
file: src/routes/invoices.ts:42
```

The agent already has the repository, so a path and line are enough for it to find and fix the
code itself. InsurStaq does not copy repository text into the agent's context, which may be a
hosted model.

### Zero-cloud wording

* **InsurStaq never uploads your source code.** Guard, the MCP server and the security engine
  run on your Mac; nothing is sent to InsurStaq servers or to a hosted model by InsurStaq.
* **Your coding tool may send code to its own model provider**; that is between you and that
  tool. A workflow that uses a hosted coding agent is not zero-cloud, and InsurStaq does not
  claim it is.

## Tools

Every tool takes `project`: the absolute path of the repository (or of any file or folder inside
it, or a `file://` URI) or the InsurStaq project id. The repository must already be added to the
InsurStaq app; unknown paths return `PROJECT_NOT_FOUND` and no data.

| Tool | What it does | Returns |
|---|---|---|
| `insurstaq_guard_status` | Current Guard state of the project. | state (`clean`, `attention`, `blocked`, `incomplete`, `stale_baseline`, `disabled`, `error`, …), baseline commit, HEAD, changed-file count, introduced/resolved counts, completeness, last run time |
| `insurstaq_verify_changes` | Runs one Guard analysis now against the baseline (deterministic engines only: no AI, no repository code execution). | `PASSED`, `PASSED_WITH_WARNINGS`, `BLOCKED` or `UNABLE_TO_VERIFY`, counts, regressions (metadata only) |
| `insurstaq_list_regressions` | Regressions introduced since the baseline, from the latest analysis. | per regression: finding id, severity, category, title, file:line, confidence, short summary |
| `insurstaq_get_finding_summary` | One finding by id (`SEC-143`). | status, severity, category, title, file:line, CWE, short description, recommended direction, attack-path status |
| `insurstaq_request_fix` | Queues a fix request in the InsurStaq app. | acknowledgement; the user reviews it in InsurStaq. Never changes files, never returns a patch |

Each result has a short text block whose first line is the verdict or state, followed by
`severity:` / `category:` / `finding:` / `summary:` / `file:` lines, plus the same data as
compact JSON in `structuredContent`, repeated as a second text block for clients that ignore
structured content. Every tool declares an `outputSchema` listing exactly the fields it can
return (`additionalProperties: false`); error results carry only `error: { code, message }`.
Clients choose what the model sees: Claude Code, for example, passes the `structuredContent`
JSON of a successful result to the model, and the text blocks of an error result. `UNABLE_TO_VERIFY` means the changes are **not**
verified: a failed or incomplete analysis is never reported as `PASSED`.

### Never returned

The server has no tool for, and its responses never contain:

* file contents, arbitrary file reads or directory listings;
* source snippets, evidence code excerpts or taint-path expressions;
* secret values or secret previews (not even redacted fragments);
* diffs or patches;
* shell or command execution, or network access;
* changes to the Guard baseline, settings, licenses or suppressions.

Descriptions and fix directions come from InsurStaq's own category catalog, not from scanner
text that can quote code. Repository-derived strings (titles, paths, names) are stripped of
control characters and length-capped; secret-like tokens are redacted. All of this is enforced
in one place inside InsurStaq and covered by tests.

## Requirements

1. **InsurStaq for Mac 1.0.6 or later**, with the repository added (Projects → Add) and audited once.
2. **The Guard license feature** (Solo and above). Without it every tool answers
   `LICENSE_REQUIRED` with an upgrade link and no data.
3. **The `insurstaq` command-line tool**, which ships inside the app. Nothing else to install.

### Where the `insurstaq` command comes from

InsurStaq.app bundles the command-line tool, signed and notarized with the app, at:

```text
/Applications/InsurStaq.app/Contents/MacOS/insurstaq-cli
```

(It is named `insurstaq-cli` inside the bundle because the app's own executable is
`Contents/MacOS/InsurStaq`, and macOS volumes are case-insensitive.)

The Claude Code plugin starts the committed launcher script
[`claude/scripts/insurstaq-mcp`](../claude/scripts/insurstaq-mcp) (`${CLAUDE_PLUGIN_ROOT}/scripts/insurstaq-mcp`,
no arguments). It runs the first of these that exists, with `mcp` as its only argument:

1. `/Applications/InsurStaq.app/Contents/MacOS/insurstaq-cli`
2. `~/Applications/InsurStaq.app/Contents/MacOS/insurstaq-cli`
3. `/usr/local/bin/insurstaq`, then `~/.local/bin/insurstaq` (the installed command-line tool)
4. `~/.cargo/bin/insurstaq` (a developer build)

If none exists it exits with a clear message on stderr ("InsurStaq is not installed on this
Mac…") and status 127. The Cursor, Codex and VS Code packages launch the same server through a
short `/bin/sh -c` line that tries the two app locations, then `insurstaq` from your `PATH` (with
`/usr/local/bin` and `~/.local/bin` appended, because apps started from the Dock do not see your
shell's `PATH`). Either way the shell only picks the binary and `exec`s it; the server process is
the InsurStaq tool itself.

To use `insurstaq` in Terminal too, open InsurStaq → **Settings → Coding Agents** and choose
**Install Command-Line Tool…**. After you confirm, the app links
`insurstaq` → the bundled tool in `/usr/local/bin` when your account may write there, otherwise in
`~/.local/bin` (and shows the line to add that folder to your `PATH`). It never asks for an
administrator password; to install it for every account, run the command it shows:

```sh
sudo mkdir -p /usr/local/bin && sudo ln -sf /Applications/InsurStaq.app/Contents/MacOS/insurstaq-cli /usr/local/bin/insurstaq
```

The same page has **Copy Path** for the bundled tool, one-click **Connect…** for every agent (see
[Install without a marketplace](#install-without-a-marketplace)) and, under **Configure by
hand**, each agent's configuration with **Write to File…** for Cursor, Codex and VS Code. If
InsurStaq lives somewhere else, or you want to pin one binary, replace a package's launcher with
that absolute path and `["mcp"]` as the arguments.

The tool uses the app's data directory by default. Pass `--data-dir` (or set
`INSURSTAQ_DATA_DIR`) only if you run InsurStaq with a non-default data directory.

**Keychain prompt.** The database key lives in the macOS Keychain item the app created. The first
time `insurstaq mcp` opens it, macOS asks whether `insurstaq` may use that item; choose
**Always Allow**. A differently signed binary (a build from source instead of the bundled tool)
is a new identity and asks again.

## Install without a marketplace

No marketplace listing is needed. The easiest way is one click in the app: open InsurStaq →
**Settings → Coding Agents** and choose **Connect…** next to the agent. InsurStaq shows exactly
what it will do and changes nothing until you confirm; the agent or editor then asks you to
approve the server too. Nothing leaves your Mac. Connecting needs a licence that includes Guard.

The commands and links below are the same ones, for doing it by hand. They assume InsurStaq is in
`/Applications`; Settings → Coding Agents shows (and copies) them with the exact path on your Mac,
which is `/usr/local/bin/insurstaq` once the command-line tool is installed.

| Agent | One click in Settings → Coding Agents | By hand |
|---|---|---|
| Claude Code | Runs `claude mcp add` for you (the `claude` command from `~/.local/bin`, `~/.claude/local`, `/opt/homebrew/bin`, `/usr/local/bin`, `~/.npm-global/bin`, `~/.bun/bin` or `~/.volta/bin`) | Terminal command below |
| Codex | Runs `codex mcp add` for you (the `codex` command from the same folders) | Terminal command below, or [`codex/config.toml`](../codex/config.toml) |
| Cursor | Opens Cursor's MCP install link; Cursor asks you to confirm | Open the link below |
| VS Code (GitHub Copilot agent mode) | Opens VS Code's MCP install link; VS Code asks you to confirm | Open the link below, or `code --add-mcp` |

**Claude Code**

```sh
claude mcp add --scope user insurstaq -- /Applications/InsurStaq.app/Contents/MacOS/insurstaq-cli mcp
# remove: claude mcp remove --scope user insurstaq
```

Or load the plugin from a checkout of this repository: `claude --plugin-dir ./claude`.

**Codex**

```sh
codex mcp add insurstaq -- /Applications/InsurStaq.app/Contents/MacOS/insurstaq-cli mcp
# remove: codex mcp remove insurstaq
```

**Cursor** (open in a browser or with `open '<link>'` in Terminal; the `config` value is the base64
of `{"command":"/Applications/InsurStaq.app/Contents/MacOS/insurstaq-cli","args":["mcp"]}`):

```text
cursor://anysphere.cursor-deeplink/mcp/install?name=insurstaq&config=eyJjb21tYW5kIjoiL0FwcGxpY2F0aW9ucy9JbnN1clN0YXEuYXBwL0NvbnRlbnRzL01hY09TL2luc3Vyc3RhcS1jbGkiLCJhcmdzIjpbIm1jcCJdfQ%3D%3D
```

**VS Code** (the link carries `{"name":"insurstaq","command":"…/insurstaq-cli","args":["mcp"]}`):

```text
vscode:mcp/install?%7B%22name%22%3A%22insurstaq%22%2C%22command%22%3A%22%2FApplications%2FInsurStaq.app%2FContents%2FMacOS%2Finsurstaq-cli%22%2C%22args%22%3A%5B%22mcp%22%5D%7D
```

or, with VS Code's `code` command:

```sh
code --add-mcp '{"name":"insurstaq","command":"/Applications/InsurStaq.app/Contents/MacOS/insurstaq-cli","args":["mcp"]}'
```

To disconnect Cursor or VS Code, remove `insurstaq` in their MCP settings. The app builds the
same links.
Product names are used only to describe compatibility. No affiliation or endorsement implied.

## Check the server by hand

```sh
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"manual","version":"1"}}}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"insurstaq_guard_status","arguments":{"project":"'"$PWD"'"}}}' \
  | /Applications/InsurStaq.app/Contents/MacOS/insurstaq-cli mcp   # or: insurstaq mcp
```

stdout carries only protocol messages; warnings go to stderr. The
server speaks both the `initialize`-handshake MCP revisions (2024-11-05 to 2025-11-25) and the
per-request revision 2026-07-28 (`server/discover`).

## Troubleshooting

| Answer | Meaning |
|---|---|
| `LICENSE_REQUIRED` | The license lacks Guard. Upgrade at https://insurstaq.ai/pricing or check Settings → License. |
| `PROJECT_NOT_FOUND` | The path is not inside a repository added to InsurStaq. Add it in the app. |
| `UNABLE_TO_VERIFY` | Guard could not complete (no baseline yet, a scan in progress, an engine failed). Nothing is verified; open InsurStaq. |
| `NO_ANALYSIS` | Guard has not analysed the project yet; call `insurstaq_verify_changes`. |
| Server fails to start | InsurStaq.app is not in `/Applications` or `~/Applications` and `insurstaq` is not on the agent's `PATH` (use **Connect…** or the absolute path from Settings → Coding Agents), or the Keychain prompt was denied. |

## License

The plugin launchers and manifests in this repository are MIT-licensed (see [LICENSE](../LICENSE)).
This licence covers only the plugin launcher and manifest files here. The
InsurStaq application and the `insurstaq` command-line tool they start are proprietary software
of KarmSakha Limited and are not covered by this licence.
