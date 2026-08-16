---
name: repo-to-feishu-wiki
description: Convert Markdown, VitePress, and Quarto content from GitHub or local repositories into Feishu Wiki spaces with navigation, internal links, images, formulas, and Mermaid diagrams. Support one-time imports, incremental mirrors, scheduled synchronization, node reuse across renames, deletion archiving, crash recovery, GitHub Actions, and remote acceptance checks. Use for repository-to-Feishu migrations or mirrors, Markdown imports, GitHub-to-Feishu automation, debugging an existing repository-to-Feishu synchronizer, and equivalent requests in English or Chinese.
---

# Repository to Feishu Wiki

Model repository content as a managed Wiki tree, then write it to Feishu in a verifiable, recoverable way that preserves unknown content by default. Reuse the repository's existing language and test framework; do not introduce a second runtime for a one-off task.

## Load Current Operational Guidance

Read the current installed version of each relevant skill before entering that stage. Treat this skill as orchestration guidance, not as a permanent source of CLI flags:

- Authentication, identity, scopes, and error envelopes: `lark-shared`
- Wiki spaces, members, and nodes: `lark-wiki`
- Docx content reads and overwrites: `lark-doc`
- Titles, resources, permissions, and token inspection: `lark-drive`
- GitHub repositories, Actions, Secrets, or pull requests: `github:github`, when available

Prefer the shortcuts recommended by those skills. Use a raw API only when a shortcut lacks required behavior, and inspect `lark-cli schema` before calling it. Never guess request fields.

## Define Success First

Record these decisions before implementation:

1. Choose one mode: one-time snapshot, continuous one-way mirror, or adoption of an existing Wiki.
2. Define the single source of truth. For a continuous mirror, default to the repository's target branch and warn that manual Feishu body edits will be overwritten.
3. Define content scope, target private space, managed root, language tree, schedule, and deletion policy.
4. Count source pages, synthetic directory nodes, and status/archive helper nodes separately. Do not report total Wiki nodes as the source-file count.
5. Define acceptance evidence: page count, hierarchy, internal links, media, formulas, diagrams, permissions, no-op reruns, and CI results.

Default to a private space, archive-on-delete, serial writes, and preview-before-apply. If the user requests only a plan or diagnosis, do not perform remote writes.

## Workflow

### 1. Audit the Repository and Remote Preconditions

- Read repository instructions such as `AGENTS.md`, inspect worktree state, and preserve existing user changes.
- Use the Git-tracked file list to determine content scope. Identify Markdown/QMD files, frontmatter, directory indexes, local assets, internal links, formulas, Mermaid blocks, and framework-specific containers.
- Select representative pages covering deep paths, images, SVG, tables, formulas, diagrams, special containers, and multilingual links.
- Count broken local assets and internal links. Report source defects before deciding whether to fix the source or degrade only in the conversion layer.
- When real Feishu writes are in scope, inspect the usable `lark-cli` version, required identity, and target visibility. Distinguish missing application scopes from missing target-space ACLs. Never switch identity silently to bypass an error.
- When CI is in scope, inspect GitHub authentication, the target branch, existing workflows, and Secret/Variable names. Check only whether sensitive values are non-empty; never print their values.

### 2. Choose the Smallest Architecture

- One-time snapshot: record path, token, import, and verification progress in a local state file. Separate `prepare -> nodes -> content -> verify` so the run can resume safely.
- Continuous mirror: implement a versioned remote manifest, read-only plan, apply mode, content-plus-asset hashes, rename detection, archiving, and crash recovery. Read [sync-design.md](references/sync-design.md) before implementation.
- Existing Wiki adoption: discover the entire target tree and output a conflict plan first. Never claim a node by title alone; require manifest tokens, an explicit mapping, or a provable interrupted-create checkpoint.

When importing into an existing space, prefer a dedicated managed root. Treat all content outside that root as unknown and unmanaged.

For a one-time import above 20 pages, keep snapshot mode but require resumable state and programmatic full verification. Use the continuous-mirror manifest only when CI, incremental updates, rename synchronization, or deletion/archive behavior is required.

### 3. Build a Stable Content Model

- Use normalized repository-relative paths as stable keys. Never use titles as primary keys.
- Derive titles from frontmatter `title`, then the first H1, then a humanized filename. Strip links, emphasis markers, HTML, and excess whitespace.
- Map a directory `index.md` to its directory homepage. Create a synthetic directory node when a directory needs to hold children but has no index.
- Distinguish the root homepage, language roots, regular pages, synthetic directories, status pages, and archive pages.
- Create or resolve the complete Wiki tree and obtain every `node_token` and `obj_token` before transforming bodies or rewriting internal links.
- Store source path, stable key, parent key, title, and Feishu tokens separately so moves and title changes do not create duplicate pages.

### 4. Transform Content

