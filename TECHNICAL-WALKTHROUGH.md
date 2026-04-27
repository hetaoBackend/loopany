# loopany Technical Walkthrough

**Version:** 0.1.0  
**Last Updated:** 2026-04-27  
**Repository:** https://github.com/superdesigndev/loopany

---

## Executive Summary

**loopany** is a long-running agent brain system—a CLI + skills framework that enables self-iterating AI agents to maintain persistent memory of their actions, track outcomes, and automatically improve through a reflect-and-propose loop. It operates on the principle that agents need to record *what they did* and *how it played out*, not just *what they know*.

**Core Architecture:**
- **Storage:** Markdown + YAML frontmatter for artifacts (source of truth); append-only JSONL for references and audit logs; optional local SQLite index for hybrid search
- **No central database:** All data is plain files, making workspaces portable and git-friendly
- **Open registries:** Kind types and relation verbs are user-extensible
- **Runtime validation:** Frontmatter schemas and state machines are parsed dynamically from markdown definitions, not hardcoded in TypeScript

**Tech Stack:**
- **Language:** TypeScript (strict mode, ESM)
- **Runtime:** Bun (built-in SQLite, native TypeScript support)
- **Validation:** Zod for schema generation from kind definitions
- **Search:** Hybrid FTS5 (BM25) + semantic (HuggingFace embeddings) via `bun:sqlite`
- **Testing:** Bun's native test runner; 188 tests (184 pass, 4 timeout issues in search.e2e)

---

## Directory Structure

```
loopany/
├── src/
│   ├── cli.ts                      # Entry point: command dispatcher
│   ├── version.ts                  # VERSION = '0.1.0'
│   ├── core/
│   │   ├── engine.ts               # Bootstrap: composes all modules (Engine interface)
│   │   ├── artifact-store.ts       # Read/write/list artifacts with storage routing
│   │   ├── kind-registry.ts        # Loads kinds/*.md, builds Zod schemas + state machines
│   │   ├── references.ts           # Append-only graph at references.jsonl
│   │   ├── index.ts                # In-memory artifact + reference index (O(n) build)
│   │   ├── markdown.ts             # YAML frontmatter parsing + serialization
│   │   ├── config.ts               # Loads config.yaml; manages enabled domains
│   │   ├── search-store.ts         # Hybrid search: FTS5 + semantic via search.db
│   │   ├── embedder.ts             # Local embedding (Xenova/all-MiniLM-L6-v2, ~22MB)
│   │   ├── link-parser.ts          # Extracts [[id]] wiki links from markdown
│   │   └── audit.ts                # Append-only operation log at audit.jsonl
│   └── commands/
│       ├── artifact-create.ts      # `loopany artifact create --kind X ...`
│       ├── artifact-get.ts         # Read an artifact
│       ├── artifact-list.ts        # List with filters
│       ├── artifact-append.ts      # Append section to body
│       ├── artifact-status.ts      # Transition status (enforces state machine)
│       ├── artifact-set.ts         # Edit frontmatter field (not status)
│       ├── kind-list.ts            # Show registered kinds + schemas
│       ├── refs.ts                 # Query and add reference edges
│       ├── trace.ts                # Walk causal lineage (BFS with relation filtering)
│       ├── search.ts               # Hybrid search query
│       ├── reindex.ts              # Rebuild search.db from artifacts
│       ├── followups.ts            # Find artifacts with check_at due
│       ├── domain.ts               # List / enable / disable domains
│       ├── factory.ts              # Pixel-factory visualization server
│       ├── doctor.ts               # Workspace integrity checks
│       ├── init.ts                 # Scaffold workspace
│       ├── argv.ts                 # CLI flag parsing
│       └── body-input.ts           # Resolve body from --content or --content-file
├── kinds/                          # Core kind definitions (8 markdown files)
│   ├── mission.md
│   ├── task.md
│   ├── signal.md
│   ├── note.md
│   ├── person.md
│   ├── brief.md
│   ├── learning.md
│   └── skill-proposal.md
├── skills/                         # Agent behavior (markdown, not code)
│   ├── RESOLVER.md                 # Dispatcher: reads this first to route other skills
│   ├── conventions/
│   │   ├── relations.md            # 6 canonical relation verbs (led-to, addresses, etc.)
│   │   ├── core-artifacts.md       # Signal/task lifecycle + body conventions
│   │   └── new-concept.md          # Decision tree: note vs kind vs domain
│   ├── reflect.md                  # Self-improvement loop: analyze outcomes → propose changes
│   ├── proposal-review.md          # Accept/reject skill proposals
│   ├── daily-followups.md          # Check what's due today
│   ├── weekly-sweep.md             # Health check + slippage detection
│   └── monthly-review.md           # Strategic review + mission alignment
├── test/                           # 12 test files, ~2700 LOC
│   ├── artifact-store.test.ts
│   ├── kind-registry.test.ts
│   ├── references.test.ts
│   ├── index.test.ts
│   ├── search-store.test.ts
│   ├── link-parser.test.ts
│   ├── markdown.test.ts
│   ├── audit.test.ts
│   ├── config.test.ts
│   ├── cli.e2e.test.ts             # Full end-to-end CLI tests
│   ├── scenario.e2e.test.ts
│   ├── search.e2e.test.ts
│   └── helpers/
│       └── cli.ts                  # Test utilities for CLI invocation
├── injections/
│   └── resolver-memory.md          # AI agent memory injection
├── CLAUDE.md                       # Design philosophy + constraints
├── README.md                       # User-facing install + usage guide
├── ONBOARDING.md                   # 5-phase first-run setup script
├── INSTALL_FOR_AGENTS.md           # Agent-facing installation instructions
├── package.json                    # Dependencies: yaml, zod, @huggingface/transformers
└── tsconfig.json                   # TypeScript ES2022, strict mode

~/loopany/                          # Runtime workspace root ($LOOPANY_HOME or ~/loopany)
├── config.yaml                     # enabled_domains: [crm, ads, ...]
├── kinds/                          # Core kind defs (copied from src on init)
├── artifacts/
│   ├── {YYYY-MM}/                  # Date-bucketed: signals, tasks, briefs, learnings
│   │   ├── sig-*.md
│   │   ├── tsk-*.md
│   │   ├── brf-*.md
│   │   └── lrn-*.md
│   ├── notes/                      # Flat storage (slug-based)
│   ├── people/                     # Entity kinds
│   ├── missions/
│   └── skill-proposals/
├── domains/                        # Domain-specific configurations
│   └── {name}/
│       ├── manifest.yaml
│       ├── kinds/
│       ├── settings.yaml
│       └── view.json
├── references.jsonl                # Append-only graph edges
├── audit.jsonl                     # Append-only operation log
└── search.db                       # Derived: hybrid search index (deletable)
```

