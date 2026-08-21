# Frontseat Architecture

**A polyglot build orchestrator keyed on the software catalog.**

Build targets are *catalog entities*, not directories. Plugins read the manifests a repository already has and attach tasks to those entities. Every hermetic action executes **remotely on a REAPI grid**, keyed by a digest that includes the exact tool versions **mise** resolved — so nothing is built twice, anywhere.

| | |
| --- | --- |
| **Targets** | Backstage catalog entities |
| **Lifecycle** | 22 phases, one ordered sequence |
| **Plugins** | ~30, each a separate gRPC process |
| **Execution** | Remote-only — no local fallback |
| **Tool versions** | User-owned; mise is the only source |
| **License** | Apache-2.0 |

---

## 1. The model: entities, pieces, plugins

Three nouns carry the whole system.

- **Entity** — a Backstage `Component`, `System` or `API` declared in `catalog-info.yaml`. The thing you own, review and release.
- **Piece** — a buildable unit discovered from a manifest: a Go module, a Dockerfile, a Helm chart, an OpenAPI spec.
- **Task** — a command a plugin attaches to a piece, bound to a lifecycle phase.

**Catalog plugins create entities.** Only a catalog plugin (backstage) turns catalog formats into entities. Language and tool plugins never create them — they enrich what already exists with pieces, tasks, rules and relationships. This isolation is what makes the catalog format replaceable: swapping it touches one plugin, not thirty.

**Pieces bind to entities by path.** A piece binds to the nearest enclosing entity root (*inferred*), or is named by the entity's `frontseat.dev/pieces` annotation as `<path>[#<type>][:<name>]` (*declared*). Binding is keyed on path, because a piece's name is plugin-derived — a package rename would break a name-keyed binding silently.

**Nothing dangles quietly.** A piece no entity owns is flagged by the `orphan-piece` rule; a declaration matching no piece is flagged by `unbound-piece-entry`. Both are warnings, not errors, which is what makes partial adoption viable: an unmodelled subtree degrades to a warning rather than blocking the build.

---

## 2. The plugin protocol

