# Archy v2 Implementation Plan

> **For agentic workers:** Before implementing ANY task, read the [Philosophical Guardrails](../specs/2026-06-25-archy-design-v2.md#philosophical-guardrails) in the design spec. If your solution violates a guardrail, it is wrong regardless of how well it works.

> **For problems marked `BRAINSTORM REQUIRED`:** These are not one-shot problems. Before implementing, you MUST:
> 1. Research the problem space and identify at least 2-3 viable approaches
> 2. Present the approaches with trade-offs to the user for discussion
> 3. Get explicit user approval on the chosen approach before writing implementation code
>
> Do NOT skip this step because the problem seems solvable. These are marked because the wrong approach creates compounding problems downstream. Interactive discussion with the user is part of the process, not an obstacle to it.

**Goal:** Build a deterministic architecture governance CLI that integrates Codebase-Memory for structural extraction, adds in-code annotation enrichment, and produces a versioned, diffable `archy.json` artifact with CI enforcement.

**Design Spec:** [2026-06-25-archy-design-v2.md](../specs/2026-06-25-archy-design-v2.md)

**Tech Stack:** TypeScript 5.x, Node.js 20+, `codebase-memory-mcp` (bundled), `commander` (CLI), `js-yaml` (config/externals), `vitest` (testing), `tsup` (bundling)

---

## Guardrail Compliance Checklist

Before any PR is merged, the author must confirm:

- [ ] **G1**: No tree-sitter queries, AST walkers, or import resolvers (except annotation comment scanning)
- [ ] **G2**: Running `archy generate` twice on identical input produces byte-identical `archy.json` (excluding `generated_at`)
- [ ] **G3**: All downstream commands consume `archy.json`, not CM directly
- [ ] **G4**: No writes to user source files
- [ ] **G5**: CM is invoked internally, never exposed to user
- [ ] **G6**: New code has unit tests that don't depend on unrelated layers
- [ ] **G7**: `@archy` comments remain useful plain-text if Archy is removed
- [ ] **G9**: No LLM calls in the generation pipeline
- [ ] **G10**: Unknown/unmappable data produces warnings, not silent drops

---

## File Structure

```
archy/
  package.json
  tsconfig.json
  vitest.config.ts
  .gitignore
  src/
    index.ts                          # CLI entry point (commander)
    cli/
      commands/
        init.ts
        generate.ts
        render.ts
        validate.ts
        diff.ts
        check.ts
    types/
      graph.ts                        # ArchyNode, ArchyEdge, ArchyModule, ArchyGraph
      output.ts                       # ArchyOutput, ViewConfig, Warning
      annotations.ts                  # ArchyDirective, AnnotationEdge
      externals.ts                    # ExternalNode, ExternalInterface
      config.ts                       # ArchyConfig, ScopeConfig, EnforcementConfig
      cm.ts                           # CM-specific types (what we expect from CM's output)
    bridge/
      cm-runner.ts                    # Invoke CM binary, manage lifecycle
      cm-reader.ts                    # Read CM output (CLI or SQLite), extract raw data
      cm-mapper.ts                    # Map CM nodes/edges → Archy types
    annotations/
      scanner.ts                      # Scan source files for @archy comments
      parser.ts                       # Parse directive syntax
      associator.ts                   # Match annotations to CM nodes by proximity
    externals/
      loader.ts                       # Parse .archy/externals.yaml
    graph/
      merger.ts                       # Merge structural + semantic + external
      modules.ts                      # Module detection and assignment
      hash.ts                         # Deterministic content hashing
    output/
      serializer.ts                   # Deterministic JSON serialization
      mermaid.ts                      # Mermaid diagram rendering
    governance/
      staleness.ts                    # Dry-run comparison (archy check)
      differ.ts                       # Semantic graph diffing (archy diff)
      validator.ts                    # Annotation anchors + cross-module checks
    config/
      loader.ts                       # Load .archy.config.yaml
      defaults.ts                     # Default configuration values
  tests/
    fixtures/                         # Test fixture projects
    bridge/
    annotations/
    externals/
    graph/
    output/
    governance/
    config/
    cli/                              # Integration tests
```

---

## Layers & Problem Decomposition

Each layer is an independent problem domain. Layers communicate through the types defined in `src/types/`. The goal here is to identify the **problem surfaces** — what needs to be figured out — not to prescribe solutions.

### Layer 0: Scaffolding & Types

**What it is**: Project setup + the type contracts between all layers.

**Problem surface**: Minimal — this is foundational plumbing.

**Depends on**: Nothing.  
**Consumed by**: Everything.

**Sub-problems**:
- Package setup (TypeScript strict ESM, vitest, tsup)
- Core type definitions (graph, output, annotations, externals, config, CM)
- Type guards for runtime validation
- Build + test + lint commands

---

### Layer 1: CM Bridge

**What it is**: The adapter between Codebase-Memory's world and Archy's world.

**Problem surface**: This is where most integration risk lives. CM's schema is richer and differently-shaped than Archy's. The mapping decisions here determine what information Archy has to work with.

**Depends on**: Layer 0 (types), CM binary.  
**Consumed by**: Layer 4 (graph merger).

**Sub-problems**:

1. **CM Lifecycle Management** (`cm-runner.ts`) — `BRAINSTORM REQUIRED`
   - How to bundle CM binary with Archy (npm postinstall? platform-specific binaries? optional peer dependency?)
   - How to invoke CM indexing for a project
   - How to detect if CM is already indexed (skip re-index)
   - Version pinning: which CM release works with which Archy release
   - Error handling: CM crashes, timeouts, corrupted databases
   - **Why brainstorm**: The bundling strategy affects installation UX, CI compatibility, and cross-platform support. Wrong choice here means painful migration later.

2. **CM Data Reading** (`cm-reader.ts`) — `BRAINSTORM REQUIRED`
   - Interface choice: CLI mode (`codebase-memory-mcp cli <tool> '<json>'`) vs. direct SQLite read
   - If CLI: which tools to call? `search_graph`, `get_architecture`, `query_graph` (Cypher)?
   - If SQLite: decompress `.codebase-memory/graph.db.zst`, query tables directly
   - Pagination: CM may have thousands of nodes — how to handle?
   - Tradeoff: CLI is stable API, SQLite is faster but couples to CM internals
   - **Why brainstorm**: This is the most consequential API coupling decision. CLI gives stability but limits us to CM's tool semantics. SQLite gives full control but breaks on CM schema changes. A hybrid approach may be best, but needs prototyping.

3. **CM → Archy Mapping** (`cm-mapper.ts`) — `BRAINSTORM REQUIRED`
   - Node type mapping: CM `Function`/`Method` → Archy `function`, CM `Class` → `class`, etc.
   - Edge type mapping: which CM edges are architecturally relevant? (CALLS, IMPORTS, IMPLEMENTS, HTTP_CALLS — probably yes. SEMANTICALLY_RELATED, SIMILAR_TO — probably no.)
   - ID generation: CM node IDs → Archy's `file:symbol` format
   - Property preservation: what CM metadata to keep in `ArchyNode.metadata`?
   - Handling CM features Archy doesn't model (confidence scores, complexity, routes)
   - **Why brainstorm**: The mapping defines what Archy "sees" architecturally. Too aggressive filtering loses useful data. Too permissive creates noise. The right mapping should be informed by running CM on real projects and examining its output.

---

### Layer 2: Annotation Scanner

**What it is**: Reads `@archy` comments from source files and produces typed directives.

**Problem surface**: Moderate. The hard part is associating comments with code entities without re-parsing the AST.

**Depends on**: Layer 0 (types), CM node list (for entity matching).  
**Consumed by**: Layer 4 (graph merger).

**Sub-problems**:

1. **Comment Extraction** (`scanner.ts`)
   - Read source files, find lines matching `// @archy` or `# @archy`
   - Handle multi-line? (probably not v1 — single line keeps it simple)
   - Language-agnostic: `//`, `#`, `/* */`, `--` comment styles
   - Performance: scanning many files for comment patterns

2. **Directive Parsing** (`parser.ts`)
   - Parse the fixed grammar: `connects-to <target> [via <medium>]`, `layer <name>`, etc.
   - Error recovery: helpful messages for typos and malformed directives
   - Quoted string handling for `description "text here"`

3. **Entity Association** (`associator.ts`) — `BRAINSTORM REQUIRED`
   - Given an `@archy` comment on line N, which CM node does it annotate?
   - Strategy: find the nearest CM node whose line number is >= N (the "next declaration" after the comment)
   - Edge case: comment at end of file with no following entity
   - Edge case: multiple annotations before a single entity
   - Edge case: annotation on an entity CM didn't extract (constant, config object)
   - **Why brainstorm**: This is the seam between CM's structural world and Archy's semantic world. The association strategy determines annotation reliability. Line-proximity is the obvious approach but has failure modes (comments separated from entities by whitespace, decorators, or other comments). Alternative approaches include explicit `@archy for <entity>` syntax, or using CM's scope/containment info. Multiple approaches need evaluation against real codebases.

---

### Layer 3: External Loader

**What it is**: Parses `.archy/externals.yaml`.

**Problem surface**: Small — this is the simplest layer.

**Depends on**: Layer 0 (types).  
**Consumed by**: Layer 4 (graph merger).

**Sub-problems**:
- YAML parsing + validation
- `ext:` prefix enforcement on IDs
- Interface parsing (name, protocol, optional properties)
- Graceful handling of missing file (return empty array)

---

### Layer 4: Graph Merger

**What it is**: Assembles the unified `ArchyGraph` from all three data sources.

**Problem surface**: Moderate. The key challenge is resolving annotation targets to CM node IDs and assigning modules correctly.

**Depends on**: Layers 1, 2, 3.  
**Consumed by**: Layers 5, 6, 7.

**Sub-problems**:

1. **Target Resolution** — `BRAINSTORM REQUIRED`
   - Annotation targets like `workers/transform.ts:processMessage` need to resolve to actual CM node IDs
   - Partial matching? Fuzzy matching? Exact only?
   - What happens when a target doesn't resolve? (warning + dangling edge)
   - **Why brainstorm**: The target syntax is user-facing — it determines how natural annotations feel to write. Too strict (`src/workers/transform.ts:processMessage` exact path) is fragile to refactoring. Too loose (just `processMessage`) is ambiguous. The right answer may involve multiple resolution strategies with a priority order, or a different target syntax altogether. Needs real-world annotation writing to evaluate.

2. **Module Assignment** (`modules.ts`)
   - Default: directory-based (each top-level dir under scope = a module)
   - Config overrides: explicit module path mappings
   - Interaction with CM's own Package/Module concepts — use CM's grouping as a hint?

3. **Merge Strategy**
   - Structural nodes + annotation metadata enrichment (add layer, description, group to existing nodes)
   - Semantic edges added alongside structural edges (not replacing)
   - External nodes added as new nodes
   - Deduplication: same edge from structural + semantic? Keep both with different `source` tags, or merge?

4. **Deterministic Sorting**
   - Nodes sorted by `id`
   - Edges sorted by `from`, then `to`, then `type`
   - Modules sorted by `id`
   - Module `contains` arrays sorted

5. **Warning Collection**
   - Unresolved annotation targets
   - Annotations on unknown entities
   - CM parse errors passed through

---

### Layer 5: Serializer

**What it is**: Produces the canonical `archy.json` from an `ArchyGraph`.

**Problem surface**: Small but critical — determinism must be absolute.

**Depends on**: Layer 4 (graph merger).  
**Consumed by**: Layer 6 (governance), users (git commit).

**Sub-problems**:
- `JSON.stringify` with sorted keys (custom replacer or pre-sort objects)
- `generated_at` timestamp handling (included in output, excluded from hash comparison)
- Content hashing: SHA-256 of source files (for staleness detection)
- Config hashing: SHA-256 of `.archy.config.yaml` + `.archy/externals.yaml`
- Schema version field for forward compatibility

---

### Layer 6: Governance Engine

**What it is**: The enforcement and analysis tools that make `archy.json` useful for CI and development.

**Problem surface**: Moderate-high. Semantic diffing is the hardest sub-problem.

**Depends on**: Layer 5 (serializer), `archy.json` files.  
**Consumed by**: CLI commands, CI pipelines.

**Sub-problems**:

1. **Staleness Detection** (`staleness.ts`)
   - Run `generate --dry-run` internally, compare output to committed `archy.json`
   - Must ignore `generated_at` field in comparison
   - Exit code 0 (fresh) or 1 (stale) for CI
   - Performance: full regeneration on every check — acceptable for v1?

2. **Semantic Diffing** (`differ.ts`) — `BRAINSTORM REQUIRED`
   - NOT raw JSON diff — understand the graph structure
   - Added/removed nodes (with type and module context)
   - Added/removed edges (with description of what connection changed)
   - Module boundary changes
   - Human-readable terminal output (colors, grouping by module)
   - Machine-readable output (JSON) for CI integration
   - **Why brainstorm**: Graph diffing is a known hard problem. Naive approaches (compare sorted node/edge arrays) miss semantic changes (a renamed function looks like a delete + add). The right algorithm depends on what changes matter most to users — is a renamed function a "change" or a "delete + add"? Is a moved function (same code, different module) the most important change to surface? Needs user input on what makes a useful diff.

3. **Validation** (`validator.ts`)
   - Anchor checking: do `ext:` targets in semantic edges exist in `.archy/externals.yaml`?
   - Cross-module enforcement: structural edges crossing module boundaries without corresponding semantic edges
   - Annotation parse errors: surface them clearly
   - Configurable severity: warning vs. error per rule

---

### Layer 7: Renderer

**What it is**: Converts `ArchyGraph` into visual diagram formats.

**Problem surface**: Moderate. Mermaid syntax has quirks (ID sanitization, subgraph nesting).

**Depends on**: Layer 4 (graph merger) or `archy.json`.  
**Consumed by**: CLI, documentation, users.

**Sub-problems**:
- Multi-level views: system (modules only), module (single module internals), detail (single file)
- View filtering: project the full graph into level-appropriate subgraph
- Mermaid ID sanitization (no special characters, unique IDs)
- Edge styling: structural (solid) vs. semantic (dotted), edge labels
- Node styling: internal (solid) vs. external (dashed)
- Subgraph grouping by module
- Deterministic output (same graph → same Mermaid string)

---

## Dependency Graph

```
Layer 0 (Types + Scaffolding)
  ├── Layer 1 (CM Bridge)
  ├── Layer 2 (Annotation Scanner)
  ├── Layer 3 (External Loader)
  │
  └── Layer 4 (Graph Merger) ← depends on 1, 2, 3
        ├── Layer 5 (Serializer)
        ├── Layer 6 (Governance Engine) ← depends on 5
        └── Layer 7 (Renderer)
              │
              └── CLI Commands ← wires everything together
```

**Parallelizable**: Layers 1, 2, 3 have no dependencies on each other.  
**Sequential**: Layer 4 needs all three inputs. Layers 5, 6, 7 need Layer 4.  
**CLI** is the integration layer that orchestrates the pipeline.

---

## Implementation Sequence

### Phase 1: Foundation
- Layer 0: Types + scaffolding
- Layer 3: External loader (simplest, proves the pattern)

### Phase 2: Input Layers (parallelizable)
- Layer 1: CM bridge
- Layer 2: Annotation scanner

### Phase 3: Core Pipeline
- Layer 4: Graph merger
- Layer 5: Serializer

### Phase 4: Governance + Output
- Layer 6: Governance engine
- Layer 7: Renderer

### Phase 5: Integration
- CLI commands (wire everything together)
- Integration tests
- README / documentation

---

## Problems Requiring Interactive Brainstorming

These sub-problems are marked `BRAINSTORM REQUIRED` throughout the plan. Each needs 2-3 approaches presented to the user with trade-offs before implementation begins.

| Problem | Layer | Why It's Hard |
|---------|-------|---------------|
| CM bundling strategy | L1: CM Bridge | Affects install UX, CI, cross-platform. Hard to change later. |
| CM data interface (CLI vs SQLite) | L1: CM Bridge | Most consequential API coupling decision. |
| CM → Archy type mapping | L1: CM Bridge | Defines what Archy "sees." Needs real CM output to evaluate. |
| Annotation → entity association | L2: Annotations | Line-proximity has failure modes. Alternative syntaxes exist. |
| Annotation target resolution | L4: Graph Merger | User-facing syntax. Too strict = fragile, too loose = ambiguous. |
| Semantic graph diffing | L6: Governance | Known hard problem. "What matters" is subjective. |

**Rule**: An agent picking up a brainstorm-required problem must use `AskUserQuestion` to present approaches before writing code. One-shot implementations of these problems will be rejected.

---

## What This Plan Intentionally Does NOT Specify

1. **Low-level implementation details** — each layer's sub-problems need their own investigation when the task is picked up
2. **Exact CM interface choice** — CLI vs SQLite needs prototyping, not up-front decision
3. **Test fixture details** — fixtures should be designed when writing the layer, not prescribed here
4. **Error handling patterns** — should emerge from the TypeScript patterns used in Layer 0
5. **Performance optimization** — v1 prioritizes correctness over speed

This plan identifies WHERE the problems are and WHAT needs to be figured out. The HOW is delegated to the agent/developer working on each layer, constrained by the guardrails.