---

## Core Concepts (Three Finals)

### 1. Artifact
A **markdown file with YAML frontmatter** living in `artifacts/`. Every action and outcome the agent produces becomes an artifact. The `kind` field determines schema, state machine, and storage location.

**Example:** `artifacts/2026-04/tsk-20260427-103045.md`
```markdown
---
kind: task
title: "Refactor output layer"
status: done
priority: high
check_at: 2026-05-04
mentions: [prs-alice-chen, mis-product-launch]
---

## Hypothesis
Output buffering causes P99 latency spikes. Refactoring to direct-write should cut tail latency by 40%.

## Outcome
Implemented direct-write path + added metrics. P99 dropped from 850ms to 430ms in staging.
Blocked on prod deployment waiting for incident commander sign-off.
```

**Key Fields:**
- `id`: Auto-generated from kind + storage strategy (timestamp: `tsk-20260427-103045`; slug: `nte-project-phoenix`)
- `kind`: Determines frontmatter schema + state machine + storage location
- `path`: `artifacts/{YYYY-MM}/{id}.md` (date-bucketed) or `artifacts/{dirName}/{id}.md` (flat)
- `frontmatter`: Validated by `KindRegistry`; enforces required fields, enum values, types
- `body`: Append-only markdown; `## Outcome` required when status transitions to terminal states

### 2. References (Graph)
Append-only directed edges in `references.jsonl`, one per line. Plus **implicit edges** inferred from frontmatter `mentions: [...]` and body `[[id]]` wiki links.

**Edge shape:**
```json
{"ts":"2026-04-27T10:30:45.000Z","from":"tsk-20260427-103045","to":"sig-20260427-100000","relation":"addresses","actor":"cli"}
```

**Six canonical relation verbs** (from `skills/conventions/relations.md`):
| Verb | Direction | Example |
|------|-----------|---------|
| `led-to` | cause → effect | `sig-bug-report` → `tsk-fix-query` |
| `addresses` | action → observation | `tsk-refactor` → `sig-performance-issue` |
| `mentions` | source → entity | `tsk-meeting` → `prs-alice`, `mis-fundraising` |
| `supersedes` | new → old | `mis-fundraising-2027` → `mis-fundraising-2026` |
| `follows-up` | continuation → original | `tsk-followup-meeting` → `tsk-initial-meeting` |
| `cites` | summary → source | `brf-weekly` → `tsk-shipped-feature` |

**Implicit vs. Persisted:**
- **Implicit:** Live in artifact frontmatter/body; auto-reconstructed at index build; change when artifact changes
- **Persisted:** In `references.jsonl`; manually created via `loopany refs add`; immutable

### 3. Domain
A **meaningfully separable scope** the agent notices (sales pipeline, paid ads, research thread). Agent proposes; user accepts. Scoped config and kinds prevent global pollution.

**Structure:**
```
domains/{name}/
  manifest.yaml          # name, description, outcome windows
  kinds/*.md             # (e.g., contact, deal — domain-specific only)
  settings.yaml          # domain-specific config
  view.json              # optional UI view
```

---

## Kind System

**Kinds are defined in markdown, not code.** `loopany/kinds/*.md` contains:
- Frontmatter schema (YAML block with field types, validation, defaults)
- Status machine (initial state + allowed transitions)
- Storage strategy (date-bucketed vs. flat; timestamp vs. slug ID)
- ID prefix and directory name
- Indexed fields for fast filtering

### Core Kinds

