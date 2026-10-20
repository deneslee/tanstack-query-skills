# Repo setup: docs and skills

Goal: agents get the right guidance with as little always-loaded context as possible.
Tested 2026-10-20 with @tanstack/intent 0.5.6, router-core 1.171.38, router-plugin 1.168.47, devtools 0.15.0.

## 1. Files

Library skills live once, outside the projects. Project docs live in each project.

```
Documents/Projects/
  SKILLS/                        shared by all projects
    tanstack-query/              package "query-skills" (from query-skills.zip)
      package.json
      skills/…                   29 skills, 9 updated
  conference-app/                this project
    AGENTS.md                    routing table: docs + skills   (always loaded, ~3 KB)
    CLAUDE.md                    rules + @AGENTS.md import      (always loaded, ~1 KB)
    docs/agents/
      README.md  router.md  query.md  sharepoint-lib.md  testing.md
      tanstack-query-skills-audit.md
```

- Only library skills go in `SKILLS/`. The app docs (README, router, query, sharepoint-lib, testing) are about this app. Move them out of `SKILLS/tanstack-query` into `docs/agents/`.
- Merge `CLAUDE.md` into yours. Keep the `@AGENTS.md` line if your CLAUDE.md doesn't import it yet.
- `repo-setup.md` (this file) is for you, not for agents. Keep it out of `docs/agents/` or don't link it.

## 2. package.json

The shared folder is a local dependency. npm links it into `node_modules` (a junction on Windows).
Intent then treats it like any npm package. Edits in `SKILLS/` show up in every linked project right away.

```json
{
  "devDependencies": {
    "query-skills": "file:../SKILLS/tanstack-query"
  },
  "scripts": {
    "intent": "node ./node_modules/@tanstack/intent/dist/cli.mjs"
  },
  "intent": {
    "skills": [
      "@tanstack/router-core",
      "@tanstack/router-plugin",
      "@tanstack/match-sorter-utils",
      "query-skills"
    ],
    "exclude": [
      "@tanstack/router-core#router-core/ssr",
      "@tanstack/router-core#router-core/auth-and-guards",
      "query-skills#lifecycle/ssr-hydration-and-streaming",
      "query-skills#lifecycle/migrate-major-versions-and-codemods",
      "query-skills#framework/use-framework-adapter-reactivity",
      "query-skills#compositions/broadcast-realtime-and-multi-tab-synchronization",
      "query-skills#compositions/persist-offline-and-restore-caches",
      "query-skills#core/paginate-and-build-infinite-queries"
    ]
  }
}
```

- `skills` is an allowlist. Devtools packages are left out on purpose (already set up). Intent prints a notice that they're hidden. That's fine.
- `exclude` turns off single skills from allowed packages. Each project has its own excludes, so another project can use other skills from the same shared folder.
- The `file:` path is relative to the project. Another project elsewhere uses its own relative path (or an absolute one).
- Then `npm install`.
- `npm install` fails on a machine without the `SKILLS` folder. Fine for one machine. For more machines or CI: push `SKILLS/tanstack-query` to its own Git repo and use `"query-skills": "github:<you>/<repo>"`. npm can't install a subfolder of a repo, so one repo per package.

## 3. Check

```sh
npm run -s intent -- list
# 4 intent-enabled packages, 33 skills
# match-sorter-utils 1, router-core 8, router-plugin 1, query-skills 23

npm run -s intent -- load "query-skills#core/design-query-keys-and-options"
# prints the SKILL.md
```

Bin conflict check:

```sh
npm exec --no -- intent --version
```

If this fails with `ERR_PACKAGE_PATH_NOT_EXPORTED … './intent-library'`, the devtools command won.
`@tanstack/devtools` 0.15.0 ships `bin/intent.js`, which imports `@tanstack/intent/intent-library`. Intent removed that export in 0.1.0.
The `npm run intent` script calls Intent directly, so it works either way. That's why AGENTS.md uses it.

## 4. What not to run

