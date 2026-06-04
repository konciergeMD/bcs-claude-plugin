# Release messaging-model

Automate the `messaging-model` release process across all three language sub-projects (Java, TypeScript, Python).

## Prerequisites

- Clean working tree on `main` (or the release branch)
- `gh` CLI authenticated and pointing to `konciergeMD/messaging-model`
- The new version follows semantic versioning: `X.Y.Z`

## Steps

1. **Determine the version** — if the user provided a version in the message, use that (validate it matches `\d+\.\d+\.\d+`). Otherwise, infer it automatically:
   ```bash
   # Get the latest tag from git
   git fetch --tags
   git tag --sort=-version:refname | head -1
   ```
   Take the latest tag, bump the **patch** segment by 1 (e.g. `1.29.9` → `1.30.0`... wait, patch: `1.29.9` → `1.29.10`). Then **confirm with the user** before proceeding:
   > "Latest release is `X.Y.Z`. I'll release `X.Y.(Z+1)`. Options: confirm / bump minor (`X.(Y+1).0`) / snapshot (`X.Y.(Z+1)-SNAPSHOT`) / provide a version"
   Wait for confirmation. If the user wants a different bump (minor, major, or snapshot), use that version instead. Validate it matches `\d+\.\d+\.\d+(-SNAPSHOT)?`.\d+`.

2. **Update TypeScript** — edit `ts/package.json`: set `"version"` to the new version.

3. **Update Java** — edit `java/pom.xml`: update the `<version>` tag directly under `<project>` (not under `<parent>` or `<dependency>`) to the new version.

4. **Python** — no file change needed; version is derived automatically from the git tag by `setuptools-git-versioning`.

5. **Commit the version bump:**
   ```bash
   git add ts/package.json java/pom.xml
   git commit -m "chore: release v<VERSION>"
   ```

6. **Push the commit:**
   ```bash
   git push origin HEAD
   ```

7. **Create GitHub release + tag** (use `--draft` if the user wants to review first):
   ```bash
   gh release create <VERSION> \
     --title "v<VERSION>" \
     --generate-notes \
     --repo konciergeMD/messaging-model
   ```

8. **Print Jenkins pipeline links** for the user to manually trigger each language build (requires VPN + CloudBees access):
   - Java: https://cicd.accolade.dev/better-call-saul/job/components/job/messaging-model/job/java-messaging-model/view/tags/
   - Python: https://cicd.accolade.dev/better-call-saul/job/components/job/messaging-model/job/messaging-model/view/tags/
   - TypeScript: https://cicd.accolade.dev/better-call-saul/job/components/job/messaging-model/job/ts-messaging-model/view/tags/

## Notes

- The `java/pom.xml` version to update is the one at the top of the file under `<project><version>`, not a dependency version.
- Never force-push or move existing tags.
- If the user passes `--draft`, add `--draft` to the `gh release create` command so the release isn't published immediately.
- After the GitHub release is created, remind the user to start each Jenkins pipeline for the tag — the pipelines do **not** trigger automatically.

## SNAPSHOT releases

If the version ends with `-SNAPSHOT` (e.g. `1.30.0-SNAPSHOT`):
- **Java**: set `pom.xml` version to `1.30.0-SNAPSHOT` — Maven handles snapshots natively. Skip the GitHub release and tag creation; instead just commit and push. Remind the user to trigger the Java Jenkins pipeline manually.
- **TypeScript**: set `package.json` version to `1.30.0-snapshot` (lowercase, npm convention). Same — no GitHub release, just commit and push.
- **Python**: no file change needed (version comes from git tag). Since there's no tag, the version will be derived as a dev build automatically — no action required.
- **Do not create a GitHub release or tag** for snapshots. Just commit, push, and print a note explaining the above.