| Kind | ID | Storage | Purpose | State Machine |
|------|----|---------|---------| ------------- |
| `mission` | `mis-` | flat/slug | Long-running pursuit (weeks/months) | active → [paused, satisfied, abandoned] |
| `task` | `tsk-` | date-bucketed/timestamp | Unit of work with outcome | todo → running → [in_review, done, failed, cancelled] |
| `signal` | `sig-` | date-bucketed/timestamp | Inbound observation (act or dismiss) | open → dismissed |
| `brief` | `brf-` | date-bucketed/timestamp | Summary (morning, weekly, post-meeting) | (no states) |
| `learning` | `lrn-` | date-bucketed/timestamp | Belief derived from outcomes | active → [superseded, archived] |
| `skill-proposal` | `spr-` | date-bucketed/timestamp | Proposed behavior change (agent proposes, user accepts) | pending → [accepted, rejected] |
| `person` | `prs-` | flat/slug | Human entity (dedup via name/aliases) | (no states) |
| `note` | `nte-` | flat/slug | Free-form markdown (fallback kind) | (no states) |

### Kind Definition File Structure

Example: `kinds/task.md`
```markdown
---
kind: task
idPrefix: tsk-
bodyMode: append
storage: date-bucketed
idStrategy: timestamp
indexedFields: [status, priority, check_at]
---

# task

A unit of work...

## Frontmatter

\`\`\`yaml
title:    { type: string, required: true }
status:   { type: enum, values: [todo, running, in_review, done, failed, cancelled] }
priority: { type: enum, values: [low, medium, high, critical], default: medium }
check_at: { type: date, required: false }
mentions: { type: 'string[]', required: false }
\`\`\`

## Status machine

\`\`\`yaml
initial: todo
transitions:
  todo:      [running, done, cancelled]
  running:   [in_review, done, failed, cancelled]
  in_review: [done, failed, cancelled]
\`\`\`

## Required sections

On `status: done` → body must contain `## Outcome`.
```

### Schema Parsing: `KindRegistry`

**File:** `src/core/kind-registry.ts`

```typescript
// Load all kinds from a directory
static async load(dir: string, opts?: { packDirs?: string[] }): Promise<KindRegistry>

// Parse a single kind definition
export function parseKindDefinition(raw: string): KindDefinition {
  // 1. Split frontmatter & body
  // 2. Extract metadata (kind, idPrefix, storage, idStrategy, etc.)
  // 3. Parse "## Frontmatter" YAML block → FieldSpec[]
  // 4. Build Zod schema dynamically from FieldSpec (handles enums, defaults, etc.)
  // 5. Parse "## Status machine" YAML block → StatusMachine (initial + transitions map)
  // 6. Return KindDefinition with all fields
}

// At runtime, KindRegistry provides:
registry.get(kind: string): KindDefinition | undefined    // by kind name
registry.getByPrefix(prefix: string): KindDefinition | undefined  // by ID prefix
registry.list(): KindDefinition[]
```

**Zod schema building** (lines 169–208):
```typescript
function buildZodSchema(spec: Record<string, FieldSpec>): z.ZodTypeAny {
  const shape: Record<string, z.ZodTypeAny> = {};
  for (const [name, field] of Object.entries(spec)) {
    let s: z.ZodTypeAny;
    switch (field.type) {
      case 'enum':
        s = z.enum(field.values as [string, ...string[]]);
        break;
      case 'string[]':
        s = z.array(z.string());
        break;
      // ... etc
    }
    if (field.default !== undefined) {
      s = s.default(field.default);
    } else if (!field.required) {
      s = s.optional();
    }
    shape[name] = s;
  }
  return z.object(shape).passthrough();
}
```

---

## Storage & File I/O

### Artifact Store: `src/core/artifact-store.ts`

```typescript
class ArtifactStore {
  async create(
    kind: string,
    frontmatter: Record<string, unknown>,
    body = '',
    opts: CreateOpts = {},
  ): Promise<Artifact>
  
  async get(id: string): Promise<Artifact | null>
  async appendSection(id: string, sectionName: string, content: string): Promise<void>
  async setField(id: string, field: string, rawValue: string): Promise<void>
  async setStatus(id: string, newStatus: string, reason?: string): Promise<void>
  async listAll(): Promise<Artifact[]>
}
```

**Key operations:**

1. **Create:** Auto-fill status from kind's initial state; validate frontmatter; allocate ID (timestamp or slug); write file
2. **Get:** Infer kind from ID prefix; read file; parse markdown
3. **Append Section:** Load artifact; use markdown section logic; rewrite file (append-only body pattern)
4. **Set Field:** Load artifact; coerce value by FieldSpec type; validate; rewrite
5. **Set Status:** Enforce state machine transitions; reject illegal moves; rewrite
6. **List All:** Walk all kinds' directories; collect all `.md` files; parse each

**ID Allocation:**

- **Timestamp strategy** (e.g., `task`): `{idPrefix}{YYYYMMDD-HHMMSS}` optionally with `-N` suffix on collision
- **Slug strategy** (e.g., `note`): `{idPrefix}{user-supplied-slug}`; error if duplicate

**Path Routing:**

```typescript
private pathFor(def: KindDefinition, id: string): string {
  if (def.storage === 'flat') {
    return join(root, 'artifacts', def.dirName, `${id}.md`);
  }
  // Date-bucketed: extract YYYYMM from id (e.g., tsk-YYYYMMDD-HHMMSS)
  const tsPart = id.slice(def.idPrefix.length);
  const yyyymm = `${tsPart.slice(0, 4)}-${tsPart.slice(4, 6)}`;
  return join(root, 'artifacts', yyyymm, `${id}.md`);
}
```

### Markdown Parsing: `src/core/markdown.ts`

```typescript
const FRONTMATTER_RE = /^---\n([\s\S]*?)\n---\n?([\s\S]*)$/;