Protect fenced code blocks before applying transformations so regex-based rewrites cannot corrupt code:

1. Remove frontmatter and the first H1 when it duplicates the Wiki title.
2. Convert VitePress or Quarto Badge, callout, container, and details syntax.
3. Resolve internal links to stable keys, then rewrite them to Wiki node links. Degrade unsupported heading anchors to page links and record warnings.
4. Verify local images, include them in asset hashes, and emit relative `@./...` resource paths according to current `lark-doc` and `lark-drive` guidance.
5. Convert Mermaid to a supported native whiteboard input and formulas to supported math blocks. Convert SVG to a supported bitmap format when necessary.
6. Escape XML/HTML-sensitive characters while preserving code and formulas verbatim.
7. Add a read-only notice, source-file link, and source commit to continuously mirrored pages.

Treat QMD as static source and do not execute code cells by default. Execute them only with explicit user authorization and only when the repository code, data, and runtime are trusted and isolated.

Compute a stable hash from the final body plus the bytes of every referenced asset. Hashing Markdown alone misses changes to image bytes at an unchanged path.

### 5. Apply Remote Writes Safely

- Prefer explicit user identity for initial private-space creation, application membership, and root creation. Use a bot already authorized for the target space for continuous CI.
- Explicitly preserve `--as user` or `--as bot` throughout each token-consuming command chain. Never rely on automatic identity selection.
- Use `lark-wiki` for node creation, movement, and listing; `lark-doc` for body overwrites; and `lark-drive +update-title` for in-place title changes that preserve node tokens.
- Reconcile topology before writing bodies. Save an `in_progress` checkpoint before mutations, and advance the successful commit SHA only after all page and archive operations succeed.
- Serialize writes within one Wiki space. Retry only rate limits and transient network failures with bounded exponential backoff. Stop on invalid parameters, permission failures, not-found responses, and corrupt manifests.
- Determine success from the process exit code or JSON `ok == true`. Do not use the legacy top-level `code == 0` convention.
- Follow the current `lark-shared` confirmation gate for high-risk commands. Never append `--yes` automatically.

### 6. Implement Lifecycle Behavior and CI

- Use Git rename evidence to migrate old source paths to new stable keys while reusing the original node. Never implement a rename as delete plus create.
- Archive only the highest stale root by default, preserving its subtree and history. Delete only manifest-owned nodes after the user explicitly confirms exact targets.
- Fail closed on unknown children, duplicate titles, manifest corruption, or token/parent drift. Never guess or recursively delete unknown content.
- Provide local conversion checks, a remote read-only `plan` or `dry-run`, incremental apply, and forced full modes.
- Pin tested runtime and `lark-cli` versions in CI. Use read-only repository permissions, full Git history, relevant path filters, a non-canceling single-concurrency queue, and an always-uploaded synchronization report.
- Store App ID/Secret in GitHub Secrets and space/root tokens in Variables. Pass the Secret through stdin or environment variables, never through arguments, files, or logs.
- Convert local schedule times explicitly to UTC and note that GitHub scheduled runs may be delayed.

### 7. Test and Verify Remotely

Read [verification.md](references/verification.md) before writing tests and again before delivery. Cover transformations, the state machine, CI, and live Feishu acceptance.

At minimum, verify:

- Every expected page has exactly one managed node with the correct parent.
- Titles contain no leftover Markdown markers, bodies are readable, and images became remote resources.
- Every expected internal link targets the correct Wiki node; unresolved or degraded links appear in the report.
- The Wiki remains private and external sharing matches the target policy.
- An unchanged incremental rerun is a no-op; one source change updates only affected pages; asset-byte changes trigger updates.
- Renames preserve tokens, deletions enter the archive, interrupted runs resume without duplicate nodes, and failures do not advance the successful SHA.
- GitHub Actions succeeds on a real runner and logs contain no Secret or access token.

Never substitute local test success for live acceptance, and never claim full success from sampling alone. Use sampling to catch early defects, then programmatically reconcile every page, link, and resource count.

## Safety Boundaries

- Never expose App Secrets, access tokens, or authorization codes in code, state, reports, command arguments, patches, or responses.
- If a secret appeared in chat, terminal output, or repository history, finish the necessary repair and explicitly recommend immediate rotation.
- Do not modify unrelated source content or overwrite existing uncommitted user changes.
- Never delete unknown nodes or nodes without manifest ownership. A matching title is not proof of ownership.
- Do not perform batch writes before confirming the target space and deletion/overwrite policy.

## Delivery Format

Lead with the result, then list verifiable evidence: Wiki URL, source commit, counts for source pages/synthetic nodes/helper nodes, asset and link counts, tests, CI run, incremental statistics, and remaining warnings. If live writes or remote acceptance did not happen, state that boundary and do not say the synchronization is complete.
