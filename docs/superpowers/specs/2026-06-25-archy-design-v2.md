# Archy v2: Architecture Governance over Codebase-Memory

**Date**: 2026-06-25
**Status**: Draft
**Supersedes**: [2026-06-23-archy-design.md](./2026-06-23-archy-design.md)
**Research**: [2026-06-25-archy-landscape-research.md](../research/2026-06-25-archy-landscape-research.md)

---

## Philosophical Guardrails

These invariants constrain ALL work on Archy — every task, every PR, every agent session. They are non-negotiable and exist to prevent well-intentioned sub-problem solutions from undermining the whole.

### G1: Archy does not parse source code

Structural extraction is Codebase-Memory's job. Archy reads CM's output. If you find yourself writing a tree-sitter query, an AST walker, or an import resolver — stop. You are solving the wrong problem. The only exception is the `@archy` annotation scanner, which reads source file comments (not structure).

### G2: Determinism is binary — it either is or it isn't

Same input files + same config must produce byte-identical `archy.json` (excluding `generated_at`). If a feature breaks this property, the feature is wrong. No "mostly deterministic" or "deterministic enough." This means: sorted collections, stable serialization, no timestamps in hashed content, no randomized IDs.

### G3: archy.json is the single source of truth

All downstream operations (diff, check, render, validate) consume `archy.json`. No command should bypass it by re-reading source or querying CM directly. The artifact IS the architecture. If information isn't in `archy.json`, it doesn't exist for governance purposes.

### G4: Annotations live in source code, not in Archy

`@archy` directives are comments in the user's source files. Archy reads them — it never writes to source files (except during `archy init` with explicit user consent). The only Archy-owned files are `archy.json`, `.archy.config.yaml`, and `.archy/externals.yaml`.

### G5: CM is an internal dependency, not a user-facing tool

Users interact with `archy` commands. They should never need to know CM exists, install it separately, or configure it. CM is bundled, versioned, and invoked internally. Its output format is an implementation detail that can change between Archy releases.

### G6: Every layer must be independently testable

The CM bridge, annotation parser, graph merger, serializer, diff engine, renderer, and CLI are separate units with clear interfaces. A test for the diff engine should not require CM to be installed — it operates on two `archy.json` files. A test for the annotation parser should not require the graph merger.

### G7: The annotation system must not create vendor lock-in

`@archy` comments are plain-text comments in source code. If a team stops using Archy, the comments are harmless documentation. No build-time dependency, no code generation, no macros. A developer reading the code without Archy installed should find the annotations useful, not confusing.

### G8: Future visual tooling must consume archy.json, not bypass it

The v2 viewer and v3 editor operate on the same `archy.json` artifact. They do not get special access to CM's graph or internal state. This ensures the CLI and visual tools always agree on what the architecture looks like.

### G9: No AI in the deterministic pipeline

The generation pipeline (CM index → import → annotate → merge → serialize) must be fully deterministic with zero LLM involvement. AI-assisted features (annotation suggestion, architecture Q&A) are separate commands that produce suggestions for human review — they never write to `archy.json` directly.

### G10: Prefer failing loudly over guessing silently

When CM's output doesn't map cleanly to Archy's model, emit a warning — don't silently drop data or invent mappings. When an annotation references a nonexistent target, fail validation — don't ignore it. Users should trust that `archy validate` passing means the architecture representation is complete and correct.

---

## Problem Statement

*Unchanged from v1* — AI agents and developers lack:
1. A reliable view of how architecture actually looks (vs. how they think it looks)
2. Tracking of architectural changes with the same rigor as code changes
3. Enforcement of architectural constraints in CI
4. Compact, reliable AI context that captures architectural intent

## What Changed from v1

The v1 spec proposed building a standalone tree-sitter extraction pipeline. Research revealed:

- **Graphify** (63K stars, YC S26) and **Codebase-Memory** (14.5K stars) already do structural extraction well
- Building a competitive extractor would be the highest-effort, lowest-differentiation work
- Archy's genuine differentiators are: **in-code annotations**, **CI enforcement**, **architecture diffing**, and the **versioned artifact**

v2 integrates Codebase-Memory as the structural extraction engine and focuses Archy's effort on the governance layer that no existing tool provides.

---

## Architecture Overview

```
User runs: archy generate
       |
       v
[Layer 1: CM Bridge]
  Invokes bundled codebase-memory-mcp
  → indexes project → reads graph output
  → maps CM nodes/edges → Archy's graph model
       |
       v
[Layer 2: Annotation Scanner]
  Reads source files for @archy comment directives
  → parses directive syntax
  → associates with nearest code entity (using CM's node IDs)
       |
       v
[Layer 3: External Loader]
  Reads .archy/externals.yaml
  → parses external boundary declarations
  → produces external nodes
       |
       v
[Layer 4: Graph Merger]
  Merges structural (CM) + semantic (annotations) + external layers
  → assigns modules
  → deduplicates
  → sorts deterministically
       |
       v
[Layer 5: Serializer]
  Produces archy.json
  → deterministic JSON (sorted keys, sorted arrays)
  → content hashing for staleness detection
       |
       v
[Layer 6: Governance Engine]
  archy check  → CI gate (is archy.json stale?)
  archy diff   → semantic architecture diff
  archy validate → annotation integrity + cross-module checks
       |
       v
[Layer 7: Renderer]
  archy render → Mermaid/D2 diagram output
  (future: archy serve → web viewer)
```