export function parseMarkdown(raw: string): ParsedMarkdown {
  const match = raw.match(FRONTMATTER_RE);
  if (!match) return { frontmatter: {}, body: raw };
  const [, yamlBlock, body] = match;
  const frontmatter = parseYaml(yamlBlock) ?? {};
  return { frontmatter, body: body ?? '' };
}

export function appendSection(
  body: string,
  sectionName: string,
  content: string,
): string {
  // 1. Find "## SectionName" header
  // 2. If exists: insert content before next H2
  // 3. If not: append new section at end
}
```

---

## Indexing: ArtifactIndex

**File:** `src/core/index.ts`

Builds an in-memory index at CLI startup (O(n) on artifact count, acceptable <~1k artifacts).

```typescript
class ArtifactIndex {
  static async build(
    store: ArtifactStore,
    refs: ReferenceGraph,
    registry?: KindRegistry,
  ): Promise<ArtifactIndex>
  
  byId(id: string): ArtifactMeta | undefined
  byKind(kind: string): ArtifactMeta[]
  byStatus(status: string): ArtifactMeta[]
  byDomain(domain: string): ArtifactMeta[]
  byField(kind: string, field: string, value: unknown): ArtifactMeta[]
  domains(): string[]
  refsOut(id: string): Edge[]
  refsIn(id: string): Edge[]
  all(): ArtifactMeta[]
  followups(today: Date): ArtifactMeta[]  // check_at on or before today
}
```

**Build process:**

1. Load all artifacts via `store.listAll()`
2. Extract metadata (id, kind, path, frontmatter) for each
3. Build indices: byId, byKind, byStatus, byDomain, byField (for indexed_fields only)
4. Load persisted edges from `references.jsonl`
5. **Promote implicit edges** from frontmatter `mentions: [...]` and body `[[id]]` wiki links
   - Reconstructed every build (no persistence needed)
   - Marked with `implicit: true` if queried
6. Return bidirectional graph (forward + reverse maps)

**Indexed Fields:**

Only fields in a kind's `indexedFields` list are indexed. Array fields are indexed per-element.
```typescript
byField(kind: string, field: string, value: unknown): ArtifactMeta[] {
  return this.metasByField.get(kind)?.get(field)?.get(stringifyValue(value)) ?? [];
}
```

---

## Reference Graph

**File:** `src/core/references.ts`

```typescript
export interface Edge extends EdgeInput {
  ts: string;
  implicit?: boolean;  // true for mentions/wiki-links
}

export class ReferenceGraph {
  async append(edge: EdgeInput): Promise<Edge>
  async load(): Promise<LoadedGraph>  // returns {forward, reverse} maps
}
```

**Storage:**
- Path: `$LOOPANY_HOME/references.jsonl`
- One edge per line: `{"ts":"...","from":"...","to":"...","relation":"...","actor":"cli"}`
- Append-only: never rewritten
- Bidirectional indices built in memory (forward + reverse)

---

## Search: Hybrid FTS5 + Semantic

**File:** `src/core/search-store.ts`

Uses `bun:sqlite` to store:
1. **Chunked artifacts** (split on `##` / `###` headings)
2. **FTS5 virtual table** for BM25 keyword search
3. **Embeddings** as BLOB (when available)

```typescript
class SearchStore {
  async indexArtifact(a: ArtifactInput): Promise<void>
  async search(query: string, opts?: SearchOptions): Promise<SearchResult[]>
  needsIndex(id: string, mtime: number): boolean
  knownArtifactIds(): Set<string>
  removeArtifact(id: string): void
}
```

**Search strategy:**

```typescript
async search(query: string, opts?: SearchOptions): Promise<SearchResult[]> {
  const limit = opts?.limit ?? 10;
  
  // 1. Keyword search via FTS5 BM25
  const keywordResults = keywordSearch(query, opts, limit * 3);
  
  // 2. Semantic search (if embedder available)
  const qEmb = await embedder.embed(query);
  const semanticResults = rows.filter(r => cosineSim(qEmb, r.embedding) >= 0.3);
  
  // 3. Fuse via Reciprocal Rank Fusion (k=60)
  const fused = reciprocalRankFusion([keywordResults, semanticResults]);
  
  // 4. Group by artifact_id, keep best-scoring chunk per artifact
  const bestByArtifact = new Map();
  for (const [score, chunk] of fused.values()) {
    if (!bestByArtifact.has(chunk.artifact_id) || score > best.score) {
      bestByArtifact.set(chunk.artifact_id, {score, chunk});
    }
  }
  
  // 5. Return top N sorted by fused score
  return [...bestByArtifact.values()]
    .sort((a, b) => b.score - a.score)
    .slice(0, limit)
    .map(toSearchResult);
}
```

**Embedding Model:**

- Model: `Xenova/all-MiniLM-L6-v2` (384-dim, ~22MB quantized)
- Transport: HuggingFace Hub, cached to `~/.cache/huggingface`
- Runtime: `@huggingface/transformers` pipeline API
- Fallback: If model fails to load, degrades to FTS5-only; logs warning to stderr; never crashes

**Chunking:**

```typescript
export function chunkMarkdown(body: string): Array<{section: string | null, content: string}> {
  // Split on ^## or ^### headings
  // Title chunk (if present) is indexed separately
  // Empty sections are dropped
}
```

---

## Commands

### Entry Point: `src/cli.ts`

