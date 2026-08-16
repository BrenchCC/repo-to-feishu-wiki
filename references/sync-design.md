# Continuous Synchronization Design

## Distilled Lessons

Three production-shaped tasks established these hard requirements:

- A one-time book import exposed an H1 wrapped in emphasis markers only during page-by-page remote verification. A successful body write is insufficient; verify titles, hierarchy, assets, and links.
- A scheduled bilingual mirror passed all local checks but still exposed environment-scope and empty-Secret failures on a real CI runner. Execute the workflow and verify non-empty injection without printing values.
- A large incremental mirror achieved a complete no-op through body-plus-asset hashes. Markdown-only hashes cannot detect changed bytes at an unchanged image path.

Treat full remote acceptance, real CI closure, asset hashing, and recoverable state as product features rather than post-launch extras.

## State Manifest

Use a versioned, machine-readable manifest. Store it in a dedicated status page or a clearly delimited root-page block; a dedicated page better isolates human content.

```json
{
  "schemaVersion": 1,
  "status": "idle",
  "targetCommit": "",
  "lastSyncedCommit": "<git-sha>",
  "pages": {
    "<stable-key>": {
      "sourcePath": "docs/example.md",
      "parentKey": "docs",
      "title": "Example",
      "nodeToken": "<wiki-node-token>",
      "objToken": "<docx-token>",
      "renderHash": "<sha256>",
      "revisionId": 1
    }
  },
  "archives": [],
  "lastReport": {}
}
```

Treat stable key, parent key, node/object tokens, title, source path, render hash, revision, and last successful commit as the minimum state. Never store secrets.

## Synchronization State Machine

1. Build the desired content tree and validate unique keys, existing parents, and readable source assets.
2. Discover the remote tree and manifest; validate schema, space ID, tokens, object types, and parent relationships.
3. Compute a read-only plan: create, recover, move, rename, update, archive/delete, or skip.
4. Persist `in_progress`, `targetCommit`, and recoverable create metadata.
5. Create or recover nodes parent-first and persist every token.
6. Move and rename nodes in place, then revalidate topology.
7. Transform links and assets against the complete node map and compute render hashes.
8. Overwrite only new, forced-full, hash-changed, or revision-drifted bodies.
9. Archive the highest stale roots. If deletion is explicitly selected, delete deepest-first and only when the manifest owns the node and no unknown children exist.
10. Reconcile remote results, then persist the chosen terminal status and `lastSyncedCommit`.

On any failure, retain `in_progress` and the previous `lastSyncedCommit` so the next run can identify unfinished work. Choose one terminal status value, such as `idle` or `complete`, and use it consistently within an implementation.

## Create Recovery

Node creation has a window where the remote call succeeds before state persistence. Use either strategy:

- Persist a unique `pendingCreateKey` before creation. Recover only an exact parent plus exact staging title associated with that key.
- Create with a deterministic staging title containing a short stable-key hash, checkpoint all tokens, then rename to final titles.

Never claim a same-title node by final title alone. Stop and report a conflict when multiple candidates exist.

## Moves and Deletions

- Use Git rename evidence such as `git diff --name-status -M <last>..<current>` to migrate state while preserving node tokens and history.
- Without complete Git history, do not guess a rename. Report delete/create risk and require full history or an explicit mapping.
- Move stale content to a dedicated archive node by default and attach its source commit.
- Archive only stale roots that have no stale ancestor, avoiding repeated movement of one subtree.
- For physical deletion, reverse the order: process deepest leaves first and reject a managed parent containing unknown children.

## Render Hash

Feed these values into SHA-256 in stable order:

1. Converter or schema version.
2. Final body with normalized line endings.
3. Normalized relative path of each local asset.
4. Byte hash of each asset.
5. Output-affecting page metadata and resolved link targets.

Sort assets deterministically. Exclude temporary absolute paths, timestamps, and randomness, or an unchanged run cannot become a no-op.

## Writes, Concurrency, Identity, and Ownership

- Serialize creates, moves, title updates, and body overwrites within one Wiki space.
- Apply bounded exponential backoff only to rate limits, gateways, and transient network failures.
- Stop and classify invalid parameters, not-found responses, permission failures, revision conflicts, and manifest corruption.
- For revision drift, explicitly overwrite in a one-way mirror. If the user requests bidirectional synchronization, stop and define a new conflict model; this skill does not default to bidirectional merging.
- Use user identity to create a private space and root, then add the application by App ID. Use bot identity for CI and explicitly preserve that identity on every command consuming bot-derived tokens.
- Validate application scopes and target-space membership/admin ACL independently. Passing one check does not prove the other.
- Treat a changed App ID or target space as a new deployment. Do not copy an old-space manifest into a new space.
