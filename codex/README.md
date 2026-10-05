# InsurStaq for Codex

A Codex plugin that connects Codex to InsurStaq Guard on your Mac. It only launches the local
InsurStaq MCP server (`insurstaq mcp`, bundled in InsurStaq.app); everything else happens in
InsurStaq.

Codex is named only to describe compatibility. No affiliation or endorsement implied.

| File | Purpose |
|---|---|
| [`plugin.json`](plugin.json) | Portable plugin manifest (Agent Plugins 1.0.0 schema) |
| [`mcp.json`](mcp.json) | The stdio server the plugin registers |
| [`config.toml`](config.toml) | The same server as a `[mcp_servers.insurstaq]` entry, for configuring Codex without the plugin |

## Install

Requirements: InsurStaq for Mac (in `/Applications`) with the repository added, and a license
that includes Guard. The app bundles the `insurstaq` command the plugin runs; nothing else to
install ([details](../docs/REFERENCE.md#where-the-insurstaq-command-comes-from)).

* **Plugin:** install InsurStaq from a plugin marketplace, or add this folder to a local
  marketplace (`~/.agents/plugins/marketplace.json` or `$REPO/.agents/plugins/marketplace.json`)
  with a `local` source pointing at it, then restart Codex.
* **Without the plugin:** append [`config.toml`](config.toml) to `~/.codex/config.toml`, or run
  `codex mcp add insurstaq -- /Applications/InsurStaq.app/Contents/MacOS/insurstaq-cli mcp`.

`tool_timeout_sec = 300` in the snippet gives `insurstaq_verify_changes` time to analyse large
repositories.

## Use

Ask Codex to verify its work, for example:

> Before you finish, call insurstaq_verify_changes on this repository and fix anything BLOCKED.

Codex receives a verdict with finding ids, severities and `file:line` locations, then fixes the
code itself or queues the finding for you in InsurStaq with `insurstaq_request_fix`. See the
[tool list](../docs/REFERENCE.md#tools).

## Privacy

InsurStaq never uploads your source code. The server returns security metadata (verdict,
severity, category, finding id, short summary, file location), never source code, secrets or
diffs. Codex itself may send code to its model provider; that is between you and Codex.