```typescript
async function main(): Promise<void> {
  const args = process.argv.slice(2);
  const head = args[0];
  
  // Recognize sub-command grouping
  const KIND_GROUPS = new Set(['kind', 'artifact', 'domain']);
  let cmdKey: string;
  if (KIND_GROUPS.has(head) && args[1]) {
    cmdKey = `${head} ${args[1]}`;
  } else if (head === 'refs' && args[1] === 'add') {
    cmdKey = 'refs add';
  } else {
    cmdKey = head;
  }
  
  // Dispatch and audit
  const meta = { op: cmdKey.replace(' ', '.') };
  try {
    await dispatch(cmdKey, rest, meta);
  } catch (e) {
    caught = e;
  }
  
  // Best-effort audit write
  await tryWriteAudit(meta, Date.now() - start, caught);
}

async function dispatch(cmd: string, rest: string[]): Promise<void> {
  switch (cmd) {
    case 'artifact create':
      const engine = await bootstrap();
      const result = await runArtifactCreate(engine, rest);
      console.log(JSON.stringify(result, null, 2));
      return;
    // ... etc
  }
}
```

### Bootstrap: `src/core/engine.ts`

```typescript
export async function bootstrap(): Promise<Engine> {
  const root = getWorkspaceRoot();  // $LOOPANY_HOME or ~/loopany
  if (!existsSync(join(root, 'kinds'))) {
    throw new WorkspaceNotFoundError(root);
  }
  
  const config = await Config.load(root);
  const packDirs = config.enabledDomains()
    .map(d => join(root, 'domains', d, 'kinds'));
  const registry = await KindRegistry.load(join(root, 'kinds'), { packDirs });
  const store = new ArtifactStore(root, registry);
  const refs = new ReferenceGraph(join(root, 'references.jsonl'));
  
  return {
    root,
    registry,
    store,
    refs,
    config,
    index: () => ArtifactIndex.build(store, refs, registry),
  };
}

export interface Engine {
  root: string;
  registry: KindRegistry;
  store: ArtifactStore;
  refs: ReferenceGraph;
  config: Config;
  index(): Promise<ArtifactIndex>;
}
```

### Sample Commands

#### `artifact create`
**File:** `src/commands/artifact-create.ts`

```typescript
export async function runArtifactCreate(engine: Engine, args: string[]): Promise<CreateResult> {
  const { flags } = parseArgs(args);
  
  const kind = flags.kind;
  if (!kind) throw new Error('Missing required flag: --kind');
  
  const def = engine.registry.get(kind);
  if (!def) throw new Error(`Unknown kind: ${kind}`);
  
  // Parse all --<field> <value> flags and coerce by FieldSpec type
  const frontmatter: Record<string, unknown> = {};
  for (const [flag, raw] of Object.entries(flags)) {
    if (RESERVED.has(flag)) continue;
    const fieldName = flagToField(flag);
    const spec = def.fieldSpecs[fieldName];
    if (!spec) throw error;
    frontmatter[fieldName] = coerceFlag(raw, spec);
  }
  
  if (flags.domain) frontmatter.domain = flags.domain;
  
  const opts: { slug?: string } = {};
  if (flags.slug) opts.slug = flags.slug;
  
  const body = await resolveBody(flags);  // --content or --content-file
  const a = await engine.store.create(kind, frontmatter, body, opts);
  return { id: a.id, kind: a.kind, path: a.path };
}
```

**Usage:**
```bash
loopany artifact create --kind task \
  --title "Refactor output layer" \
  --priority high \
  --check-at 2026-05-04 \
  --content "Initial hypothesis..."

loopany artifact create --kind note \
  --slug project-phoenix \
  --title "Project overview" \
  --tags feature,performance \
  --content-file ./notes.md
```

#### `refs add` & `refs` (query)
**File:** `src/commands/refs.ts`

```typescript
export async function runRefsAdd(engine: Engine, args: string[]): Promise<Edge> {
  const { flags } = parseArgs(args);
  if (!flags.from || !flags.to || !flags.relation) {
    throw new Error('refs add requires --from, --to, --relation');
  }
  return engine.refs.append({
    from: flags.from,
    to: flags.to,
    relation: flags.relation,
    actor: 'cli',
  });
}

export async function runRefsQuery(engine: Engine, args: string[]): Promise<Edge[]> {
  const { positional, flags } = parseArgs(args);
  const id = positional[0];
  const direction = flags.direction ?? 'out';
  const depth = flags.depth ? parseDepth(flags.depth) : 1;
  
  const idx = await engine.index();
  return traverse(idx, id, { direction, depth, relation: flags.relation });
}

// BFS traversal with visited tracking
function traverse(idx: ArtifactIndex, startId: string, opts: TraverseOptions): Edge[] {
  const visited = new Set<string>([startId]);
  const out: Edge[] = [];
  let frontier = [startId];
  
  for (let d = 0; d < opts.depth && frontier.length > 0; d++) {
    const next: string[] = [];
    for (const node of frontier) {
      const edges = opts.direction === 'out' ? idx.refsOut(node) : idx.refsIn(node);
      for (const e of edges) {
        if (opts.relation && e.relation !== opts.relation) continue;
        out.push(e);
        const other = e.from === node ? e.to : e.from;
        if (!visited.has(other)) {
          visited.add(other);
          next.push(other);
        }
      }
    }
    frontier = next;
  }
  return out;
}
```

