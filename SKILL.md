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

## User-Learned Best Practices & Constraints

> **Auto-Generated Section**: This section is maintained by `skill-evolution-manager`. Do not edit manually.

### User Preferences
- 当源站使用多语言侧边栏且用户要求独立维护时，将每种语言建模为同一知识空间中的独立一级 Wiki 文档，并优先保留既有语言根节点 token。
- 知识空间与根文档简介应聚焦内容用途；同步、覆盖与控制信息放在正文末尾来源引用或系统控制页，不占用内容简介。
- 持续镜像的日常内容同步默认使用 incremental；full 用于首次播种拓扑签名、拓扑变更后的协调或定期完整审计，不应把提高超时时间当作加速方案。

### Known Fixes & Workarounds
- VitePress 目录顺序应只从 sidebar 数组提取，不能混入 nav；目录节点以自身或后代页面在 sidebar 中的首次出现位置排序，未列出页面稳定追加到末尾。
- 飞书移动节点 API 没有同级位置参数；需要重排时，只移动清单归属且不在可保留前缀中的节点到安全暂存节点，再按目标顺序移回，并逐父节点校验完整 token 序列；出现未知节点立即失败关闭。
- 状态清单必须强制 node_token 唯一归属。多个同名 index 目录可能误领同一节点；应释放重复映射、为 token 预留唯一 stable key、创建缺失目录并将已有子节点移回正确父节点。
- 临时暂存中断后，恢复逻辑必须核对远端实际父节点成员关系，不能只比较状态清单中的 parentKey。
- 独立语言根迁到知识空间根级时，把状态页和归档页收纳到主语言根下的系统节点，避免污染面向读者的侧边栏顺序。
- 同步耗时异常时，先按执行日志的时间戳和 action 分类量化瓶颈，区分正文覆盖、节点移动与无写入时仍发生的串行层级读取；大量 move 通常是一次性重排成本，而无更新仍耗时通常说明全树扫描是瓶颈。
- 大型 Wiki 的内容增量可在清单中持久化版本化拓扑签名；签名至少覆盖 stable key、parent key、标题、source path、synthetic 标记与 sidebar 顺序。仅当 incremental、状态为 idle、已有成功 commit、签名一致、node/obj token 映射完整且 node_token 唯一、根节点和系统辅助节点匹配时，才跳过远端全树快照、拓扑协调与同级排序。
- 拓扑快速路径必须失败关闭：旧状态缺少签名、上次同步中断、映射缺失或重复、根节点异常、页面增删/重命名/移动、标题或 sidebar 顺序变化，以及 full 模式，均自动回退完整拓扑校验；首次升级应完整协调一次，成功后再写入签名。
- 快速路径仍须在正文写入前持久化 in_progress 与 targetCommit，正文写入保持同一 Wiki 空间内串行；失败时不得推进 lastSyncedCommit，下次因非 idle 状态自动完整恢复。只读层级查询可做有界并发，但不能用并发写入换速度。
- 飞书 Markdown 公式不能沿用 XML `<latex>...</latex>` 包装或额外转义反斜杠；转换层应输出飞书原生 `$$...$$` 公式块，并让源 LaTeX 在进入公式归一化前保持原样。
- 飞书 Markdown 转换会移除 `\text{...}` 和 `\texttt{...}` 内容中的 `\_` 转义，导致 `hidden_size` 等公式失效；上传前应在这些命令内部把 `\_` 归一化为 `\textunderscore{}`，并保留真实失败公式作为回归测试。
- 公式兼容性不能只靠抽样或本地 KaTeX 解析判断；应扫描全部待同步文档，对转换后的每个公式执行严格解析，再对代表性高风险公式做飞书写入与回读，确认公式告警为零且回读内容未被再次改写。
- 当同步测试新增 KaTeX 等直接导入的 Node 包时，必须同时更新 `devDependencies` 与 lockfile，并确保 GitHub Actions 在测试前执行 `npm ci`；`actions/setup-node` 只配置运行时与缓存，不会安装依赖。
- 飞书公式载体必须以当前已安装 lark-doc 对目标 doc-format 的参考为准，不能沿用历史的“Markdown 一律使用 $$”或“XML 一律不可用”结论；若当前 Markdown 导入允许内嵌 XML，且 lark-doc XML 参考将 <latex>...</latex> 列为公式标签，display math 应输出独立的 <latex> 块，并对公式正文做 XML 转义。
- 修复公式兼容性时，应保留 Quarto 源文件的原生 display math；仅在飞书转换层选择当前受支持的载体。对每个新增公式规则加入转换回归测试，并检查实际生成的飞书写入文本；对于比较符等高风险符号，优先采用等价且不含歧义符号的数学记法。
- 当当前飞书转换路径使用 <latex>…</latex> 时，公式正文不可做 XML 转义：飞书会把 &amp; 原样交给 LaTeX，破坏 aligned 环境的 &。写入前应将公式内的换行及周围空白规整为单个空格，防止 \mid 与下一行变量粘连为未定义命令 \midp；此规则覆盖此前“对 <latex> 正文做 XML 转义”的经验。

### Custom Instruction Injection

处理多语言 VitePress 仓库时，先盘点 sidebar 顺序、语言根边界、远端实际父子关系与 node_token 唯一性；预览创建、拆分、移动和重排数量并取得确认后再写入。简介保持内容导向，控制元数据收纳在系统页或页尾。对大型持续镜像先从执行日志量化读取、移动与正文写入成本；日常 incremental 在严格校验拓扑签名和 token 清单后走内容快速路径，任何状态、拓扑或映射异常立即回退完整协调。写操作保持串行且先写 in_progress 检查点，成功后才推进 lastSyncedCommit。公式转换应以飞书实际 Markdown 解析行为为准：输出原生公式块，在 `\text{}`/`\texttt{}` 内将 `\_` 归一化为 `\textunderscore{}`，用全量严格解析和代表性远端写入回读共同验收。新增测试依赖时同步更新清单与 lockfile，并保证 CI 在运行测试前完成干净安装。