# Verification and Release Checklist

## Contents

- Local content tests
- State machine and fake-remote tests
- CI checks and Feishu preflight
- Full remote, incremental, and recovery acceptance
- Common failure triage
- Completion gate

## Local Content Tests

Cover these inputs and assert deterministic output:

- Frontmatter, title precedence, H1 wrapped in emphasis or links, and files without H1.
- Directory `index.md`, synthetic directories, language roots, and deep paths.
- QMD with code execution disabled by default; when explicitly enabled, verify runtime isolation and deterministic artifacts.
- Markdown and HTML images, changed bytes at the same path, missing assets, and SVG conversion.
- Inline formulas, formula blocks, Mermaid, multiple code fences, tables, and literal angle brackets.
- VitePress Badge/container/details and Quarto callouts.
- Relative links, absolute internal links, directory links, omitted extensions, query strings, heading anchors, and missing targets.
- Windows/Unix line endings, URL-encoded paths, XML escaping, and code-fence protection.

Run identical input twice and require identical body output, asset lists, node models, and render hashes.

## State Machine and Fake-Remote Tests

Use a fake executor or mocked `lark-cli` to cover:

- Initial creation followed by an unchanged no-op.
- One body change, an asset-only change, and forced full synchronization.
- Title changes, directory moves, Git renames, and cross-parent moves.
- Archive-on-delete, explicitly confirmed physical deletion, and refusal to delete unknown children.
- Interruption after a successful create, checkpoint recovery, and fail-closed same-title conflicts.
- Schema mismatch, corrupt manifest, token drift, and revision conflict.
- Successful rate-limit retry, retry exhaustion, no retry for permission failures, and no successful-SHA advance after failure.

Assert this mutation order: checkpoint, topology, bodies, archive/delete, terminal state.

## CI Checks

- Parse workflow YAML and run an available Actions linter.
- Pin the runtime, dependencies, and `lark-cli` version.
- Fetch complete Git history for rename support and use `contents: read`.
- Restrict trigger paths and expose `workflow_dispatch` plan/apply or incremental/full modes.
- Use one concurrency group with `cancel-in-progress: false`.
- Check required variables for non-empty values before CLI setup without printing them.
- Define GitHub Actions environment variables at the correct step/job scope; do not expect shell-local variables from one step to reach another.
- Upload a machine-readable report on success or failure with a sensible retention period.
- Run the workflow on a real runner. Local-shell success does not replace this check.

## Feishu Preflight

1. Confirm target space name, description, privacy, and external-sharing policy.
2. Confirm explicit user and bot identity selection.
3. Confirm the bot has required application scopes and target-space member/admin access.
4. Run local conversion checks.
5. Run a remote read-only plan or dry run and confirm every proposed action.
6. Apply a representative small batch and inspect titles, bodies, images, formulas, Mermaid, and internal links.
7. Run the full synchronization.

## Full Remote Acceptance

Traverse every managed node programmatically rather than relying on samples:

- Compare expected and actual node-token sets, parent tokens, titles, and object types.
- Fetch every Docx, check a stable content marker or source link, and record the revision.
- Extract expected Wiki targets from final converted bodies and compare them with remote link targets page by page.
- Compare expected image occurrences with remote image blocks and, when useful, compare unique-resource counts separately.
- Verify representative formula and Mermaid blocks are readable and editable remotely.
- Report source pages, synthetic directories, status page, archive page, and total nodes separately.
- Read space permissions and confirm privacy and external-sharing settings.

## Incremental and Recovery Acceptance

Run and preserve reports for these scenarios:

1. Unchanged rerun: zero body create/update/archive/delete operations.
2. One page edit: update only that page and truly dependent generated pages.
3. Asset-byte edit: update every page referencing the asset.
4. File rename: preserve the node token while updating path and title.
5. Test-page deletion: archive or delete according to policy without affecting unknown nodes.
6. Forced interruption: resume without duplicate nodes.
7. Injected failure: retain unfinished state and do not advance `lastSyncedCommit`.

## Common Failure Triage

| Symptom | Check first |
|---|---|
| Local success, CI setup failure | Non-empty Secret, environment scope, pinned CLI version, shell differences |
| Bot has scopes but cannot find space | Application membership, explicit bot identity, correct space ID |
| Node API returns not found | Confused space ID, node token, object token, or identity |
| Title contains `**` or link syntax | Inline Markdown stripping and page-by-page title verification |
| Image path exists but page did not update | Asset bytes in render hash and cwd-relative `@./` paths |
| Internal links still target repository | Complete node map before conversion and directory/index normalization |
| Rerun creates duplicate nodes | Title-only adoption or missing create checkpoint/staging title |
| Deletion may affect unknown content | Manifest ownership, unknown-child discovery, archive-first policy |
| Synchronization is slow or rate-limited | Serial writes, hash-based skips, bounded retries |

## Completion Gate

Claim completion only when all applicable conditions hold:

- Local tests and static checks pass.
- Full remote node, body, link, and asset reconciliation passes.
- Privacy and permissions pass.
- An unchanged incremental run and at least one changed-content scenario pass.
- A real CI runner succeeds and produces a synchronization report.
- Repository and log scans reveal no Secret or token.

For an explicitly one-time import without CI, omit only the CI condition. Keep full remote acceptance and resumable local state.