**Usage:**
```bash
# Add edge
loopany refs add --from sig-bug-123 --to tsk-fix-456 --relation led-to

# Query outgoing edges (1 hop)
loopany refs tsk-123 --direction out --depth 1

# Find all sources (backward trace, 3 hops)
loopany refs tsk-123 --direction in --depth 3

# Filter by relation type
loopany refs mis-fundraising --relation led-to
```

#### `trace` (Causal Lineage)
**File:** `src/commands/trace.ts`

Walks the graph using a subset of relations (by default: `led-to`, `addresses`, `supersedes`, `follows-up`, `cites`; `mentions` excluded).

```typescript
export async function runTrace(engine: Engine, args: string[]): Promise<TraceResult> {
  const { positional, flags } = parseArgs(args);
  const id = positional[0];
  const direction = flags.direction ?? 'both';
  const relations = flags.relations 
    ? flags.relations.split(',')
    : DEFAULT_RELATIONS;
  
  const idx = await engine.index();
  const root = idx.byId(id);
  
  const nodesByDist = new Map([[id, 0]]);
  const edges: Edge[] = [];
  const relSet = new Set(relations);
  
  if (direction === 'forward' || direction === 'both') {
    walk(idx, id, +1, relSet, Infinity, nodesByDist, edges, new Set());
  }
  if (direction === 'backward' || direction === 'both') {
    walk(idx, id, -1, relSet, Infinity, nodesByDist, edges, new Set());
  }
  
  const nodes = [...nodesByDist.entries()]
    .map(([nodeId, dist]) => ({ ...idx.byId(nodeId), distance: dist }))
    .sort((a, b) => a.distance - b.distance);
  
  return { root: id, nodes, edges };
}
```

**Output:** Nodes sorted by distance (negative = causes, 0 = root, positive = effects).

#### `search`
**File:** `src/commands/search.ts`

```typescript
export async function runSearch(engine: Engine, args: string[]): Promise<SearchResult[]> {
  const { positional, flags } = parseArgs(args);
  const query = positional.join(' ');
  
  const dbPath = join(engine.root, 'search.db');
  if (!existsSync(dbPath)) {
    process.stderr.write('loopany: no search index found — run `loopany reindex` first.\n');
    return [];
  }
  
  const store = new SearchStore(dbPath, new TransformersEmbedder());
  try {
    return await store.search(query, {
      kind: flags.kind,
      domain: flags.domain,
      status: flags.status,
      limit: flags.limit ? parsePositiveInt(flags.limit) : 10,
    });
  } finally {
    store.close();
  }
}
```

**Usage:**
```bash
loopany search "authentication bug"
loopany search "billing" --kind note --limit 5
loopany search "onboarding" --domain ads --status running
```

---

## Audit Logging

**File:** `src/core/audit.ts`

Append-only operational log at `$LOOPANY_HOME/audit.jsonl`. Records every CLI invocation.

```typescript
export interface AuditEntry extends AuditEntryInput {
  ts: string;
}

export class AuditLog {
  async write(entry: AuditEntryInput): Promise<AuditEntry>
  async load(): Promise<AuditEntry[]>
}
```

**Entry shape:**
```json
{
  "ts": "2026-04-27T10:30:45.000Z",
  "op": "artifact.create",
  "actor": "cli",
  "duration_ms": 125,
  "kind": "task",
  "id": "tsk-20260427-103045",
  "error": null
}
```

---

## Wiki Link Parsing

**File:** `src/core/link-parser.ts`

Extracts `[[<id>]]` patterns from markdown bodies, skipping code blocks.

```typescript
export function extractLinks(body: string, validPrefixes: Set<string>): string[] {
  const stripped = stripCode(body);  // replace ``` and ` regions with spaces
  const out: string[] = [];
  const re = /\[\[([^\]\s]+?)\]\]/g;
  let m;
  while ((m = re.exec(stripped)) !== null) {
    const candidate = m[1];
    const prefix = candidate.slice(0, candidate.indexOf('-') + 1);
    if (validPrefixes.has(prefix)) out.push(candidate);
  }
  return out;
}
```

**Used by:** `ArtifactIndex.build()` to promote implicit edges.

---

## Testing

**Test suite:** 12 files, ~2700 LOC, 188 tests (184 pass).

```bash
bun test                    # Run all
bun test artifact-store     # Single file
```

**Test tiers:**

1. **Unit** (no FS mocking):
   - `kind-registry.test.ts` — parsing kind definitions, Zod schema building
   - `artifact-store.test.ts` — CRUD, ID allocation, field coercion
   - `markdown.test.ts` — frontmatter parsing, section append
   - `references.test.ts` — edge append/load
   - `index.test.ts` — index building, implicit edges
   - `search-store.test.ts` — chunking, FTS query, RRF fusion
   - `link-parser.test.ts` — wiki link extraction
   - `audit.test.ts` — audit log write/load
   - `config.test.ts` — config YAML load/save

2. **End-to-end:**
   - `cli.e2e.test.ts` — (1258 LOC) Full CLI workflows: init, artifact lifecycle, refs, status transitions, filtering
   - `scenario.e2e.test.ts` — Multi-step scenarios (e.g., create signal → task → complete with outcome)
   - `search.e2e.test.ts` — Reindex + search queries (4 tests timing out; known issue)

**Helper:** `test/helpers/cli.ts` — Utility to invoke CLI against temp workspace.

---

