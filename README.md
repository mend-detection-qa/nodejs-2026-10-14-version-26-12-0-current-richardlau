# bundled-pm-node26

Probe pattern: `bundled-pm-node26`
Generated: 2026-10-14

## Feature exercised

This probe exercises three npm behavior areas that changed across
bundled-npm versions, triggered by the Node.js 26.12.0 (Current)
release on 2026-10-14:

1. **`engines.node` constraint** — the root project declares
   `"engines": {"node": ">=26.12.0"}`. Mend must surface this
   constraint without using it as a filter that removes packages.
2. **Peer dependency resolution (npm 7+ auto-install)** — `react-dom`
   declares `react` as a peer dependency. npm 7+ automatically
   installs the peer; the peer appears in the lockfile under
   `node_modules/react`. The workspace package `@probe/ui` depends on
   `use-sync-external-store`, which also peers on `react`. Mend must
   correctly represent both peer relationships.
3. **npm workspace** — the root declares `"workspaces": ["packages/ui"]`.
   The workspace package `@probe/ui` appears in the lockfile under
   `packages/ui` (a non-`node_modules/` key in the v3 `packages`
   block). Mend must detect it as `source: local` and attribute its
   dependencies correctly.

Lockfile: `lockfileVersion: 3` (npm 8+, as bundled with Node.js
26.12.0 via npm 10.9.2). No legacy `dependencies` block — only the
`packages` block is present.

## Package graph

```
bundled-pm-node26 (root, engines.node >= 26.12.0)
├── @probe/ui@1.0.0 (workspace: packages/ui, source: local)
│   └── use-sync-external-store@1.2.0 (peers: react >=16.8 || ...)
├── react@18.2.0
│   └── loose-envify@1.4.0
│       └── js-tokens@4.0.0
└── react-dom@18.2.0 (peers: react ^18.2.0)
    ├── loose-envify@1.4.0 (deduped)
    └── scheduler@0.23.0
        └── loose-envify@1.4.0 (deduped)
```

## Expected dependency tree

All packages are `source: registry` except `@probe/ui` which is
`source: local` (workspace). All packages are `group: main`.
`loose-envify` is deduplicated to a single entry shared by react,
react-dom, and scheduler. Peer dependency edges (`react-dom -> react`,
`use-sync-external-store -> react`) are represented as entries in
`react-dom.dependencies` and inferred from the lockfile's
`peerDependencies` fields.

| Package | Version | Source | Group | Parent(s) |
|---|---|---|---|---|
| @probe/ui | 1.0.0 | local | main | root |
| js-tokens | 4.0.0 | registry | main | loose-envify |
| loose-envify | 1.4.0 | registry | main | react, react-dom, scheduler |
| react | 18.2.0 | registry | main | root, react-dom (peer) |
| react-dom | 18.2.0 | registry | main | root |
| scheduler | 0.23.0 | registry | main | react-dom |
| use-sync-external-store | 1.2.0 | registry | main | @probe/ui |

## Mend config

Bucket A — `js-npm` has no dynamic version detection from the
manifest. This probe ships `.whitesource` with exact pins:

```json
{
  "scanSettings": {
    "configMode": "AUTO",
    "versioning": {
      "node": "26.12.0",
      "npm": "10.9.2"
    }
  }
}
```

`configMode` is `"AUTO"` because no `whitesource.config` is present
in the probe root. The node version is pinned to `26.12.0` to
reproduce the exact bundled-npm conditions this probe was built
against. The npm version `10.9.2` matches the npm bundled with Node
26.x.

## Mend failure modes targeted

- Workspace `@probe/ui` reported as `source: registry` instead of
  `source: local` — the NpmLockCollector workspace detection path
  (`link: true` handling in the v3 `packages` block).
- Peer dep on `react` from `react-dom` absent from the tree — Mend
  omits the peer edge when the peer is already a direct root dep.
- Peer dep from `use-sync-external-store` (workspace sub-dep) absent
  — Mend misses peers of packages scoped to workspace sub-projects.
- `engines.node` on root silently dropped — the `marker` field empty
  when the lockfile root entry has `"engines": {"node": ">=26.12.0"}`.
- Resolver falls through to NpmLsCollector / NpmFsCollector because
  the v3 `packages` block (no `dependencies` block) is not parsed,
  yielding a partial or completely flat tree.
- `loose-envify` duplicated once per parent instead of deduplicated
  to a single entry.

## Probe metadata

| Field | Value |
|---|---|
| Pattern | `bundled-pm-node26` |
| Category | `bundled_pm_change` |
| PM | npm |
| Node.js version | 26.12.0 (Current) |
| npm version | 10.9.2 |
| lockfileVersion | 3 |
| Release date | 2026-10-14 |
| Tagged by | @richardlau |
| Source | https://nodejs.org/en/blog/release/v26.12.0 |
| Status | untested |