| Option                          | Always-loaded cost (measured)                                         | Why not                                                             |
| ------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `intent install`                | 0.9 KB block                                                          | Tells agents to run `intent list` per task: ~15 KB output each time |
| `intent install --map`          | ~14 KB (33 skills, full descriptions)                                 | Our table does the same in ~3 KB                                    |
| `intent hooks install` (Claude) | ~7 KB catalog at every session start (19 skills), grows with the list | Its load command points at the conflicting `intent` bin             |

If one of them ran anyway: delete the `<!-- intent-skills:start -->…<!-- intent-skills:end -->` block from AGENTS.md, and the Intent hooks from `.claude/settings.json`.

## 5. Context budget

About 4 bytes per token.

| What                             | When                   | Size                           |
| -------------------------------- | ---------------------- | ------------------------------ |
| AGENTS.md + CLAUDE.md            | every session          | ~4 KB                          |
| one Query skill                  | when its row matches   | 3–7.5 KB (avg 4.4 KB)          |
| one router-core skill            | when its row matches   | 6–15 KB                        |
| `sharepoint-lib.md` / `query.md` | when the topic matches | 20 KB / 15 KB, read by section |
| other docs                       | when the topic matches | 5–10 KB                        |

## 6. Archiving: shrink the set over time

Don't move or delete skill files to archive them. Turn them off:

```sh
npm run -s intent -- exclude add "@tanstack/router-core#router-core/code-splitting"
```

Then delete its row from AGENTS.md. `exclude remove` brings it back.
This also works for npm skills, which you can't delete.

Suggested phases:

| Phase                               | Skill rows in AGENTS.md                                                           |
| ----------------------------------- | --------------------------------------------------------------------------------- |
| TanStack work (now)                 | all 20                                                                            |
| SharePoint lib, codegen, debug page | about 5: search-params, data-loading, design-keys, write-mutations, handle-status |
| Tests                               | add `test-query-code` back                                                        |
| Server-side mode                    | re-include `query-skills#core/paginate-and-build-infinite-queries`                |

Docs are not archived, they're updated.
Once real code exists, replace doc snippets with a pointer ("see `src/features/conferences/queries.ts`"). The code becomes the example. The doc keeps the why.

## 7. When TanStack Query ships official skills

1. Remove the `query-skills` devDependency (`npm uninstall query-skills`).
2. Find the package: `npm run -s intent -- list --show-hidden`. Add it to `intent.skills`.
3. Replace the `query-skills#…` ids in AGENTS.md. Skill names may change too.
4. Re-check the excludes and the audit.
5. Once no project links `SKILLS/tanstack-query` any more, delete it.

## 8. Router skills

- `@tanstack/router-core` ships its skills on npm: `node_modules/@tanstack/router-core/skills/router-core` is the same folder as `packages/router-core/skills/router-core` on GitHub, matching your installed version.
  Nothing to copy. Intent loads them from `node_modules`. They are the `@tanstack/router-core#…` rows in AGENTS.md.
- `@tanstack/react-router` is different. GitHub `main` has `packages/react-router/skills` (`react-router`, `compositions/router-query`, `lifecycle/migrate-from-react-router`), and its `package.json` already lists `skills` in `files`.
  But the published 1.170.45 contains no skills. They will most likely ship with a later release.
- Until then: use the updated `query-skills#compositions/compose-query-with-tanstack-router-and-start` for loaders + Query.
- When a release ships them: add `"@tanstack/react-router"` to `intent.skills`, exclude `#lifecycle/migrate-from-react-router`, and switch the "route loaders + Query" row to `@tanstack/react-router#compositions/router-query`.
- Don't copy them into `SKILLS/` meanwhile. It would be one more copy to keep in sync.

## 9. After upgrading TanStack packages

- router-core, router-plugin and match-sorter skills update with their packages. Nothing to do.
- `SKILLS/tanstack-query` does not. On Query minor upgrades, check the changelog for new deprecations.