## Dependencies

```json
{
  "dependencies": {
    "@huggingface/transformers": "^4.2.0",
    "yaml": "^2.5.0",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@types/bun": "latest",
    "typescript": "^5.5.0"
  }
}
```

**Built-in (via Bun):**
- `bun:sqlite` — SQLite for search index
- TypeScript & ESM support

---

## Workspace Initialization

**File:** `src/commands/init.ts`

```bash
loopany init
```

Creates:
- `kinds/` with 8 core kind definitions
- `artifacts/` directory
- `config.yaml` (empty)
- `references.jsonl` (empty)
- `audit.jsonl` (empty)

Then runs onboarding (see `ONBOARDING.md`).

---

## Configuration: `config.yaml`

**Path:** `$LOOPANY_HOME/config.yaml`

```yaml
enabled_domains:
  - crm
  - ads
```

**Loaded by:** `Config.load(root)` at bootstrap.

---

## Skills (Markdown, Not Code)

Skills are **not executables**. They are markdown documents read by agents to understand operational procedures. Examples:

- `RESOLVER.md` — Dispatcher table; read this first to route to other skills
- `conventions/relations.md` — 6 canonical relation verbs + examples
- `conventions/core-artifacts.md` — Signal/task lifecycle + body conventions
- `reflect.md` — Self-improvement loop: analyze outcomes → propose behavior changes
- `proposal-review.md` — Accept/reject skill proposals + commit to git
- `daily-followups.md` — "What's due today?" check-in
- `weekly-sweep.md` — Health check + slippage detection
- `monthly-review.md` — Strategic review + mission alignment

---

## Design Principles (from CLAUDE.md)

1. **Thin harness, fat skills** — CLI is ~2000 LOC; skills carry domain judgment
2. **Latent vs. deterministic** — LLMs for judgment/synthesis; code for queries/validation
3. **Immutable for cited, mutable for "current understanding"** — Artifacts are append-only; config/skills are git-tracked
4. **All artifact operations are `artifact_*`** — Never `signal_create`, `task_create`, etc.
5. **Kind and relation are open registries** — Not closed enums
6. **Kind definitions live in markdown** — Runtime is data-driven
7. **Artifacts never edit-in-place** — Append, flip status, or supersede
8. **Agent proposes, human accepts** — For skills, kinds, migrations
9. **Markdown + frontmatter is the format** — Structured pieces in fenced blocks inside body, not JSON envelopes
10. **Skill-to-skill links use `[[other-skill]]` in body** — Reference graph is artifact-only

---

## Execution Flow Example

### Scenario: Create task, complete it, reflect

```bash
# 1. Create a task
loopany artifact create \
  --kind task \
  --title "Refactor output buffer" \
  --priority high \
  --content "Initial hypothesis: tail latency spikes are due to buffer contention."

# Output: { id: "tsk-20260427-103045", kind: "task", path: "..." }
```

**Behind the scenes:**
1. `bootstrap()` loads engine (registry, store, refs, config)
2. `runArtifactCreate()` parses flags
3. `KindRegistry.get('task')` returns definition with Zod schema
4. Frontmatter validated via Zod
5. `ArtifactStore.allocateId()` generates timestamp ID
6. Artifact written to `artifacts/2026-04/tsk-20260427-103045.md`
7. `AuditLog.write()` records `{op: 'artifact.create', kind: 'task', id: '...', duration_ms: X}`

```bash
# 2. Read the task
loopany artifact get tsk-20260427-103045

# Output: JSON with id, kind, path; or markdown body if --format body
```

```bash
# 3. Complete the task
loopany artifact status tsk-20260427-103045 done

# Behind the scenes:
# - Load artifact
# - Check state machine: todo → running → in_review → done (legal)
# - Update frontmatter.status = 'done'
# - Rewrite file
# - Audit log entry
```

```bash
# 4. Append outcome section
loopany artifact append tsk-20260427-103045 --section "Outcome" \
  --content "Implemented direct-write path. P99 latency dropped from 850ms to 430ms in staging."

# Behind the scenes:
# - Load artifact
# - Find "## Outcome" section; if missing, create it at end
# - Append content under that section
# - Rewrite file
```