---

## Layer Descriptions

### Layer 1: CM Bridge

**Problem**: Codebase-Memory produces a knowledge graph with its own schema (node types, edge types, properties). Archy needs to consume this and map it to Archy's simpler, architecture-focused model.

**Boundary**: Input = CM's graph data (via CLI calls or direct SQLite read). Output = array of `ArchyNode` and `ArchyEdge` in Archy's type system.

**Key decisions to make**:
- Which CM node types map to which Archy node types? (CM has: Function, Method, Class, Interface, Enum, Type, Module, File, Folder, Package, Project, Route)
- Which CM edge types are architecturally relevant? (CM has: CALLS, IMPORTS, IMPLEMENTS, HTTP_CALLS, ASYNC_CALLS, RESOLVED_CALLS, DATA_FLOWS, EMITS, LISTENS_ON, SEMANTICALLY_RELATED, SIMILAR_TO)
- What's the right interface — CLI invocation, direct SQLite, or both?
- How to handle CM's richer properties (confidence scores, complexity metrics) — drop or preserve as metadata?
- Versioning: how to pin CM releases and handle breaking changes in CM's schema

**What NOT to do**: Parse source code. Resolve imports. Walk ASTs.

### Layer 2: Annotation Scanner

**Problem**: Extract `@archy` directives from source code comments and associate them with the code entities they annotate.

**Boundary**: Input = list of source file paths + CM's node list (for entity matching). Output = list of `AnnotationEdge` and `NodeMetadata` with parse errors.

**Key decisions to make**:
- How to associate a comment with its "nearest code entity" without re-parsing the AST (CM already has line numbers for nodes — use proximity matching?)
- Multi-line annotation syntax? Or single-line only?
- How to handle annotations on entities CM didn't extract (e.g., a constant, a config object)
- Error recovery: what happens when annotation syntax is close but not quite right?

**What NOT to do**: Walk ASTs. Use tree-sitter. Write to source files.

**Supported directives** (unchanged from v1):
- `@archy connects-to <target> [via <medium>]`
- `@archy triggers <target> [via <medium>]`
- `@archy reads-from <target>` / `@archy writes-to <target>`
- `@archy exposes <endpoint>`
- `@archy layer <name>`
- `@archy description "<text>"`
- `@archy group <name>`

### Layer 3: External Loader

**Problem**: Parse `.archy/externals.yaml` for entities outside the codebase (databases, APIs, devices, queues).

**Boundary**: Input = path to externals file. Output = array of `ExternalNode`.

**Key decisions**: Minimal — this is the simplest layer. Validate YAML, validate `ext:` prefix, return typed nodes.

### Layer 4: Graph Merger

**Problem**: Combine three data sources (CM structural, annotation semantic, external) into one unified `ArchyGraph`.

**Boundary**: Input = structural nodes/edges + annotation edges/metadata + external nodes. Output = `ArchyGraph` with module assignments.

**Key decisions to make**:
- Module detection: directory-based default vs. config overrides. How does this interact with CM's own Package/Module concepts?
- Edge deduplication: if CM says A calls B and an annotation says A connects-to B, are these the same edge or different?
- Node enrichment: annotations add metadata (layer, description, group) to structural nodes. How to handle conflicts?
- Warning generation: what conditions produce warnings? (unresolved annotation targets, orphan nodes, etc.)

### Layer 5: Serializer

**Problem**: Produce byte-identical JSON for the same graph input.

**Boundary**: Input = `ArchyGraph` + metadata. Output = JSON string.

**Key decisions to make**:
- `generated_at` handling (must be excluded from hash comparison)
- Key ordering strategy (alphabetical? schema-defined?)
- Float precision (if any numeric values are present)
- Pretty-print format (2-space indent, trailing newline)

### Layer 6: Governance Engine

**Problem**: Three types of architectural governance: staleness detection, semantic diffing, and constraint validation.

**Boundary**: Input = one or two `archy.json` files + enforcement config. Output = pass/fail + detailed report.

**Sub-problems**:
- **Staleness check** (`archy check`): regenerate in dry-run mode, compare against committed file. Exit 1 if different. This is the CI gate.
- **Semantic diff** (`archy diff`): not raw JSON diff — understand added/removed/modified nodes and edges, present in human-readable format.
- **Validation** (`archy validate`): annotation anchor checking (do `ext:` targets exist?), cross-module enforcement (are cross-boundary connections annotated?), annotation parse error reporting.

**Key decisions to make**:
- Diff algorithm: how to produce meaningful diffs of graph structures? (node matching by ID, edge matching by from+to+type)
- Enforcement levels: warning vs. error, configurable per rule
- Output format: terminal-friendly with colors? JSON for CI? Both?

### Layer 7: Renderer

