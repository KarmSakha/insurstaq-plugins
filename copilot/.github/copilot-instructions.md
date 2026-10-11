# InsurStaq Guard (advisory)

This repository is checked by InsurStaq Guard on the developer's Mac through the local
`insurstaq` MCP server. GitHub Copilot in VS Code has no hook support InsurStaq relies on, so these
instructions are **advisory**: nothing stops a turn that skips them.

- Before you report work as done, call `insurstaq_verify_changes` with the repository path.
- `BLOCKED`: fix the listed findings at their file locations, then verify again. Do not report
  the change as checked.
- `UNABLE_TO_VERIFY`: the change is not verified. Say so.
- `PASSED_WITH_LIMITED_COVERAGE`: no blocking regression, but coverage is limited; do not call it
  fully verified.
- For a change to a private path, ask the user, or queue it with `insurstaq_build_delegate`.
- Call `insurstaq_project_rules` for the security rules of this repository's stack.

InsurStaq returns security metadata only (verdict, severity, category, finding id, `file:line`),
never source code. Guard checks are evidence for a person to weigh, not a promise.
