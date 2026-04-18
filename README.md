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

This repository solves that with a **minimal-packument** strategy:

1. **Phase 1** — installs each sub-project into a temporary cache and generates
   an `npm-shrinkwrap.json` for it. Shrinkwraps lock the full dependency tree to
   exact versions, eliminating all resolution queries at install time.
2. **Phase 2** — discards the temporary cache and rebuilds it from scratch using
   the shrinkwraps, plus three flags that suppress every remaining network probe:
   `--legacy-peer-deps`, `--no-audit`, `--no-fund`. Only tarballs land in the
   final cache — no packuments.

The result is a ~118 MB cache that satisfies every `npm install` in the
DocumentServer build with `--offline`, on FreeBSD, for both `amd64` and
`aarch64`.

---

## Repository contents

| Path | Description |
|------|-------------|
| `shrinkwrap/` | 11 `npm-shrinkwrap.json` files, one per DS sub-project |
| `yao-pkg-npm-shrinkwrap-amd64.json` | Shrinkwrap for `@yao-pkg/pkg` (x64) |
| `yao-pkg-npm-shrinkwrap-aarch64.json` | Shrinkwrap for `@yao-pkg/pkg` (arm64) |
| `yao-pkg-package.json` | Seed `package.json` used for yao-pkg installs |
| `sdkjs-package.json` | Generated `package.json` for the sdkjs build-deps install |
| `extra-patch-pkg-fetch_patches_node.v*.cpp.patch` | Composite FreeBSD patch for `@yao-pkg/pkg-fetch`, combining the upstream patch with FreeBSD ports tree patches for the matching Node version |
| `.github/workflows/build-npm-cache.yml` | The workflow that produces all of the above |

The release tarball (attached to each GitHub Release) additionally contains:

| Path | Description |
|------|-------------|
| `_cacache/` | The npm cache directory (`content-v2` tarballs + `index-v5` metadata) |
| `build-info.txt` | Exact versions of Node, npm, `@yao-pkg/pkg`, and `@yao-pkg/pkg-fetch` used to build this cache |

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
| 3 | `server/Metrics` | |
| 4 | `server/FileConverter` | |
| 5 | `server/Common` | |
| 6 | `server` | Patched `package.json` |
| 7 | `server/DocService` | Patched `package.json` |
| 8 | `document-server-integration/.../nodejs` | `--omit=dev` |

**Full mode** (no `NODE_ENV`):

| # | Directory | Notes |
|---|-----------|-------|
| 9 | `sdkjs` | Generated `package.json` (grunt + grunt-cli) |
| 10 | `web-apps/build` | Patched `package.json` |
| 11 | `document-server-package/.../home/npm` | |
| 12a | `@yao-pkg/pkg` (amd64) | `--cpu x64` |
| 12b | `@yao-pkg/pkg` (aarch64) | `--cpu arm64` |

---

## Version dependency chain

The workflow inputs are designed so that no two inputs can conflict with each
other. The dependency chain flows in one direction only:

```
@yao-pkg/pkg version  (you supply this)
    └─▶ @yao-pkg/pkg-fetch version  (resolved from pkg's package.json)
            └─▶ Exact Node patch version  (resolved from pkg-fetch's patch filenames)
                    └─▶ npm version  (you supply this — must match www/npm-nodeXX in the jail)
```

You supply `node_major` (e.g. `20`) as a filter; the workflow resolves the exact
`20.x.y` by inspecting which Node versions `pkg-fetch` actually has patches for,
then installs that exact version. This guarantees the Node binary that `pkg-fetch`
will compile against is always the one it has a patch for.

---

## Triggering the workflow

Go to **Actions → Build npm Cache (Zero Packuments) → Run workflow** and fill in
the five inputs:

| Input | Description | Example |
|-------|-------------|---------|
| `ds_version` | OnlyOffice DocumentServer `DISTVERSION` | `9.3.1` |
| `ds_build` | DS build number | `8` |
| `node_major` | Node.js major version — minor/patch resolved automatically from `@yao-pkg/pkg-fetch` | `20` |
| `npm_version` | npm version used in the Poudriere jail — must exactly match `www/npm-nodeXX` in the ports tree | `11.12.1` |
| `yao_pkg_ver` | `@yao-pkg/pkg` version — `@yao-pkg/pkg-fetch` and the exact Node version are resolved from this | `6.14.2` |

The workflow will fail fast with a clear error if the requested `node_major` has
no corresponding patch in the resolved `@yao-pkg/pkg-fetch` version.

### What the workflow does, step by step

1. **Bootstrap Node** — installs `node_major.x` (any patch) to get `npm` available
2. **Resolve versions** — installs `@yao-pkg/pkg` into a temp directory; reads
   `@yao-pkg/pkg-fetch`'s patch filenames to determine the exact Node patch
   version; reads `pkg-fetch`'s version from `pkg`'s `package.json`; cleans up
3. **Install exact Node** — re-runs `setup-node` with the resolved exact version
4. **Install jail npm** — shadows the runner's npm with the version matching the
   Poudriere jail
5. **Clone sources** — shallow-clones all five ONLYOFFICE repositories at the
   DS release tag, plus the FreeBSD ports tree
6. **Apply patches** — creates patched `package.json` files for `server`,
   `server/DocService`, and `web-apps/build`
7. **Phase 1** — runs all 13 installs into a temporary cache, generating
   `npm-shrinkwrap.json` for each
8. **Phase 2** — discards the temporary cache, rebuilds using shrinkwraps with
   `--legacy-peer-deps --no-audit --no-fund` to produce a tarballs-only cache
9. **Generate pkg-fetch patch** — combines the upstream `@yao-pkg/pkg-fetch`
   FreeBSD node patch with patches from the FreeBSD ports tree for the exact Node
   version, producing `extra-patch-pkg-fetch_patches_node.vX.Y.Z.cpp.patch`
10. **Smoke test** — verifies all 13 directories install successfully from the
    cache with `--offline`
11. **Commit metadata** — commits shrinkwraps and supporting files to the repo
12. **Create release** — tags the commit and publishes a GitHub Release with the
    cache tarball attached

### Release tarball layout

```
onlyoffice-DS-npm-cache-<tag>-<datetime>/
├── _cacache/                          # npm cache (content-v2 + index-v5)
├── shrinkwrap/                        # 11 npm-shrinkwrap.json files
├── build-info.txt                     # Exact build versions
├── extra-patch-pkg-fetch_patches_node.v*.cpp.patch
├── sdkjs-package.json
├── yao-pkg-npm-shrinkwrap-aarch64.json
├── yao-pkg-npm-shrinkwrap-amd64.json
└── yao-pkg-package.json
```

The tarball is created with `--sort=name`, `--mtime='UTC 2020-01-01'`,
`--owner=0`, and `--group=0` for reproducible `distinfo` SHA256 hashes.

---

## Using the cache in a Poudriere build

Specify the _PKGFETCH_NODE_VERSION and NPM_CACHE_TAG variables
in the FreeBSD www/onlyoffice-documentserver Makefile.

Ensure the version of npm-node that is specified matches, as well.

---

## License

BSD 2-Clause. See [LICENSE](LICENSE).

The npm package tarballs cached by this tool remain under their own respective
licenses. This repository contains only the workflow automation and metadata
(shrinkwrap files) needed to reproduce the cache — not the cached packages
themselves.
