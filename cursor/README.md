# InsurStaq for Cursor

A Cursor plugin that connects Cursor's agent to InsurStaq Guard on your Mac. It only launches the
local InsurStaq MCP server (`insurstaq mcp`, bundled in InsurStaq.app; see [`mcp.json`](mcp.json));
everything else happens in InsurStaq.

Cursor is named only to describe compatibility. No affiliation or endorsement implied.

## Install

Requirements: InsurStaq for Mac (in `/Applications`) with the repository added, and a license
that includes Guard. The app bundles the `insurstaq` command the plugin runs; nothing else to
install ([details](../docs/REFERENCE.md#where-the-insurstaq-command-comes-from)).

* **Plugin:** install InsurStaq from the Cursor Marketplace (Customize → Plugins), or load this
  folder (`.cursor-plugin/plugin.json` + `mcp.json`) as a local plugin.
* **Without the plugin:** copy the `insurstaq` entry of [`mcp.json`](mcp.json) into
  `~/.cursor/mcp.json` (all projects) or `.cursor/mcp.json` (one project).

Check Settings → MCP: the `insurstaq` server should be enabled with five tools. If Cursor cannot
start it, set `command` to the bundled tool's absolute path (Settings → Coding Agents → Copy
Path, normally `/Applications/InsurStaq.app/Contents/MacOS/insurstaq-cli`) and `args` to `["mcp"]`.

## Use

Ask the agent to verify its work, for example:

> When you are done, run InsurStaq verify on this repository and fix anything BLOCKED.

The agent calls `insurstaq_verify_changes` and receives a verdict with finding ids, severities
and `file:line` locations, then fixes the code itself or queues the finding for you in InsurStaq
with `insurstaq_request_fix`. See the [tool list](../docs/REFERENCE.md#tools).

## Privacy

InsurStaq never uploads your source code. The server returns security metadata (verdict,
severity, category, finding id, short summary, file location), never source code, secrets or
diffs. Cursor itself may send code to its model provider; that is between you and Cursor.
