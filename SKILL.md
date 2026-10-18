---
name: android-apk-repair
description: Review and repair Android applications from source repositories or APKs, especially WebView/hybrid apps; apply focused bug fixes, verify persistence and UI behavior, prepare Git changes, and build/sign APKs when the complete project and signing setup are available. Use when asked to inspect, modify, test, commit, or build an Android app/APK.
---

# Android APK Repair

Use this skill for authorized Android app maintenance. Prefer the real source project; treat an APK as a diagnostic or asset-recovery source, not as a substitute for the complete Android project.

## Workflow

1. **Confirm scope and access.** Identify the requested defect/feature, supplied APK/source/repository, intended output (patch, branch, APK), and whether the user asked for any remote write. Do not infer permission to commit, push, publish, or replace an existing release from a repository link alone.
2. **Inspect the repository before editing.** Check `git status`, current branch, remotes, recent commit, tracked file tree, README, and available branches/tags. Locate the actual Android module and web assets. If the repository is empty or only has a README, state that clearly; do not claim it contains application source or modify it as though it did.
3. **Inspect the APK if useful.** Record file size/type and inspect archive entries (`unzip -l`). Identify package/version and assets where tooling permits (`apkanalyzer`, `aapt`, `apktool`, `jadx`). For a WebView app, extract only the relevant web assets into a working copy. Do not alter the user's input APK in place.
4. **Understand the code path.** Trace the full mutation flow: validation, in-memory update, persistence API, failure path, success toast, screen transition, and any caller/form behavior. Determine whether persistence is synchronous or asynchronous and what its return value actually guarantees. Do not add `await` or trust a boolean without inspecting the bridge/API contract.
5. **Plan the smallest safe fix.** Preserve prior data on write failure where feasible, prevent success UI/navigation until durable persistence is confirmed, and ensure the UI does not retain stale references after rollback. Handle all relevant call sites consistently, but avoid blind global text replacements. For a mutation form, keep validation errors distinct from persistence errors and prevent a failed submit from being treated as a success.
6. **Implement in a working branch/copy.** Keep the original intact; use a focused branch for source projects. Do not expose signing keys, passwords, customer data, or other secrets in source control, logs, patches, or chat.
7. **Verify the change.** Run the project's tests and lint/build commands when available. For inline JavaScript, extract/check scripts with `node --check`. Exercise both persistence success and failure, confirming: failure returns/propagates as failure, in-memory state is not falsely presented as saved, no success toast or navigation occurs, and success behavior remains unchanged. Report tests not possible in the environment.
8. **Review and deliver.** Inspect `git diff --check` and the full diff. Provide changed files, tests, and limitations. If source is unavailable, deliver a clearly labeled extracted-asset patch—not an APK—and explain how it can be integrated. Never claim a build, test, commit, or push that did not happen.
9. **Commit, push, or publish only when explicitly requested.** Prefer a named branch and reviewable diff. Before pushing, confirm the remote/branch and that credentials are available without printing secrets. If the user asked only how to do it, provide commands rather than performing remote writes.

## Persistence-success bug pattern

For a synchronous persistence method that returns `true` only after storage succeeds, use this general shape (adapt to the application's state model and error handling):

```js
if (!save()) return;
toast('Saved successfully');
routeToNextScreen();
```

The guard alone is insufficient if failed mutations remain in memory. Maintain or capture a last-known-good state and restore it on failure, or implement a transaction that commits only after persistence succeeds. If rollback replaces the state object, re-render/rebind UI handlers that captured references to old objects. For async persistence, await the actual durable operation and handle rejection before success UI:

```js
try {
  await save();
} catch (error) {
  showSaveFailure(error);
  return;
}
toast('Saved successfully');
routeToNextScreen();
```

Keep error messages actionable and avoid swallowing failures. A platform bridge response must be checked according to its documented contract.

## APK and signing constraints

- A standalone APK generally cannot be safely turned back into a maintainable Android/Gradle project just by replacing an HTML asset. Establish the source project structure, build scripts, native wrapper, package ID, and dependencies before attempting a rebuild.
- Use the project's Gradle wrapper where present, for example `./gradlew test` and `./gradlew assembleDebug`. Use release tasks only after inspecting the configured signing process.
- An update install normally requires the same application ID and signing certificate as the installed release, plus a higher version code. Never overwrite or remove a user's existing app/data as a workaround. Explain that a missing original signing key prevents a compatible in-place update; do not ask the user to paste key material into chat.
- Verify produced APKs with available Android tooling, and distinguish a debug APK from a signed release APK.

## Git command guidance

When preparing commands for a user, use the discovered asset path rather than guessing `app/src/main/assets`. A safe pattern is:

```bash
git status
git switch -c fix/<short-description>
# Copy the edited file to the verified source path.
git diff --check
git diff -- <verified-path>
git add <verified-path>
git commit -m "fix: <short description>"
git push -u origin fix/<short-description>
```

Only include `git push` if the user requested uploading; otherwise leave changes local and summarize how to push them.

## Completion summary

Report: source/repository status; files changed; behavioral fix; verification performed and results; whether an APK was actually built/signed; and any missing source/signing requirements. Link deliverable files using their exact absolute paths.
