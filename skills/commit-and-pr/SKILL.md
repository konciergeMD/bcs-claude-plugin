# Commit and Create PR

Stage changes, commit with a conventional message, and open a pull request.

## Steps

1. Run `git status` and `git diff` (staged + unstaged) to understand what changed.
2. Run `git log --oneline -5` to match the repo's commit style.
3. Stage files:
   - If the user specified files in their message, stage only those.
   - Otherwise, stage all tracked modifications (`git add -u`). Never `git add .` — avoid accidentally staging `.env` or build artifacts.
4. Run `git diff --staged` to confirm what will be committed.
5. Write a conventional commit message (type: feat|fix|chore|refactor|docs|test, short imperative subject, no trailing period). Pass it via heredoc to avoid shell escaping issues.
6. Commit. If a pre-commit hook fails, fix the issue and re-commit — never use `--no-verify`.
7. Push the branch to origin (`git push -u origin HEAD`).
8. **Ask the user before opening the PR.** Show the proposed PR title and body, then ask: "Ready to open this PR? (yes / no / draft)". Wait for confirmation before proceeding.
   - `yes` → open PR normally
   - `draft` → open with `--draft`
   - `no` → stop here; tell the user the branch is pushed and they can open the PR manually
9. Create the PR with `gh pr create` using this body structure:
   ```
   ## Summary
   - <bullet points of what changed and why>

   ## Test plan
   - [ ] <manual or automated steps to verify>

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   ```
10. Return the PR URL.

## Notes
- Do not commit if there are no staged changes after step 3.
- Use `--draft` flag on `gh pr create` if the user says the work is not ready for review.
- Never force-push or amend published commits.
