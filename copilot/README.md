# InsurStaq for VS Code and GitHub Copilot

An MCP configuration that connects VS Code's agent mode (GitHub Copilot) to InsurStaq Guard on
your Mac. It only launches the local InsurStaq MCP server (`insurstaq mcp`, bundled in
InsurStaq.app; see [`.vscode/mcp.json`](.vscode/mcp.json)); everything else happens in InsurStaq.

VS Code and GitHub Copilot are named only to describe compatibility.
No affiliation or endorsement implied.

## Install

Requirements: InsurStaq for Mac (in `/Applications`) with the repository added, and a license
that includes Guard. The app bundles the `insurstaq` command the plugin runs; nothing else to
install ([details](../docs/REFERENCE.md#where-the-insurstaq-command-comes-from)). MCP tools in Copilot
need VS Code's agent mode; availability depends on your Copilot plan and organization policy.

* **One repository:** copy [`.vscode/mcp.json`](.vscode/mcp.json) into the repository's
  `.vscode/` folder (merge the `insurstaq` entry if the file exists).
* **All workspaces:** run **MCP: Open User Configuration** and add the same `insurstaq` entry
  under `servers`.

Start the server from the `mcp.json` editor (or **MCP: List Servers**) and confirm the five
InsurStaq tools appear in the agent's tool picker. If VS Code cannot start it, set
`command` to the bundled tool's absolute path (InsurStaq → Settings → Coding Agents → Copy Path,
normally `/Applications/InsurStaq.app/Contents/MacOS/insurstaq-cli`) and `args` to `["mcp"]`.

## Use

In agent mode, ask Copilot to verify its work, for example:

> After the change, use InsurStaq to verify this repository and fix anything BLOCKED.

Copilot receives a verdict with finding ids, severities and `file:line` locations, then fixes the
code itself or queues the finding for you in InsurStaq with `insurstaq_request_fix`. See the
[tool list](../docs/REFERENCE.md#tools).

## Privacy

InsurStaq never uploads your source code. The server returns security metadata (verdict,
severity, category, finding id, short summary, file location), never source code, secrets or
diffs. Copilot itself may send code to its model provider; that is between you and GitHub.