**Problem**: Convert `ArchyGraph` into visual diagram syntax.

**Boundary**: Input = `ArchyGraph` (possibly filtered by view level). Output = Mermaid/D2 string.

**Key decisions to make**:
- Multi-level views: system (modules only), module (single module detail), detail (single file)
- Mermaid node ID sanitization (special characters)
- Styling: structural vs. semantic edges (solid vs. dotted), external nodes (dashed border)
- Layout direction (TB vs. LR)

---

## Type System (Core Types)

These types are the contracts between layers. They must be defined first and remain stable.

```typescript
// What kind of thing is this node?
type NodeType = 'function' | 'class' | 'interface' | 'type' | 'module'
              | 'variable' | 'device' | 'service' | 'route' | 'enum';

// What kind of relationship?
type EdgeType = 'calls' | 'imports' | 'contains' | 'inherits' | 'implements'
              | 'exports' | 'connects-to' | 'triggers' | 'reads-from'
              | 'writes-to' | 'exposes';

// Where did this come from?
type Source = 'structural' | 'semantic' | 'external';

interface ArchyNode {
  id: string;           // e.g. "src/api/routes.ts:ingestHandler" or "ext:stripe-api"
  type: NodeType;
  source: Source;
  file?: string;        // absent for external nodes
  line?: number;
  module?: string;
  signature?: string;
  description?: string;
  layer?: string;
  metadata?: Record<string, unknown>;  // preserves CM properties we don't model
}

interface ArchyEdge {
  from: string;         // node ID
  to: string;           // node ID
  type: EdgeType;
  source: Source;
  via?: string;         // communication medium (e.g. "SQS:ingest-queue")
  description?: string;
}

interface ArchyModule {
  id: string;
  path: string;
  contains: string[];   // node IDs
}

interface ArchyGraph {
  nodes: ArchyNode[];
  edges: ArchyEdge[];
  modules: ArchyModule[];
}

// The full archy.json schema
interface ArchyOutput {
  version: string;
  generated_at: string;
  source_hash: string;
  config_hash: string;
  cm_version: string;   // pinned CM version used
  languages: string[];
  graph: ArchyGraph;
  views: Record<string, ViewConfig>;
  warnings: Warning[];
  metrics: Record<string, unknown>;
}
```

---

## Configuration

### `.archy.config.yaml`

```yaml
# What to include/exclude (passed to CM for indexing scope)
scope:
  include:
    - "src/**"
    - "drivers/**"
  exclude:
    - "**/*.test.ts"
    - "node_modules/**"
    - "dist/**"

# Module boundary definitions
modules:
  api:
    path: "src/api"
  workers:
    path: "src/workers"

# Enforcement rules
enforcement:
  require_current: true                    # archy check fails if stale
  require_cross_module_annotations: true   # warn/error on unannotated cross-module edges
  strict_anchors: true                     # error on broken ext: references
  locked_modules: []                       # modules that cannot gain new external edges

# CM configuration passthrough
cm:
  # Any CM-specific config that needs tuning
  # (e.g., framework detection settings, indexing depth)

# Output
output:
  path: "archy.json"
  views:
    - level: 0
      name: "system"
    - level: 1
      name: "module-detail"
```

---

## CLI Interface

```bash
# Initialize Archy in a project
archy init

# Generate architecture (deterministic)
archy generate [--dry-run]

# Show what changed since last generation
archy diff

# Render to visual format
archy render --format mermaid [--level system|module|detail] [--module <name>]

# Validate annotations and constraints
archy validate

# CI gate: fail if archy.json is stale
archy check [--exit-code]
```

---

## Versioning Strategy

### v1: CLI + Governance (this spec)
- Bundled CM integration
- `@archy` annotation system
- Deterministic `archy.json` artifact
- `generate`, `check`, `diff`, `validate`, `render`, `init` commands
- Mermaid rendering
- CI enforcement

### v2: Read-Only Viewer
- `archy serve` — local web app rendering `archy.json`
- Interactive graph (D3/Cytoscape.js)
- Click-to-navigate: node → source file
- Module zoom: system → module → detail views
- Consumes `archy.json` only (guardrail G8)

### v3: Visual Editor
- Edit architecture visually in the web viewer
- Add new components, draw connections
- Emits `@archy` annotation stubs as patches
- LLM integration: suggest annotations, validate generated code against architecture
- `archy annotate --suggest` — AI-assisted annotation proposals

---

## Relationship to Existing Specs

- **v1 design** ([2026-06-23](./2026-06-23-archy-design.md)): Deprecated. Core concepts (annotation syntax, output format, CLI interface, enforcement rules) carry forward. Tree-sitter extraction pipeline replaced by CM integration.
- **v1 implementation plan** ([2026-06-24](../plans/2026-06-24-archy-implementation.md)): To be replaced by new plan aligned with this spec. Tasks 3-6 (parser engine, extractors, import resolution) eliminated. New tasks for CM bridge layer.
- **Research** ([2026-06-25](../research/2026-06-25-archy-landscape-research.md)): Justifies all architectural decisions in this spec.
