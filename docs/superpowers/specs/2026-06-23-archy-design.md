# Archy: Deterministic Architecture Generation

**Date**: 2026-06-23
**Status**: Deprecated — superseded by [2026-06-25-archy-design-v2.md](./2026-06-25-archy-design-v2.md)
**Author**: SHV-118 Research Session

## Problem Statement

AI agents are powerful at reading code but terrible at understanding the big picture. They consume enormous token budgets scanning files to build mental models that are inconsistent across sessions. Meanwhile, developers lack a reliable way to:

1. See how their architecture actually looks (vs. how they think it looks)
2. Track architectural changes over time with the same rigor as code changes
3. Enforce architectural constraints in CI
4. Provide AI agents with efficient, reliable project context

Existing tools either use AI (non-deterministic, unreliable for versioning), focus only on dependency graphs (miss semantic architecture), or are query-based runtime systems (not versionable artifacts).

## Solution

Archy is a CLI tool that **deterministically generates a versioned architecture representation** from source code. It produces a canonical JSON file (`archy.json`) that:

- Lives in git alongside the code
- Diffs meaningfully when architecture changes
- Renders to Mermaid/D2 for human visualization
- Serves as compact, reliable AI context
- Enables architectural testing and enforcement in CI

## Core Principles

1. **Deterministic**: Same code always produces the same output. No AI in the generation pipeline.
2. **Versioned**: `archy.json` is a committed artifact. `git diff archy.json` shows architecture changes.
3. **Multi-language**: Tree-sitter parsing supports TS/JS and Python in MVP, with C/C++ and 60+ other languages as incremental additions.
4. **Layered**: Structural (auto-extracted) + semantic (annotated) + external (boundary declarations) in one unified graph.
5. **Compact**: Optimized for token efficiency when used as AI context.

## Competitive Landscape

| Tool | Approach | Why Archy Is Different |
|------|----------|----------------------|
| Swark | LLM-based diagram generation | Non-deterministic. Can't version or diff reliably. |
| Codebase-Memory | Tree-sitter knowledge graph via MCP | Runtime query engine, not a versioned artifact. No diffing. |
| CodeGraph | Tree-sitter + SQLite + MCP | Same as above — queryable, not versionable. |
| Drift | Static analysis erosion detection | Detects violations but doesn't generate architecture. Python-only. |
| dependency-cruiser/Madge | Import graph analysis | JS/TS only. Dependency graph != architecture. |

**Archy's unique position**: The only tool producing a deterministic, versioned, diffable, multi-language architecture representation with semantic enrichment, optimized for both AI consumption and human visualization.

## Architecture Model

### The Unified Graph

Archy produces a single graph with three layers, all merged into one data structure:

#### Layer 1: Structural (Auto-Extracted)

Tree-sitter parses source files and extracts:

- **Nodes**: Functions, classes, interfaces, modules, type definitions
- **Edges**:
  - `calls` — function A calls function B
  - `imports` — module A imports from module B
  - `contains` — class A contains method B
  - `inherits` / `implements` — class A extends/implements B
  - `exports` — module A exports symbol B

Each node includes: file path, line number, symbol name, type, and module assignment.

#### Layer 2: Semantic (In-Code Annotations)

Developers or AI assistants enrich the graph with semantic connections that static analysis cannot infer. These live **in the source code as structured comments**, using the `@archy` prefix:

```typescript
// @archy connects-to workers/transform.ts:processMessage via SQS:ingest-queue
// @archy description "Enqueued messages trigger the transform worker"
export function enqueue(message: Message) { ... }
```

```python
# @archy connects-to workers/transform.py:process_message via SQS:ingest-queue
# @archy description "Enqueued messages trigger the transform worker"
def enqueue(message: dict):
```

```c
// @archy connects-to ext:UUT via UART:TX
// @archy description "Sends test commands to Unit Under Test"
void uart_send(uint8_t* data, size_t len) {
```

