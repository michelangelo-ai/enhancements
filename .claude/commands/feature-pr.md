# /feature-pr

Format and create a GitHub PR using the feature-proposal template.

## What this does

Reads `.github/ISSUE_TEMPLATE/feature-proposal.md`, fills it in from the current branch's
changes and git context, then creates a PR with the formatted body.

## Steps

1. Run `git diff main...HEAD` and `git log main..HEAD --oneline` to understand what changed.
2. Read `.github/ISSUE_TEMPLATE/feature-proposal.md` for the template structure.
3. Fill in each section from the diff and commit context:
   - **Problem statement** — what gap or breakage the change addresses; frame as "As a [user type]..."
   - **Proposed solution** — what the PR does, with bullet points for key decisions
   - **Alternatives considered** — at least one alternative with tradeoffs
   - **Acceptance criteria** — testable checklist items derived from the change; mark already-satisfied items checked
   - **Additional context** — link related RFCs, issues, or design docs found in the diff
   - **Checklist** — fill in the three standard items
4. Set the PR title to `[FEAT] <short description>` matching the change.
5. Create the PR:
   ```
   gh pr create --title "[FEAT] <title>" --body "<filled template>" --base main
   ```
6. Print the PR URL.

## Notes

- If `$ARGUMENTS` is provided, use it as the PR title (strip `[FEAT]` prefix if already present).
- Do not invent acceptance criteria that can't be verified from the diff — only include what the code actually does.
- If an RFC file is being added, link it in Additional context.
