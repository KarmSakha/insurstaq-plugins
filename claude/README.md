# InsurStaq for Claude Code

A Claude Code plugin that connects Claude Code to InsurStaq Guard on your Mac. It only launches
the local InsurStaq MCP server (`insurstaq mcp`, bundled in InsurStaq.app) through the committed
launcher [`scripts/insurstaq-mcp`](scripts/insurstaq-mcp) (see [`.mcp.json`](.mcp.json));
everything else happens in InsurStaq. If InsurStaq is not installed, the launcher says so and
exits.

Claude Code is named only to describe compatibility. No affiliation or endorsement implied.

## Install

Requirements: InsurStaq for Mac 1.0.6 or later (in `/Applications`) with the repository added,
and a license that includes Guard. The app bundles the `insurstaq` command the plugin runs; nothing else to
install ([details](../README.md#where-the-insurstaq-command-comes-from)).

Without any marketplace: in InsurStaq, open **Settings → Coding Agents** and choose **Connect…**
next to Claude Code, or run the command in
[Install without a marketplace](../README.md#install-without-a-marketplace).

From this repository's marketplace, inside Claude Code:

```text
/plugin marketplace add KarmSakha/insurstaq-plugins
/plugin install insurstaq@insurstaq
```

To try it from a checkout:

```sh
claude --plugin-dir ./claude
```

Run `/mcp` in Claude Code to confirm the `insurstaq` server is connected and its five tools are
listed.

## Use

Ask Claude to check its work, for example:

> After you finish, verify the changes with InsurStaq and fix anything it blocks.

Claude calls `insurstaq_verify_changes` with the repository path and gets back `PASSED`,
`PASSED_WITH_WARNINGS`, `BLOCKED` or `UNABLE_TO_VERIFY` with finding ids, severities and
`file:line` locations. It can then open those files itself, or call `insurstaq_request_fix` to
queue the finding for you to review in the InsurStaq app. See the [tool list](../README.md#tools).

## Privacy

InsurStaq never uploads your source code. The server returns security metadata (verdict,
severity, category, finding id, short summary, file location), never source code, secrets or
diffs. Claude Code itself may send code to its model provider; that is between you and Claude
Code.