```bash
# 5. Later: reflect on outcomes
# Agent runs reflect skill (reads RESOLVER.md → reflect.md)
# Which reads recent `task` outcomes via:
loopany artifact list --kind task --status done

# Generates `learning` artifact:
loopany artifact create --kind learning \
  --title "Direct-write reduces tail latency by ~50%" \
  --evidence [tsk-20260427-103045] \
  --domain performance \
  --check-at 2026-05-25 \
  --content "Three recent refactors (output buffer, cache thrashing, request parsing) all show ~50% tail-latency improvement when using direct-write patterns. Worth systematizing."

# Then generates `skill-proposal`:
loopany artifact create --kind skill-proposal \
  --title "Default to direct-write in hot paths" \
  --target-skill "skills/conventions/core-artifacts.md" \
  --change-type "add" \
  --evidence [lrn-...] \
  --mention [lrn-...] \
  --content "## Proposed change\nAdd subsection under 'Refactor patterns' recommending direct-write for P99-sensitive paths."

# 6. User reviews and accepts:
loopany artifact status spr-... accepted

# Behind the scenes:
# - skill-proposal artifact updates to status: accepted
# - Diff applied to target skill file (manual process, not automated in v0.1)
# - Git commit created
# - `## Outcome` appended to spr artifact
# - check_at scheduled for follow-up: "did this help?"
```

---

## Command Reference

| Command | Signature | Purpose |
|---------|-----------|---------|
| `artifact create` | `--kind K [--slug S] [--<field> V]... [--content C \| --content-file F]` | Create artifact |
| `artifact get` | `<id> [--format json\|body]` | Read artifact |
| `artifact list` | `[--kind K] [--status S] [--domain D]` | List artifacts |
| `artifact append` | `<id> --section SEC --content C` | Append body section |
| `artifact status` | `<id> <new-status> [--reason R]` | Transition status |
| `artifact set` | `<id> --<field> <value>` | Edit frontmatter field (not status) |
| `refs add` | `--from A --to B --relation R` | Add edge |
| `refs` | `<id> [--direction in\|out\|both] [--relation R] [--depth N]` | Query edges (BFS) |
| `trace` | `<id> [--direction forward\|backward\|both] [--relations CSV] [--max-depth N]` | Causal lineage |
| `search` | `<query> [--kind K] [--domain D] [--status S] [--limit N]` | Hybrid search |
| `reindex` | `[--force] [--no-embed]` | Rebuild search index |
| `followups` | `[--due today\|overdue]` | Find check_at due items |
| `kind list` | | Show registered kinds |
| `domain enable \| disable \| list` | `[name]` | Manage domains |
| `doctor` | `[--format json]` | Health check |
| `factory` | `[--port N] [--no-open]` | Pixel-factory UI server |
| `init` | | Scaffold workspace |
| `--version` | | Print version |

---

## Key Files & Line Counts

| File | Lines | Role |
|------|-------|------|
| `src/cli.ts` | 336 | Entry point & dispatch |
| `src/core/artifact-store.ts` | 273 | Artifact I/O |
| `src/core/kind-registry.ts` | 209 | Kind parsing & validation |
| `src/core/index.ts` | 225 | In-memory indexing |
| `src/core/references.ts` | 78 | Graph edges |
| `src/core/search-store.ts` | 442 | Hybrid search |
| `src/core/markdown.ts` | 62 | YAML + markdown parsing |
| `src/core/config.ts` | 57 | Workspace config |
| `src/core/engine.ts` | 57 | Bootstrap |
| `src/core/audit.ts` | 49 | Operation logging |
| `src/core/embedder.ts` | 95 | Embedding model |
| `src/core/link-parser.ts` | 62 | Wiki link extraction |
| `src/commands/*.ts` | ~1000 | Command implementations |
| **Total Core** | **~2700** | |
| `test/*.ts` | ~2700 | Tests |

---

## Constraints & Anti-Patterns (from CLAUDE.md)

**Must do:**
1. All artifact operations are `artifact_*` (kind is a parameter)
2. Kinds are open registries (config, not code changes)
3. Kind definitions live in markdown files
4. Artifacts are append-only for cited kinds; only status flip, field update, or supersede
5. Storage (artifacts/) and organization (domains/) are separate
6. Agent proposes, human accepts (skills, kinds, migrations)
7. Markdown + frontmatter is the format
8. Skill-to-skill links are `[[other-skill]]` in body, not in references.jsonl

**Avoid:**
- Editing artifacts in place (use append, status flip, or supersede)
- Inventing new relation verbs (use the 6 canonical ones)
- Writing opposite-direction edges (query `--direction in` instead)
- Hardcoding domain concepts in code (move to config/kinds)
- Mixing code-driven validation with skill-driven judgment (keep separate)

---

## Future Roadmap (Not Yet Implemented)

From `CLAUDE.md` § "Future":
- Domain packs (pre-built CRM / ads / content / metrics domains)
- MCP server (skills accessible via Claude Code's tool system)
- UI/dashboard (read-only visualization)
- Workflow engine (scheduled tasks, cron skills)
- Multi-user workspace (with audit trails)

Current status: **v0.1.0 — usable for single-user dogfooding.**

---

## Glossary

| Term | Definition |
|------|-----------|
| **Artifact** | Markdown file with YAML frontmatter + body. Unit of agent memory. |
| **Kind** | Type/schema of an artifact (task, signal, note, mission, etc.). Defined in markdown. |
| **Frontmatter** | YAML metadata at top of artifact (title, status, priority, etc.). |
| **Body** | Markdown content after frontmatter. Append-only for most kinds. |
| **Status machine** | State diagram (initial state + legal transitions) for a kind. |
| **Reference edge** | Directed graph edge in references.jsonl (from → to → relation → ts). |
| **Implicit edge** | Edge inferred from artifact frontmatter `mentions[]` or body `[[id]]`. Not persisted. |
| **Persisted edge** | Edge explicitly written to references.jsonl via `loopany refs add`. |
| **Domain** | Organizational scope (CRM, ads, research). Scoped config + kinds. |
| **Skill** | Markdown document describing operational procedures (not executable code). |
| **Relation verb** | Directed edge semantic (led-to, addresses, mentions, supersedes, follows-up, cites). |
| **Trace** | Walk causal lineage via subset of relation verbs (default: excludes `mentions`). |
| **Artifact index** | In-memory map built at CLI startup (byId, byKind, byStatus, byDomain, byField). |
| **Search index** | SQLite DB (search.db) with chunked artifacts, FTS5, embeddings. Derived; deletable. |

---

End of Technical Walkthrough
