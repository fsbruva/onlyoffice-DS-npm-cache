# onlyoffice-DS-npm-cache

Pre-built, offline npm cache tarballs for building
[OnlyOffice DocumentServer](https://github.com/ONLYOFFICE/DocumentServer) as a
FreeBSD port inside a [Poudriere](https://github.com/freebsd/poudriere) jail.

The jail has no outbound internet access by design. This cache supplies every npm
tarball the build needs so that all thirteen `npm install` invocations complete
fully offline, with minimal packument (registry metadata) fetches.

---

## Background

OnlyOffice DocumentServer's build system calls `npm install` across multiple
sub-projects. Poudriere jails are intentionally network-isolated, so all required
packages must be available before the build starts. A naive approach of
pre-downloading everything tends to include large amounts of registry metadata
(packuments) that bloat the cache and can still trigger network probes at install
time.

This repository solves that with a **minimal-packument** strategy, built on npm
12's `package-lock.json` (lockfile-version 3):

1. **Phase 1** — installs each sub-project into a temporary cache and generates a
   `package-lock.json` for it. A lockfile pins the full dependency tree to exact
   versions, eliminating all resolution queries at install time.
2. **Phase 2** — discards the temporary cache and rebuilds it from scratch using
   the generated lockfiles, plus three flags that suppress every remaining
   network probe: `--legacy-peer-deps`, `--no-audit`, `--no-fund`. What lands in
   the final cache is almost entirely tarballs, with minimal packuments.

npm 12 dropped support for `npm-shrinkwrap.json` for these trees — it is
silently ignored rather than honored — so this is a hard requirement, not a
preference. Any `npm-shrinkwrap.json`/`package-lock.json` shipped upstream is
removed before Phase 1 runs, so every lockfile in this repo is always generated
fresh by this workflow rather than trusted from the source tag.

npm 12 also introduced `allowScripts`, a per-project allow-list (stored in
`package.json`) of which dependencies are permitted to run install/postinstall
lifecycle scripts. This workflow treats that allow-list as something to audit
and commit deliberately, not something to leave implicit — see
[Lifecycle script policy](#lifecycle-script-policy) below.

The result is an offline cache that satisfies every `npm install` in the
DocumentServer build with `--offline`, on FreeBSD, for both `amd64` and
`aarch64`.

---

## Repository contents

| Path | Description |
|------|-------------|
| `package-locks/<source-relative-dir>/package-lock.json` | One per ordinary sub-project (11 total), always present |
| `package-locks/<source-relative-dir>/package.json` | Only present where this workflow patched, created, or approved lifecycle scripts for that directory — its presence is itself the signal that upstream's file should be overwritten |
| `yao-pkg-package-amd64.json` / `yao-pkg-package-aarch64.json` | `package.json` for the `@yao-pkg/pkg` install, per architecture |
| `yao-pkg-npm-package-lock-amd64.json` / `yao-pkg-npm-package-lock-aarch64.json` | `package-lock.json` for the `@yao-pkg/pkg` install, per architecture |
| `npm-allow-scripts.txt` | The lifecycle-script approval table: which directories may run install scripts for which packages, and why |
| `extra-patch-pkg-fetch_patches_node.v*.patch` | Composite FreeBSD patch for `@yao-pkg/pkg-fetch`, combining the (possibly leapfrogged) upstream Node patch with FreeBSD ports tree patches for the matching Node version |
| `.github/workflows/build-npm-cache.yml` | The workflow that produces all of the above |

`yao-pkg` is deliberately **not** nested inside `package-locks/`: it has two
independent, architecture-specific dependency trees rather than a
source-relative directory, so it's kept as four flat, arch-suffixed files at
the repository root. This lets a consumer (e.g. a port Makefile) treat
`package-locks/` with one generic recursive copy that never needs to know
`yao-pkg` exists, while `yao-pkg` gets its own small, explicit,
architecture-selected step.

The release tarball (attached to each GitHub Release) additionally contains:

| Path | Description |
|------|-------------|
| `_cacache/` | The npm cache directory (`content-v2` tarballs + `index-v5` metadata) |
| `build-info.txt` | Exact versions of Node, npm, `@yao-pkg/pkg`, `@yao-pkg/pkg-fetch`, and the pkg-fetch leapfrog decision used to build this cache |

---

## Covered sub-projects

The cache covers thirteen install targets across the DocumentServer source tree.
All installs use `--os freebsd` and `--cpu x64` (or `arm64` for the yao-pkg
aarch64 variant) so that platform-specific optional dependencies resolve
correctly even when building on a Linux runner.

**Production mode** (`NODE_ENV=production`):

| # | Directory | Notes |
|---|-----------|-------|
| 1 | `sdkjs/build` | |
| 2 | `web-apps/vendor/framework7-react` | `--include=dev` |
| 3 | `server/Metrics` | Patched `package.json` (adds `patch-package`); approved: `modern-syslog` |
| 4 | `server/FileConverter` | |
| 5 | `server/Common` | Approved: `win-ca` |
| 6 | `server` | Patched `package.json` |
| 7 | `server/DocService` | Patched `package.json` (multer bump, `patch-package` added, native build assets re-scoped); approved: `oracledb`, `sharp` |
| 8 | `document-server-integration/.../nodejs` | `--omit=dev` |

**Full mode** (no `NODE_ENV`):

| # | Directory | Notes |
|---|-----------|-------|
| 9 | `sdkjs` | Generated `package.json` (grunt + grunt-cli) — upstream ships none |
| 10 | `web-apps/build` | Patched `package.json` (grunt bump, `optipng-bin` + `patch-package` added, `postinstall: patch-package`); approved: `gifsicle`, `jpegtran-bin`, `optipng-bin` |
| 11 | `document-server-package/.../home/npm` | |
| 12a | `@yao-pkg/pkg` (amd64) | `--cpu x64`; seed `package.json`, no upstream equivalent; approved: `esbuild` |
| 12b | `@yao-pkg/pkg` (aarch64) | `--cpu arm64`; seed `package.json`, no upstream equivalent; approved: `esbuild` |

---

## Lifecycle script policy

npm 12's `approve-scripts` command flags any dependency in the tree that ships
an install/preinstall/postinstall lifecycle script. This workflow audits **all
13** target directories against that policy — not only the ones known in
advance to need it — so a package that newly gains an install script (via an
upstream dependency bump) fails the build loudly instead of silently shipping
an unreviewed script.

The audit runs in three passes:

1. **Before approval** — reports every directory's pending scripts, for
   visibility in the build log.
2. **Apply approvals** — runs `npm approve-scripts <packages>` for each
   directory listed in the approval table below.
3. **After approval** — re-audits all 13 directories. If anything is still
   pending, the build fails.

Current approval table (also written to `npm-allow-scripts.txt` in every
build, and included in the release tarball for auditability):

| Directory | Approved packages |
|-----------|--------------------|
| `server/Common` | `win-ca` |
| `server/DocService` | `oracledb`, `sharp` |
| `server/Metrics` | `modern-syslog` |
| `web-apps/build` | `gifsicle`, `jpegtran-bin`, `optipng-bin` |
| `@yao-pkg/pkg` (amd64 / aarch64) | `esbuild` |

To approve a newly-surfaced script, add a line to the `APPROVE_MAP` heredoc in
the *"Audit and approve npm lifecycle scripts"* step of the workflow.

Every `npm install` this workflow runs — Phase 1, Phase 2, the smoke test, and
the pkg-fetch patch generation probe installs — passes `--ignore-scripts`, so
no install script from any dependency ever executes on the GitHub-hosted
runner. That flag has nothing to do with the runner not being trusted with
*this particular* code; it's there because the runner's only job is producing
lockfiles and a tarball cache — running arbitrary third-party install scripts
during that process serves no purpose and is pure exposure. The `allowScripts`
approvals recorded here are consumed later, inside the actual FreeBSD jail
build, where those scripts are expected to run and their output (e.g.
`esbuild`'s or `sharp`'s prebuilt native binaries) is actually needed.

---

## Version dependency chain

Two independent version chains meet in this workflow, and the FreeBSD ports
tree — not `@yao-pkg/pkg-fetch` — is the authority on which exact Node version
the cache must target:

```
FreeBSD ports tree (www/node<major>/Makefile.version, NODEJS_PORTVERSION)
    └─▶ FREEBSD_NODE_VER   (the exact Node version this cache MUST support)

@yao-pkg/pkg version  (you supply this)
    └─▶ @yao-pkg/pkg-fetch version  (resolved from pkg's package.json)
            └─▶ PUBLISHED_NODE_VER  (resolved from pkg-fetch's patch filenames)
```

If `PUBLISHED_NODE_VER == FREEBSD_NODE_VER`, the released `@yao-pkg/pkg-fetch`
already has the patch this cache needs, and it's used directly.

If they differ — which happens whenever the FreeBSD Node port moves ahead of
the last `@yao-pkg/pkg-fetch` release — the workflow **leapfrogs**: it clones
`@yao-pkg/pkg-fetch` HEAD, verifies (via `patches/patches.json`, not just the
filename) that HEAD actually maps `FREEBSD_NODE_VER` to an exact patch, and
substitutes that patch and `patches.json` entry into the released package
before continuing. FreeBSD ports-tree patches for that Node version are then
appended, and `patch-package` produces one composite patch,
`extra-patch-pkg-fetch_patches_node.v<FREEBSD_NODE_VER>.patch`, that is verified
with a `patch --dry-run` against the real downloaded Node source tarball before
the build proceeds.

`npm_version` is supplied separately and must exactly match `www/npm-node<major>`
in the ports tree — the workflow verifies both `node --version` and
`npm --version` against their expected values before Phase 1 begins.

The leapfrog decision (`published` vs. `pkg-fetch-head`, plus the HEAD commit
SHA when leapfrogged) is recorded in `build-info.txt` and the GitHub Release
notes, so any cache can be traced back to exactly which pkg-fetch source
produced its patch.

---

## Triggering the workflow

Go to **Actions → Build npm Cache (npm12 / package-lock) → Run workflow** and
fill in the inputs:

| Input | Description | Default |
|-------|-------------|---------|
| `ds_version` | OnlyOffice DocumentServer `DISTVERSION` | `9.4.0` |
| `ds_build` | DS build number | `129` |
| `node_major` | Node.js major version — exact minor/patch resolved from the FreeBSD ports tree | `24` |
| `npm_version` | npm version used in the Poudriere jail — must exactly match `www/npm-node<major>` in the ports tree | `12.0.2` |
| `yao_pkg_ver` | `@yao-pkg/pkg` version — `@yao-pkg/pkg-fetch` and its published Node patch are resolved from this | `6.22.0` |
| `publish_release` | Create a tag + GitHub Release + push the metadata commit. Uncheck for a metadata-only dry run — the tarball is still produced and attached as a workflow artifact instead | `true` (checked) |

The workflow also runs automatically on `push: tags: v*`, in which case
`DS_TAG` is taken directly from the pushed tag and every other input falls
back to its default.

The workflow fails fast, with a clear error, if:
- the requested `node_major` has no `NODEJS_PORTVERSION` in the FreeBSD ports
  tree,
- `@yao-pkg/pkg-fetch` HEAD has no exact patch for that version when a
  leapfrog is required,
- the installed Node/npm don't exactly match what was requested,
- the generated pkg-fetch patch fails a dry-run against real Node source, or
- any lifecycle script goes unapproved after the audit.

### What the workflow does, step by step

1. **Bootstrap Node** — installs `node_major.x` (any patch) to get `npm` available.
2. **Checkout** this cache repository.
3. **Clone the FreeBSD ports tree** and read `NODEJS_PORTVERSION` from
   `www/node<major>/Makefile.version` — this becomes `FREEBSD_NODE_VER`, the
   authoritative target.
4. **Resolve the published pkg-fetch version** — installs `@yao-pkg/pkg` into a
   temp probe directory, reads which Node version its bundled `pkg-fetch`
   actually has a patch for, and compares it against `FREEBSD_NODE_VER`.
5. **Install the exact target Node** (`FREEBSD_NODE_VER`), replacing the
   bootstrap version.
6. **Install jail-matched npm** and verify both `node --version` and
   `npm --version` exactly.
7. **Clone sources** — shallow-clones all five ONLYOFFICE repositories at the
   DS release tag.
8. **Create patches** — writes the inline `patch -p0` diffs applied to
   `server`, `server/DocService`, `server/Metrics`, and `web-apps/build`.
9. **Generate the pkg-fetch FreeBSD patch** — uses the published patch
   directly, or leapfrogs to `@yao-pkg/pkg-fetch` HEAD as described above;
   appends the FreeBSD ports-tree Node patches; produces one composite patch
   via `patch-package`.
10. **Verify the patch** — downloads the real Node source tarball for
    `FREEBSD_NODE_VER` and dry-run applies the generated patch against it.
11. **Remove upstream lockfiles** — clears any `package-lock.json` or
    `npm-shrinkwrap.json` shipped upstream in every target directory, so Phase
    1 always generates fresh.
12. **Phase 1** — runs all 13 installs (`--ignore-scripts`,
    `--lockfile-version 3`) into a temporary cache, generating a
    `package-lock.json` for each.
13. **Audit and approve lifecycle scripts** — the three-pass process described
    above; writes `npm-allow-scripts.txt`.
14. **Capture package-lock files and package metadata** — builds the
    `package-locks/` tree (lockfile always, `package.json` only for
    patched/created/approved directories) plus the four flat yao-pkg root
    files.
15. **Phase 2** — discards the temporary cache, rebuilds using the captured
    lockfiles with `--legacy-peer-deps --no-audit --no-fund --ignore-scripts`
    to produce a tarballs-only cache.
16. **Smoke test** — verifies all 13 directories install successfully from the
    cache with `--offline`.
17. **Report statistics** — file counts, cache size, tarball/packument split,
    lockfile/package.json counts, resolved versions.
18. **Clean obsolete/stale artifacts** — removes old shrinkwrap-era files, any
    `extra-patch-pkg-fetch_patches_node.v*.patch` for a Node version other than
    the current one, and root-level `*-package.json` files left over from
    earlier revisions of this workflow, from both the working tree and the git
    index.
19. **Create the release tarball**, with a reproducible, sorted, stable-mtime
    layout for consistent `distinfo` hashes.
20. **Commit metadata** — `package-locks/`, `npm-allow-scripts.txt`, the
    current `extra-patch-*`, and the four yao-pkg root files; `git add -u`
    picks up anything the cleanup step deleted.
21. **Push, tag, and create the GitHub Release** — only when `publish_release`
    is checked (or on a tag push). Otherwise the tarball is uploaded as a
    workflow artifact instead, so dry runs still produce something to inspect.

### Release tarball layout

```
onlyoffice-DS-npm-cache-<tag>-<datetime>/
├── _cacache/                                   # npm cache (content-v2 + index-v5)
├── package-locks/                              # 11 sub-projects, mirroring source-relative paths
│   ├── sdkjs/
│   │   ├── build/package-lock.json
│   │   └── package-lock.json + package.json    # created from scratch
│   ├── server/
│   │   ├── package-lock.json + package.json    # patched
│   │   ├── Common/package-lock.json + package.json      # approved (win-ca)
│   │   ├── DocService/package-lock.json + package.json  # patched + approved
│   │   ├── FileConverter/package-lock.json
│   │   └── Metrics/package-lock.json + package.json     # patched + approved
│   ├── web-apps/
│   │   ├── build/package-lock.json + package.json       # patched + approved
│   │   └── vendor/framework7-react/package-lock.json
│   ├── document-server-integration/.../nodejs/package-lock.json
│   └── document-server-package/.../npm/package-lock.json
├── yao-pkg-package-amd64.json
├── yao-pkg-package-aarch64.json
├── yao-pkg-npm-package-lock-amd64.json
├── yao-pkg-npm-package-lock-aarch64.json
├── npm-allow-scripts.txt
├── extra-patch-pkg-fetch_patches_node.v*.patch
└── build-info.txt
```

`package.json` is present only where noted above — its presence in a given
directory is the signal that upstream's file should be overwritten; its
absence means leave upstream's file alone.

The tarball is created with `--sort=name`, `--mtime='UTC 2020-01-01'`,
`--owner=0`, and `--group=0` for reproducible `distinfo` SHA256 hashes, which
are also printed in the build log and included in the GitHub Release notes.

---

## Using the cache in a Poudriere build

The port's Makefile installs this cache in two passes:

1. A **generic recursive copy** over `package-locks/`, matching `package-lock.json`
   and `package.json` and installing each at the same source-relative path
   under `${WRKSRC}`, overwriting upstream where a `package.json` is present.
2. A **small, explicit, `${MACHINE_ARCH}`-selected step** for `yao-pkg`, which
   copies `yao-pkg-package-${MACHINE_ARCH}.json` and
   `yao-pkg-npm-package-lock-${MACHINE_ARCH}.json` into place before running
   `npm install` there. FreeBSD's `${MACHINE_ARCH}` (`amd64` / `aarch64`) maps
   directly onto the workflow's file suffixes — no translation needed.

Unlike every `npm install` in this cache-build workflow, the jail's own
`npm install` for `yao-pkg` should **not** pass `--ignore-scripts`: this is the
one place the `allowScripts`-approved lifecycle scripts (`esbuild`, `sharp`,
`gifsicle`, etc.) are actually meant to run, since their output is needed by
the packaged software.

Specify the `_PKGFETCH_NODE_VERSION` and `_NPM_CACHE_TAG` variables in the
FreeBSD `www/onlyoffice-documentserver` Makefile, and ensure the version of
`www/npm-node<major>` in `LIB_DEPENDS` matches the `npm_version` this cache was
built with exactly — `npm --version` inside the jail must equal the `npm_version`
recorded in `build-info.txt`, or the captured `package.json` `allowScripts`
policy (an npm 12 feature) will not apply as expected.

---

## License

BSD 2-Clause. See [LICENSE](LICENSE).

The npm package tarballs cached by this tool remain under their own respective
licenses. This repository contains only the workflow automation and metadata
(lockfiles, approved-script table, and patches) needed to reproduce the cache —
not the cached packages themselves.