**Why in-code**: Annotations that live in the code move with it — if a function is renamed, deleted, or refactored, the annotation follows. This eliminates the anchor drift problem entirely. Annotations are also visible in code review, discoverable while reading code, and diffable in git alongside the changes they describe. This follows the same pattern as Spring Boot annotations or JSDoc tags, but works across all languages via comments.

**Comment syntax**: `@archy <directive> <args>`. Tree-sitter already parses comments, so extracting `@archy` directives is a natural extension of the structural parsing pass — no second file to read.

**Supported directives**:
- `@archy connects-to <target> [via <medium>]` — declares a connection to another entity
- `@archy triggers <target> [via <medium>]` — declares an async trigger relationship
- `@archy reads-from <target>` / `@archy writes-to <target>` — data flow declarations
- `@archy exposes <endpoint>` — declares an exposed interface (REST route, gRPC service, etc.)
- `@archy layer <name>` — assigns this entity to an architectural layer
- `@archy description "<text>"` — human-readable description of architectural role
- `@archy group <name>` — overrides automatic module assignment

#### Layer 3: External (Boundary Declarations)

Entities outside the codebase have no source file to attach annotations to, so they are declared in `.archy/externals.yaml`:

```yaml
externals:
  - id: "ext:UUT"
    type: "device"
    description: "Unit Under Test - external hardware DUT"
    interfaces:
      - name: "UART"
        protocol: "UART"
        baud: 115200

  - id: "ext:stripe-api"
    type: "service"
    description: "Stripe payment processing API"
    interfaces:
      - name: "REST"
        protocol: "HTTPS"
```

External nodes use an `ext:` prefix. They render with distinct visual styling (dashed borders) to differentiate from in-codebase entities. The `.archy/externals.yaml` file is the only separate annotation file — it exists because external entities have no code to attach to.

### Module Detection

Modules are determined by:
1. **Directory structure** as the default (each top-level directory = a module)
2. **Configuration overrides** in `.archy.config.yaml` for custom module boundaries
3. **Community detection** (Louvain algorithm) as an optional auto-grouping mode based on call-graph density

### Multi-Level Views

The graph supports filtered views at different zoom levels:

- **Level 0 (System)**: Modules + external entities + cross-boundary edges only
- **Level 1 (Module)**: All nodes within a specific module + their internal/external edges
- **Level 2 (Detail)**: Full call graph for a specific file or class

Each level is a filter on the same graph, not a separate data structure.

## Output Format

### `archy.json` (Canonical)

```json
{
  "version": "1.0.0",
  "generated_at": "2026-06-23T15:30:00Z",
  "source_hash": "sha256:...",   // Hash of all in-scope source files (for staleness detection)
  "config_hash": "sha256:...",   // Hash of .archy.config.yaml + .archy/semantic.yaml
  "languages": ["typescript", "python"],
  "graph": {
    "nodes": [
      {
        "id": "src/api/routes/ingest.ts:ingestHandler",
        "type": "function",
        "source": "structural",
        "file": "src/api/routes/ingest.ts",
        "line": 42,
        "module": "api",
        "signature": "(payload: IngestPayload) => Promise<void>"
      },
      {
        "id": "ext:UUT",
        "type": "device",
        "source": "external",
        "description": "Unit Under Test"
      }
    ],
    "edges": [
      {
        "from": "src/api/routes/ingest.ts:ingestHandler",
        "to": "src/services/queue.ts:enqueue",
        "type": "calls",
        "source": "structural"
      },
      {
        "from": "src/services/queue.ts:enqueue",
        "to": "src/workers/transform.ts:processMessage",
        "type": "triggers",
        "source": "semantic",
        "via": "SQS:ingest-queue"
      }
    ],
    "modules": [
      {
        "id": "api",
        "path": "src/api",
        "contains": ["src/api/routes/ingest.ts:ingestHandler"]
      }
    ]
  },
  "views": {
    "system": {
      "level": 0,
      "description": "Module-level architecture overview"
    }
  },
  "warnings": [],
  "metrics": {}
}
```

### Rendered Outputs

Generated on demand from `archy.json`:

- **Mermaid**: For docs, Notion, Linear, GitHub markdown
- **D2**: For richer diagrams with styling
- **SVG/PNG**: For embedding anywhere
- **JSON (filtered)**: Level-specific JSON views for AI context injection

## CLI Interface

```bash
# Initialize Archy in a project
archy init

# Generate architecture (deterministic)
archy generate

# Show what changed since last generation
archy diff

# Render to visual format
archy render --format mermaid                    # Full graph
archy render --format mermaid --level system     # Module-level only
archy render --format mermaid --module drivers   # Specific module detail

# Validate annotations against current code
archy validate

# CI gate: fail if archy.json is stale
archy check

# Serve local viewer (future)
archy serve
```

### `archy init`

Interactive bootstrapping:
1. Detect languages in the project
2. Detect project structure (monorepo vs. single)
3. Generate `.archy.config.yaml` with sensible defaults
4. Run first `archy generate`
5. Display the system-level Mermaid diagram

## Configuration

### `.archy.config.yaml`

```yaml
# What to include/exclude
scope:
  include:
    - "src/**"
    - "drivers/**"
    - "lib/**"
  exclude:
    - "**/*.test.ts"
    - "**/*.spec.ts"
    - "node_modules/**"
    - "generated/**"
    - "dist/**"

# Module boundary definitions (override auto-detection)
modules:
  api:
    path: "src/api"
  workers:
    path: "src/workers"
  drivers:
    path: "drivers"

# Language-specific configuration
languages:
  typescript:
    tsconfig: "./tsconfig.json"  # For path resolution
  python:
    root: "src"                  # For import resolution
  c:
    include_paths:
      - "include/"
      - "drivers/include/"

# Enforcement rules
enforcement:
  require_current: true
  require_cross_module_annotations: true
  strict_anchors: true
  locked_modules:
    - "drivers/timing"
    - "core/state-machine"

# Output
output:
  path: "archy.json"
  views:
    - level: 0
      name: "system"
    - level: 1
      name: "module-detail"
```

## Import/Path Resolution Strategy

This is the hardest technical challenge for v1. Strategy per language:

### TypeScript/JavaScript
- Read `tsconfig.json` `paths` and `baseUrl` for alias resolution
- Read `package.json` `exports` for package boundary detection
- Resolve relative imports directly
- Fall back to node module resolution for `node_modules`

### Python
- Detect project root via `pyproject.toml`, `setup.py`, or `setup.cfg`
- Resolve relative imports within packages
- Use directory structure for absolute imports
- Note: Dynamic imports (`importlib`) cannot be resolved statically — these become candidates for semantic annotation

### C/C++
- Use configured `include_paths` for header resolution
- Parse `#include` directives (both `""` and `<>` forms)
- CMakeLists.txt parsing for include directories (future plugin)
- Note: Macro-heavy code and conditional compilation (`#ifdef`) degrade accuracy — documented limitation

### V1 Approach
Start with basic resolution (relative imports, configured paths). Flag unresolved imports as warnings. Users can add semantic annotations to bridge gaps. Improve resolution incrementally based on real-world usage.

## Drift Detection & Enforcement

### Three Types of Drift

1. **Structural drift**: Code changed but `archy.json` wasn't regenerated
   - Detection: `archy generate --dry-run` produces different output than committed file
   - CI: Fail the build

2. **Annotation drift**: Largely eliminated by in-code annotations (they move with the code). Only applies to `.archy/externals.yaml` references — `archy validate` checks that `ext:` targets referenced from code annotations actually exist in the externals file.
   - CI: Warning or error (configurable via `strict_anchors`)

3. **Undocumented connections**: New cross-module calls without semantic annotation
   - Detection: New structural edges crossing module boundaries that lack a corresponding semantic edge
   - CI: Warning or error (configurable via `require_cross_module_annotations`)

### Architecture Testing

Because `archy.json` is structured JSON, teams can write architecture tests:

```typescript
import arch from './archy.json';

test('timing module only depends on HAL', () => {
  const timingOutbound = arch.graph.edges.filter(e =>
    e.from.startsWith('drivers/timing') &&
    !e.to.startsWith('drivers/timing')
  );
  expect(timingOutbound.every(e => e.to.startsWith('hal/'))).toBe(true);
});

test('no circular module dependencies', () => {
  // Build module-level adjacency and check for cycles
  const moduleDeps = buildModuleGraph(arch);
  expect(hasCycle(moduleDeps)).toBe(false);
});
```

## Future Extensions (Post-MVP)

### Metrics Attachment
The `metrics` field on nodes and edges is reserved for observability data:
```json
{
  "id": "api/routes/ingest.ts:ingestHandler",
  "metrics": {
    "p99_latency_ms": 245,
    "error_rate": 0.02,
    "calls_per_minute": 1200
  }
}
```

A future `archy metrics attach` command merges runtime data onto the architecture graph, enabling diagrams colored by performance — instantly see architectural bottlenecks.

### Plugin System
Framework-specific extractors that auto-generate semantic edges:
- **Terraform plugin**: Read `.tf` files → extract resource topology
- **Express/Fastify plugin**: Extract route definitions and middleware chains
- **Embedded C plugin**: Detect ISR registrations, timer configs, pin mappings

Plugins output the same annotation format as manual semantic edges. They emerge from patterns observed in real annotation usage.

### PR Architecture Diff
A GitHub Action that comments on PRs with a rendered before/after architecture diff. Shows added/removed/modified nodes and edges. Key adoption driver for teams.

### Visual Architecture Editor
The long-term vision: a visual editor where you design architecture blocks, and an AI agent implements the code to match. The deterministic `archy.json` becomes the contract between the visual design and the implementation.

### Incremental Generation
Hash-based file change detection to only re-parse modified files. Important for large repositories.

## Technical Stack

- **Language**: TypeScript (Node.js)
- **Parser**: tree-sitter via `tree-sitter` npm bindings
- **Output**: JSON (canonical), Mermaid/D2 (rendered)
- **Distribution**: npm (`npx archy generate`)
- **Dependencies**: Minimal — tree-sitter grammars for target languages, yaml parser for config/annotations

## MVP Scope

The first working version includes:

1. **`archy init`** — project bootstrapping with language detection
2. **`archy generate`** — tree-sitter parsing for TS/JS and Python, structural graph extraction, JSON output
3. **`archy render`** — Mermaid output at system and module levels
4. **`archy validate`** — annotation anchor checking
5. **`archy diff`** — show changes since last generation
6. **`archy check`** — CI gate for stale architecture
7. **Semantic annotations** — `@archy` in-code comment directives + `.archy/externals.yaml` for boundary declarations
8. **Scope configuration** — `.archy.config.yaml` with include/exclude
9. **Basic path resolution** — tsconfig paths for TS, relative imports for Python

**Not in MVP**: C/C++ support, plugins, metrics, PR diff action, visual editor, incremental generation, `archy serve`.

## Open Questions

1. **Graph diffing algorithm**: How to produce human-readable diffs of the architecture graph (not raw JSON diff)? May need a custom differ that understands node/edge semantics.
2. **Module detection heuristic**: When should auto-detection (Louvain) be used vs. directory-based defaults? Need user testing to calibrate.
3. **Annotation generation assistants**: Should Archy include a command like `archy annotate --suggest` that uses AI to propose semantic annotations? This keeps AI out of the deterministic pipeline but uses it as a helper for the annotation authoring workflow.

## References

- [Swark — LLM-based architecture diagrams](https://github.com/swark-io/swark)
- [Codebase-Memory — Tree-sitter knowledge graphs for LLM agents](https://arxiv.org/html/2603.27277v1)
- [CodeGraph — MCP-native code knowledge graph](https://www.bighatgroup.com/blog/codegraph-2026-05-26/)
- [Drift — Architectural erosion detection](https://github.com/sauremilk/drift)
- [Codified Context — Infrastructure for AI agents in complex codebases](https://arxiv.org/html/2602.20478v1)
- [Anthropic — Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Thoughtworks — Architecture drift reduction with LLMs](https://www.thoughtworks.com/radar/techniques/architecture-drift-reduction-with-llms)
