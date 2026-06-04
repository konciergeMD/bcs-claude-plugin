# Release messaging-model

Automate the `messaging-model` release process across all three language sub-projects (Java, TypeScript, Python).

## Prerequisites

- `gh` CLI authenticated and pointing to `konciergeMD/messaging-model`
- The new version follows semantic versioning: `X.Y.Z`

## Steps

1. **Determine the release type and version:**

   First, identify the release type (final, beta, or snapshot). If not provided in the message, ask:
   > "What type of release? (final / beta / snapshot)"

   Then determine the base version (`X.Y.Z`):
   - If the user provided a version, use it.
   - Otherwise, infer automatically:
     ```bash
     git fetch --tags
     git tag --sort=-version:refname | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | head -1
     ```
     Take the latest **final** release tag and bump patch by 1 (e.g. `1.29.9` → `1.29.10`).

   Build the full version string based on type:
   - **final**: `X.Y.Z` — must be on `main` branch
   - **beta**: `X.Y.Z-beta.N` — must be on a feature branch. Find N by checking existing beta tags for this base version:
     ```bash
     git tag --sort=-version:refname | grep "^X.Y.Z-beta\." | head -1
     ```
     If none exist, N = 1. Otherwise N = latest + 1.
   - **snapshot**: `X.Y.Z-SNAPSHOT` — see SNAPSHOT section below.

   **Confirm with the user** before proceeding:
   > "I'll release `<full-version>` from branch `<current-branch>`. Options: confirm / bump minor / provide a version"

2. **Branch check:**
   - For **final**: verify current branch is `main`. If not, warn and ask for confirmation to continue.
   - For **beta**: verify current branch is NOT `main`. If it is, abort with an error — beta releases must come from a feature branch.

3. **Update TypeScript** — edit `ts/package.json`: set `"version"` to the full version string.

4. **Update Java** — edit `java/pom.xml`: update the `<version>` tag directly under `<project>` (not under `<parent>` or `<dependency>`) to the full version string.

5. **Python** — no file change needed; version is derived automatically from the git tag by `setuptools-git-versioning`.

6. **Commit the version bump:**
   ```bash
   git add ts/package.json java/pom.xml
   git commit -m "chore: release v<FULL-VERSION>"
   ```

7. **Push the commit:**
   ```bash
   git push origin HEAD
   ```

8. **Create GitHub release + tag:**
   ```bash
   gh release create <FULL-VERSION> \
     --title "v<FULL-VERSION>" \
     --generate-notes \
     --repo konciergeMD/messaging-model \
     [--prerelease]   # add this flag for beta releases
   ```
   Use `--prerelease` for beta. Use `--draft` if the user requested it.

9. **Print Jenkins pipeline links** for the user to manually trigger each language build (requires VPN + CloudBees access):
   - Java: https://cicd.accolade.dev/better-call-saul/job/components/job/messaging-model/job/java-messaging-model/view/tags/
   - Python: https://cicd.accolade.dev/better-call-saul/job/components/job/messaging-model/job/messaging-model/view/tags/
   - TypeScript: https://cicd.accolade.dev/better-call-saul/job/components/job/messaging-model/job/ts-messaging-model/view/tags/

## Notes

- The `java/pom.xml` version to update is the one at the top of the file under `<project><version>`, not a dependency version.
- Never force-push or move existing tags.
- After the GitHub release is created, remind the user to start each Jenkins pipeline for the tag — the pipelines do **not** trigger automatically.

## SNAPSHOT releases

If the version is a SNAPSHOT (e.g. `1.30.0-SNAPSHOT`):
- **Java**: set `pom.xml` version to `1.30.0-SNAPSHOT` — Maven handles snapshots natively. Skip the GitHub release and tag creation; just commit and push. Remind the user to trigger the Java Jenkins pipeline manually.
- **TypeScript**: set `package.json` version to `1.30.0-snapshot` (lowercase, npm convention). Same — no GitHub release, just commit and push.
- **Python**: no file change needed (version comes from git tag). Since there's no tag, the version will be derived as a dev build automatically.
- **Do not create a GitHub release or tag** for snapshots. Just commit, push, and print a note explaining the above.