Each plugin is a standalone binary named `frontseat-plugin-<name>`, discovered from `$PATH` and managed by [HashiCorp go-plugin](https://github.com/hashicorp/go-plugin) over gRPC on Unix domain sockets. Plugins are installed by mise from the same versioned release stream as the CLI, and the daemon keeps them warm between commands.

The service is `frontseat.plugin.FrontseatPlugin`. A plugin implements only the RPCs it has something to say through:

| Area | RPCs |
| --- | --- |
| Lifecycle | `GetInfo`, `Configure` |
| Discovery | `CreateEntities`, `CreateRelationships`, `CreateTasks` |
| Pieces | `ListPieces`, `ProposePieces` |
| Conformance | `GetRules`, `CheckConformance` |
| Sync generators | `GetSyncGenerators`, `RunSyncGenerator` |
| Build definitions | `GetTasks`, `GetDependencyInstall` |
| Release | `GetStrategies`, `CalculateVersion`, `GetBranchingStrategies`, `ClassifyBranch` |

`GetInfo` returns the plugin's file patterns; the host matches files against them and passes only the relevant ones in. That routing is how a second catalog format plugs in — it declares different patterns and returns the same entity type.

Because plugins are separate processes behind a proto contract, third parties extend the system without touching or recompiling the core, and a plugin crash cannot take the daemon down.

---

## 3. One lifecycle, three groups

Phases are ordered, and running a phase runs everything before it.

| Group | Phases |
| --- | --- |
| **Setup** | `bootstrap` → `sync` → `conform` → `install` |
| **Build** (`frontseat build`) | *(`generate`)* → `format` → `lint` → `compile` → `test` → `package` → **`verify`** |
| **Ship** (`frontseat ship`) | `assemble` → `changelog` → `checksum` → `catalog` → `sign` → `deploy` → `upload` → `release` → `prepare` → `publish` → **`announce`** |

`verify` and `announce` are where the `build` and `ship` aliases stop.

`generate` is deliberately **outside** the hermetic build: codegen mutates the working tree, so it is a local verb (`frontseat generate`). The same principle governs setup — every setup phase is a check or an install, and the mutating variants (`sync --fix`, `conform --fix`) are explicit local commands, never lifecycle actions. Checks run before the expensive dependency install, so `conform` and `sync` need only tools and a non-conformant workspace fails fast.

A fixed phase frame is what makes thirty independently-written plugins compose. Cross-ecosystem ordering is automatic — nobody writes an edge saying the Helm chart packages after the Go binary compiles.

---

## 4. How tasks are inferred

Inference reads facts; it never guesses. A plugin walks the entities, asks whether the entity owns a piece of its type, parses that piece's manifest, and emits a task definition carrying everything the runner needs to key and sandbox it.

```
manifest found  →  piece bound to entity  →  task attached  →  typed action DAG
   go.mod            inferred or declared      phase, command,     phase-ordered,
   package.json      (before task discovery)   cwd, inputs,        dependency-resolved
   Chart.yaml                                  outputs, tools
```

**What a plugin emits.** For a Go piece: `Phase: compile`, a command derived from the real `main` packages found under the module root, `Inputs` as globs (`**/*.go`, `go.mod`, `go.sum`, plus any `//go:embed` resources declared explicitly), declared `Outputs`, and `Tools: ["go"]` — a name, never a version. Also carried: `Cwd`, the piece's own `Root` and `Manifest` (an entity is a logical anchor and may hold manifests in unrelated directories), cache policy, and network policy.

**Ownership is drawn on purpose.** For Node, the package-manager plugins (npm, yarn, pnpm) own installs and workspaces but contribute *no* build tasks — script names are convention, not contract. Framework plugins (vite, fumadocs, esbuild) own the tasks, keyed on their own config files.

---

## 5. Planning: the action DAG

Tasks become typed nodes — `InstallToolsAction`, `InstallDependenciesAction`, `TaskAction`, `ConformanceAction`, `SyncAction`, `ComplianceAction` — each workspace-, entity- or piece-scoped, each declaring whether it is cacheable.

Edges come from four rules:

1. Entity-scoped actions depend on workspace-scoped actions in the same phase.
2. Phase order applies **within a scope** — per entity, not globally. Entity A's compile does not wait on entity B's compile.
3. Entity phase actions depend on earlier populated workspace phases.
4. Dependency edges between entities carry the scope the manifest declared.

That last rule matters for parallelism. A dependency's scope — `compile`, `test`, `runtime`, `build` — determines *which* phases must wait: a test-only dependency (a JUnit-scoped jar, a devDependency, a `testImplementation`) is needed to run the consumer's tests, not to compile it, so it does not gate compilation. Every other scope blocks, including an unset one — under-ordering builds against a stale artifact, while over-ordering only costs parallelism.

The practical consequence: "strict lifecycle" is a per-entity ladder with entities running concurrently, not a global phase barrier.

---

## 6. mise: the toolchain contract

Frontseat never picks a tool version. Actions declare tool **names**; versions come exclusively from the user's mise config, which merges per directory — so resolution is directory-sensitive and a nested `mise.toml` genuinely changes the toolchain for that subtree. A declared tool with no configured version is a **hard failure at execution**, never a silent fall back to `latest`.

**Resolve.** The runner resolves declared names against the user's config for the action's working directory, caching per directory for the build's lifetime. The mise plugin's `require-tool-versions` rule surfaces missing pins at conform time, long before a build fails.

**Key.** Resolved versions are folded into the action's command environment as inert `FRONTSEAT_TOOL_<name>` markers, so the action digest is tool-sensitive. Where a version is not an identity — java 21.0.5 from Temurin and from Zulu are different artifacts under one name — a manager that can name the artifact exactly supplies an identity, and it wins. Tools without one keep their version string, which leaves existing cache entries valid.

**Provision.** `mise.lock` gives per-platform URLs and sha256 checksums. Tool archives reach the grid through the Remote Asset service and are extracted once per digest by a dedicated tool-extract action whose outputs — files, symlinks, executable bits — flow through the CAS. A failed extract degrades to in-sandbox extraction, never a failed build.

**Execute.** Locally the runner captures `mise env` per directory on top of a sanitized host environment and execs the command directly, with no `mise x` wrapper. Remote workers wrap with a *version-less* `mise x --`, because the version is already pinned into the action.

mise also delivers Frontseat itself: one backend plugin installs the CLI as `frontseat:cli` and every plugin as `frontseat:<name>`. That list must stay in step with `spec.plugins` in `frontseat.yaml` — a plugin declared there with no installed binary fails plugin loading outright.

---

## 7. REAPI: execution and caching

`frontseat build` and `frontseat ship` have no local execution mode. Every hermetic action is submitted through the Bazel Remote Execution API, using upstream `bazelbuild/remote-apis` and `remote-apis-sdks` — the real protos and the real client.

Action identity is computed **entirely on the client**: the Command and Action protos plus the Merkle input root. The digest computed locally is by construction the digest the grid caches under; there is no second keying implementation to drift from.

### What the action digest hashes

| Component | Detail |
| --- | --- |
| **command + args** | The exact argv, working directory normalized to empty (never `.`, which some grids reject) |
| **input root** | A Merkle tree over every declared input — sources, staged dependencies, pre-extracted tool trees. Undeclared files are simply not present. |
| **environment** | The task env, sorted, plus one `FRONTSEAT_TOOL_<name>` marker per resolved tool version or artifact identity |
| **platform** | OS family and architecture, so a darwin build and a linux build never collide; plus the worker container image when one is configured |
| **network policy** | `network: none` keys differently, so isolated results never collide with unrestricted ones |
| **declared outputs** | The paths the action promises to produce, collected into the CAS and replayed verbatim on a cache hit |

**Dependencies are fetched, not uploaded.** Go modules, npm tarballs, `.m2` jars and tool archives are provisioned through the Remote Asset `FetchBlob` API: the grid downloads them, verifies the checksum over every byte, and memoizes `(uri, checksum) → digest`. An unchanged lockfile resolves with zero network I/O and no client bytes.

**Zero-config grid by default.** With no `reapi:` endpoint configured and nothing listening on `localhost:50051`, the daemon starts an embedded no-Docker grid in-process — Capabilities, CAS, ByteStream, ActionCache, Execution and Remote Asset Fetch in a single process. A grid already listening wins; `frontseat grid serve` runs the same engine standalone.

**An explicit endpoint is a commitment.** If `frontseat.yaml` names a `reapi:` address, unreachable is a hard error. Frontseat never silently falls back to a different grid or cache — a cache you didn't ask for is a correctness problem, not a convenience.

### The scaling path

Because the action digest is computed client-side, the same action has the same digest on every backend. Moving up a tier is a configuration change, and cache entries stay compatible.

| | **1 · Embedded grid** | **2 · Persistent CI runners** | **3 · Managed elastic execution** |
| --- | --- | --- | --- |
| **For** | Onboarding, small and mid-size workspaces | Teams outgrowing one laptop | Large workspaces, maximum throughput |
| **Setup** | None — auto-starts in-process | Store on a persistent mount, no cache step | Point `reapi:` at the provider |
| **Parallelism** | One machine, `NumCPU` actions | One larger machine | Worker fleet |
| **Cache scope** | That machine | That runner pool | Shared across the organization |
| **Isolation** | Bare-runner, no sandbox | Bare-runner | Sandboxed |
| **`oci://` assets** | Not supported | Not supported | Supported (needed by apko, gib) |

`oci://` support is a *capability* boundary, not a scale one — a small repository that builds OCI images needs a real grid from day one. Cross-machine cache sharing arrives at tier 3: the read-through upstream cache that would give tier 2 a shared CAS and ActionCache is designed but not built, so tier 2 today means a warm store per runner pool.

---

## 8. Hermeticity and trust model

Correctness is enforced on the client, and the sandbox on shared grids is defense in depth rather than the primary mechanism.

- **Environment.** Actions run with exactly the declared environment. The host environment is sanitized with an explicit passthrough list before the tool manager's env is layered on, so ambient shell state cannot leak into a cache key or a build.
- **Inputs.** Only declared inputs are materialized. An undeclared file is not merely unhashed — it is absent from the input root.
- **Dependencies.** Every asset is checksum-verified over its bytes against the lockfile pin; a mismatch is a hard failure, never admitted to the CAS. Egress can be restricted by host allow-list; the checksum pin is the integrity guarantee either way.
- **Toolchains as inputs.** Tools are staged as CAS content, not installed into the worker. A linux grid gets a fully-static mise build so the sandbox needs nothing from the base image.
- **Ecosystem escape hatches are closed deliberately.** The go plugin pins `GOPROXY=off` inside the command text rather than in the task env, specifically so a user's `mise.toml` `[env]` cannot layer over it and reopen network egress.

Two honest limits. The embedded grid runs bare-runner semantics with no namespace sandbox, so an action reading an undeclared absolute path is not prevented locally — that risk is accepted for local dev and CI, matching the "bare" isolation tier other vendors offer. And `network: none` currently affects cache keying only; kernel-level enforcement is not implemented.

---

## 9. Integrating an external grid

The whole integration surface is one block in `frontseat.yaml`:

```yaml
spec:
  reapi:
    options:
      address: grid.example.com:443           # Execution, CAS, ActionCache, ByteStream
      instanceName: frontseat                 # REAPI instance name
      assetAddress: assets.example.com:443    # Remote Asset (Fetch) service
      logStreamAddress: logs.example.com:443  # optional; absence degrades to the main server
      image: registry/worker@sha256:...       # optional worker image, immutable reference
```

Notes for an executor vendor:

- **No client shim.** Frontseat speaks upstream REAPI; a compliant executor needs no Frontseat-specific work.
- **The Remote Asset service is load-bearing**, not optional polish. All dependency provisioning goes through `FetchBlob` — this is the API most implementations skip, and its absence is the single most likely integration gap.
- **`image` does double duty.** It is stamped into every action's digest as the `container-image` platform property, so bumping the image invalidates exactly the results produced inside the old one; on BuildBuddy the same property selects the executor image. One field handles routing and cache-keying together. Empty adds no property, which is why existing keys stay valid on the embedded grid.
- **Strictness matches production.** The embedded grid rejects `working_directory == "."` and unregistered platform property keys, mirroring the real grids — so a workspace that builds locally builds on yours.
- **LogStream is optional.** Its absence is a logged warning; streaming falls back to the main server.

---

## 10. Performance characteristics and limits

- **Client fan-out** is roughly 16 concurrent actions; the embedded grid executes up to `NumCPU` in parallel and queues beyond that.
- **Granularity is piece-level, not package-level.** A Go module compiles as one action with `**/*.go` as its inputs. Package-level tracking is precisely what forces generated build metadata, so refusing it is the no-rewrite promise, not an oversight.
- **Within-module incrementality is delegated, not lost.** Native toolchains already do it well, and their caches are piped through the action graph as declared outputs — the go plugin ships a `.gocache-frag.tar` fragment that consumers stage — so incremental compilation survives across hermetic actions.
- **The ceiling is a single very large module**: one action, one invalidation, and no amount of grid capacity helps. For a repository whose scale lives inside one module, Bazel remains the better tool.
- **Content-defined chunking is deferred deliberately.** It would cut cache traffic materially, but `SplitBlob`/`SpliceBlob` are vendor proto extensions rather than upstream `remote-apis`, and adopting them would mean either a local-only feature or standardizing on one backend.

---

## 11. Processes at runtime

```
CLI / MCP / IDE  ──Connect RPC (HTTP/2)──▶  Daemon  ──gRPC──▶  Plugin processes
                                              │                (one per plugin,
                                              │                 kept warm)
                                              ▼
                                          Pipeline  ──▶  Runner  ──REAPI──▶  Grid
                                       (typed DAG)      (digest,           (CAS, AC,
                                                         assets,            Execution,
                                                         ~16 in flight)     Fetch)
```

```sh
frontseat init            # detect ecosystems, install plugins, write frontseat.yaml + mise.toml
frontseat verify //...    # setup + build phases, hermetically, on the grid
frontseat conform         # rules only — needs tools, not dependencies
frontseat grid serve      # standalone embedded grid (no Docker)
frontseat announce //...  # the full ship pipeline through release and publish
```

---

## 12. Where each concern lives

Go workspace, 40+ modules, prefix `github.com/frontseat/frontseat/`.

| Module | Owns |
| --- | --- |
| `libs/frontseat-core` | Entity, piece and task models; the 22-phase lifecycle; every plugin interface |
| `libs/frontseat-plugin-host` | Plugin discovery from `$PATH`, gRPC transport, process lifecycle |
| `libs/frontseat-pipeline` | The action DAG: typed nodes, phase ordering, dependency planning |
| `libs/frontseat-runner` | Action digests, input trees, asset and tool staging, REAPI execution, the embedded grid |
| `libs/frontseat-tool` | Tool-manager abstraction: resolution, mise lockfile model, artifact identity |
| `libs/frontseat-daemon` / `-daemonclient` | Connect RPC server and the client that supervises it |
| `libs/frontseat-store` | Local content-addressed store — journal, CAS, spool |
| `apis/` | Protobuf service definitions; generated Go lives in `libs/frontseat-api` and `libs/frontseat-plugin-api` and is never hand-edited |
| `plugins/` | One module per ecosystem: go, cargo, dotnet, npm/yarn/pnpm, maven, gradle, tycho, uv, deno, vite, fumadocs, esbuild, buf, helm, apko, backstage, github, release strategies, publishers |
