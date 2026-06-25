# Archy Implementation Plan (DEPRECATED)

> **Status**: Deprecated — superseded by [2026-06-25-archy-implementation-v2.md](./2026-06-25-archy-implementation-v2.md)

**Goal:** Build a deterministic CLI tool that generates versioned architecture representations (JSON + Mermaid) from source code using tree-sitter, with in-code annotation support and CI enforcement.

**Architecture:** A TypeScript Node.js CLI (`archy`) organized as a pipeline: file discovery (scope config) -> tree-sitter parsing (per-language extractors) -> annotation extraction (@archy comments) -> graph assembly (merge + module detection) -> output (JSON canonical + Mermaid rendering). Each pipeline stage is a pure function operating on typed intermediate representations, making the whole pipeline deterministic and testable.

**Tech Stack:** TypeScript 5.x, Node.js 20+, tree-sitter (via `tree-sitter` npm bindings), `tree-sitter-typescript`, `tree-sitter-python`, `commander` (CLI), `js-yaml` (config/externals), `vitest` (testing), `tsup` (bundling)

## Global Constraints

- Node.js >= 20.0.0
- TypeScript strict mode, ESM modules
- Zero runtime AI dependencies — determinism is non-negotiable
- All output must be deterministic: same input files -> byte-identical JSON output
- Sort all collections (nodes, edges, modules) by ID for determinism
- Use SHA-256 for content hashing
- Commit messages follow conventional commits (`feat:`, `fix:`, `test:`, `docs:`)
- Tests written before implementation (TDD)
- Every task produces independently testable, committable work

---

## File Structure

```
archy/
  package.json
  tsconfig.json
  vitest.config.ts
  src/
    index.ts                          # CLI entry point
    cli/
      commands/
        init.ts                       # archy init
        generate.ts                   # archy generate [--dry-run]
        render.ts                     # archy render --format mermaid [--level] [--module]
        validate.ts                   # archy validate
        diff.ts                       # archy diff
        check.ts                      # archy check [--exit-code]
    config/
      loader.ts                       # Load and validate .archy.config.yaml
      schema.ts                       # Config type definitions + defaults
      scope.ts                        # File discovery using include/exclude globs
    parser/
      engine.ts                       # Tree-sitter initialization + language dispatch
      types.ts                        # Shared AST extraction types (RawNode, RawEdge)
      extractors/
        typescript.ts                 # TS/JS node/edge extraction from AST
        python.ts                     # Python node/edge extraction from AST
    annotations/
      parser.ts                       # @archy comment directive parser
      types.ts                        # Annotation type definitions
    externals/
      loader.ts                       # .archy/externals.yaml parser
      types.ts                        # External node type definitions
    resolver/
      types.ts                        # Import resolution types
      typescript.ts                   # TS/JS import path resolution
      python.ts                       # Python import path resolution
    graph/
      builder.ts                      # Merge structural + semantic + external into unified graph
      modules.ts                      # Module detection (directory-based + config overrides)
      types.ts                        # ArchyGraph, ArchyNode, ArchyEdge type definitions
      views.ts                        # Multi-level view filtering (system/module/detail)
      hash.ts                         # Deterministic hashing for source_hash, config_hash
    output/
      json.ts                         # Canonical JSON serialization (sorted, deterministic)
      mermaid.ts                      # Mermaid diagram rendering from graph views
      schema.ts                       # archy.json schema type definitions
    validation/
      anchors.ts                      # Validate ext: references exist in externals
      drift.ts                        # Structural drift detection (dry-run compare)
      cross-module.ts                 # Detect unannotated cross-module connections
  tests/
    fixtures/                         # Test fixture projects
      simple-ts/                      # Minimal TS project for parser tests
      simple-py/                      # Minimal Python project for parser tests
      annotated/                      # Project with @archy annotations
      multi-module/                   # Multi-module project for graph/view tests
    config/
      loader.test.ts
      scope.test.ts
    parser/
      engine.test.ts
      extractors/
        typescript.test.ts
        python.test.ts
    annotations/
      parser.test.ts
    externals/
      loader.test.ts
    resolver/
      typescript.test.ts
      python.test.ts
    graph/
      builder.test.ts
      modules.test.ts
      views.test.ts
      hash.test.ts
    output/
      json.test.ts
      mermaid.test.ts
    validation/
      anchors.test.ts
      drift.test.ts
      cross-module.test.ts
    cli/
      generate.integration.test.ts    # End-to-end: fixture project -> archy.json
      render.integration.test.ts      # End-to-end: archy.json -> mermaid output
```

---

## Task 1: Project Scaffolding & Core Types

**Summary:** Set up the TypeScript project, install dependencies, configure build/test tooling, and define the core type system that every other task depends on.

**Files:**
- Create: `package.json`, `tsconfig.json`, `vitest.config.ts`, `.gitignore`, `.env.example`
- Create: `src/graph/types.ts` — `ArchyGraph`, `ArchyNode`, `ArchyEdge`, `ArchyModule`
- Create: `src/output/schema.ts` — `ArchyOutput` (the full archy.json schema type)
- Create: `src/parser/types.ts` — `RawNode`, `RawEdge` (parser intermediary types)
- Create: `src/annotations/types.ts` — `ArchyDirective`, `AnnotationEdge`
- Create: `src/externals/types.ts` — `ExternalNode`, `ExternalInterface`
- Create: `src/config/schema.ts` — `ArchyConfig`, `ScopeConfig`, `EnforcementConfig`
- Create: `src/resolver/types.ts` — `ResolvedImport`, `UnresolvedImport`
- Test: `tests/graph/types.test.ts` — type guard tests

**Produces:**
- All shared TypeScript types that every subsequent task imports
- Working `npm test`, `npm run build`, `npm run lint` commands
- `ArchyNode` type: `{ id: string, type: NodeType, source: 'structural' | 'semantic' | 'external', file?: string, line?: number, module?: string, signature?: string, description?: string, metadata?: Record<string, unknown> }`
- `ArchyEdge` type: `{ from: string, to: string, type: EdgeType, source: 'structural' | 'semantic', via?: string, description?: string }`
- `ArchyGraph` type: `{ nodes: ArchyNode[], edges: ArchyEdge[], modules: ArchyModule[] }`
- `ArchyOutput` type: `{ version: string, generated_at: string, source_hash: string, config_hash: string, languages: string[], graph: ArchyGraph, views: Record<string, ViewConfig>, warnings: Warning[], metrics: Record<string, unknown> }`

**Steps:**

- [ ] Initialize npm project with `npm init`, install dependencies: `typescript`, `tree-sitter`, `tree-sitter-typescript`, `tree-sitter-python`, `commander`, `js-yaml`, `glob`. Dev deps: `vitest`, `tsup`, `@types/node`, `@types/js-yaml`
- [ ] Configure `tsconfig.json` with strict mode, ESM, `src` as rootDir, `dist` as outDir
- [ ] Configure `vitest.config.ts` with `src` alias and test file patterns
- [ ] Add `.gitignore` with `node_modules/`, `dist/`, `.env`, `*.tsbuildinfo`
- [ ] Write `src/graph/types.ts` with `ArchyNode`, `ArchyEdge`, `ArchyModule`, `ArchyGraph`, enums for `NodeType` (`function`, `class`, `interface`, `type`, `module`, `variable`, `device`, `service`) and `EdgeType` (`calls`, `imports`, `contains`, `inherits`, `implements`, `exports`, `connects-to`, `triggers`, `reads-from`, `writes-to`, `exposes`)
- [ ] Write `src/output/schema.ts` with `ArchyOutput`, `ViewConfig`, `Warning` types
- [ ] Write `src/parser/types.ts` with `RawNode`, `RawEdge` (pre-resolution intermediary types from individual extractors)
- [ ] Write `src/annotations/types.ts` with `ArchyDirective` union type for each directive, `AnnotationEdge`
- [ ] Write `src/externals/types.ts` with `ExternalNode`, `ExternalInterface`
- [ ] Write `src/config/schema.ts` with `ArchyConfig`, defaults, and validation function
- [ ] Write `src/resolver/types.ts` with `ResolvedImport`, `UnresolvedImport`
- [ ] Write type guard tests in `tests/graph/types.test.ts` to verify node/edge type narrowing
- [ ] Verify `npm test` passes, `npm run build` produces output
- [ ] Commit: `feat: scaffold project and define core type system`

---

## Task 2: Configuration System

**Summary:** Load, validate, and apply `.archy.config.yaml`. Implement file discovery using include/exclude glob patterns. This is the entry point of the pipeline — every other component needs to know what files are in scope.

**Files:**
- Create: `src/config/loader.ts` — `loadConfig(projectRoot: string): ArchyConfig`
- Create: `src/config/scope.ts` — `discoverFiles(projectRoot: string, scope: ScopeConfig): string[]`
- Test: `tests/config/loader.test.ts`
- Test: `tests/config/scope.test.ts`
- Fixture: `tests/fixtures/simple-ts/` — minimal TS project with `src/` structure

**Consumes:**
- `ArchyConfig`, `ScopeConfig` from `src/config/schema.ts`

**Produces:**
- `loadConfig(projectRoot: string): ArchyConfig` — reads `.archy.config.yaml`, validates, merges with defaults, returns typed config. Throws if invalid.
- `discoverFiles(projectRoot: string, scope: ScopeConfig): string[]` — returns sorted list of absolute file paths matching include globs minus exclude globs. Sorted alphabetically for determinism.

**Steps:**

- [ ] Create test fixture at `tests/fixtures/simple-ts/` with a minimal TS project: `src/index.ts`, `src/utils/helper.ts`, `src/utils/helper.test.ts`, `node_modules/dep/index.js` (dummy)
- [ ] Create `.archy.config.yaml` in fixture with include `src/**` and exclude `**/*.test.ts`, `node_modules/**`
- [ ] Write failing tests for `loadConfig`: loads valid config, returns defaults when no config file, throws on invalid YAML, merges partial config with defaults
- [ ] Implement `loadConfig` in `src/config/loader.ts` — reads YAML, validates against schema, merges with defaults
- [ ] Write failing tests for `discoverFiles`: respects include globs, filters out exclude patterns, sorts deterministically, handles empty scope
- [ ] Implement `discoverFiles` in `src/config/scope.ts` using `glob` package
- [ ] Verify all tests pass
- [ ] Commit: `feat: add configuration loading and file discovery`

---

## Task 3: Tree-sitter Parser Engine

**Summary:** Initialize tree-sitter with language grammars, dispatch files to the correct language parser, and define the abstract interface that language-specific extractors implement. This task builds the engine — the extractors come in Tasks 4 and 5.

**Files:**
- Create: `src/parser/engine.ts` — `parseProject(files: string[], config: ArchyConfig): RawExtractionResult`
- Test: `tests/parser/engine.test.ts`

**Consumes:**
- `discoverFiles` from `src/config/scope.ts`
- `ArchyConfig` from `src/config/schema.ts`
- `RawNode`, `RawEdge` from `src/parser/types.ts`

**Produces:**
- `LanguageExtractor` interface: `{ supportedExtensions: string[], extract(tree: Tree, filePath: string, source: string): { nodes: RawNode[], edges: RawEdge[] } }`
- `parseProject(files: string[], config: ArchyConfig): RawExtractionResult` — reads each file, selects parser by extension, runs tree-sitter, dispatches to extractor, accumulates results
- `RawExtractionResult`: `{ nodes: RawNode[], edges: RawEdge[], languages: string[], parseErrors: ParseError[] }`

**Steps:**

- [ ] Write failing tests: parser engine initializes tree-sitter, identifies TS files correctly, identifies Python files correctly, returns empty result for unsupported extensions, accumulates nodes/edges from multiple files
- [ ] Implement `LanguageExtractor` interface in `src/parser/engine.ts`
- [ ] Implement `parseProject` — file reading, extension-based language dispatch, tree-sitter parse, extractor invocation, result accumulation
- [ ] Register a stub extractor for `.ts`/`.tsx`/`.js`/`.jsx` and `.py` that returns empty results (real extractors in Tasks 4-5)
- [ ] Write integration test with fixture: parse `tests/fixtures/simple-ts/` and verify it finds the right files and returns a `RawExtractionResult`
- [ ] Verify all tests pass
- [ ] Commit: `feat: add tree-sitter parser engine with language dispatch`

---

## Task 4: TypeScript/JavaScript Extractor

**Summary:** Walk the TS/JS AST from tree-sitter and extract structural nodes (functions, classes, interfaces, type aliases, variables) and edges (imports, calls, contains, inherits/implements, exports). This is the richest and most important extractor.

**Files:**
- Create: `src/parser/extractors/typescript.ts` — implements `LanguageExtractor`
- Test: `tests/parser/extractors/typescript.test.ts`
- Fixture: expand `tests/fixtures/simple-ts/` with richer code examples

**Consumes:**
- `LanguageExtractor` interface from `src/parser/engine.ts`
- `RawNode`, `RawEdge` from `src/parser/types.ts`

**Produces:**
- `TypeScriptExtractor` class implementing `LanguageExtractor`
- Extracts: function declarations, arrow functions (named exports), class declarations, interface declarations, type alias declarations, enum declarations
- Extracts edges: `import_statement` -> `imports` edges, `call_expression` -> `calls` edges, `class_heritage` -> `inherits`/`implements` edges, `export_statement` -> `exports` edges, class member containment -> `contains` edges

**Steps:**

- [ ] Expand `tests/fixtures/simple-ts/` with representative TS code: classes with inheritance, interfaces, exported functions, import statements, call chains, type aliases
- [ ] Write failing tests for node extraction: finds function declarations with correct file/line/name, finds class declarations, finds interface declarations, finds type aliases, finds arrow function exports
- [ ] Implement node extraction in `TypeScriptExtractor` — walk AST for `function_declaration`, `class_declaration`, `interface_declaration`, `type_alias_declaration`, `lexical_declaration` (arrow functions)
- [ ] Run tests, verify passing
- [ ] Write failing tests for edge extraction: import statements produce `imports` edges, call expressions produce `calls` edges, class heritage produces `inherits`/`implements` edges, export statements produce `exports` edges
- [ ] Implement edge extraction — walk AST for `import_statement`, `call_expression`, `class_heritage`, `export_statement`
- [ ] Write failing tests for `contains` edges: methods inside classes, nested functions
- [ ] Implement `contains` edge extraction
- [ ] Write determinism test: parse same file twice, verify byte-identical output
- [ ] Verify all tests pass
- [ ] Commit: `feat: add TypeScript/JavaScript AST extractor`

---

## Task 5: Python Extractor

**Summary:** Walk the Python AST from tree-sitter and extract structural nodes and edges. Similar to Task 4 but for Python's different syntax (def, class, decorators, relative imports).

**Files:**
- Create: `src/parser/extractors/python.ts` — implements `LanguageExtractor`
- Test: `tests/parser/extractors/python.test.ts`
- Fixture: Create `tests/fixtures/simple-py/` with representative Python code

**Consumes:**
- `LanguageExtractor` interface from `src/parser/engine.ts`
- `RawNode`, `RawEdge` from `src/parser/types.ts`

**Produces:**
- `PythonExtractor` class implementing `LanguageExtractor`
- Extracts: `def` functions, classes, decorated functions/classes
- Extracts edges: `import_statement`/`import_from_statement` -> `imports`, `call` -> `calls`, class inheritance -> `inherits`, method containment -> `contains`

**Steps:**

- [ ] Create `tests/fixtures/simple-py/` with: classes with inheritance, functions, imports (relative and absolute), decorators, `__init__.py` files, call chains
- [ ] Write failing tests for node extraction: finds `def` functions, finds classes, finds decorated functions, captures correct line numbers
- [ ] Implement node extraction in `PythonExtractor` — walk AST for `function_definition`, `class_definition`
- [ ] Run tests, verify passing
- [ ] Write failing tests for edge extraction: import statements, from-imports, relative imports, call expressions, class bases
- [ ] Implement edge extraction
- [ ] Write determinism test: parse same file twice, verify identical output
- [ ] Verify all tests pass
- [ ] Commit: `feat: add Python AST extractor`

---

## Task 6: Import Path Resolution

**Summary:** Resolve raw import strings (e.g., `@/utils/helper`, `from ..models import User`) to actual file paths within the project. This turns raw `imports` edges (which contain unresolved import strings) into edges pointing to real file:symbol targets.

**Files:**
- Create: `src/resolver/typescript.ts` — `resolveTypeScriptImport(importPath: string, fromFile: string, config: ArchyConfig): ResolvedImport | UnresolvedImport`
- Create: `src/resolver/python.ts` — `resolvePythonImport(importPath: string, fromFile: string, config: ArchyConfig): ResolvedImport | UnresolvedImport`
- Test: `tests/resolver/typescript.test.ts`
- Test: `tests/resolver/python.test.ts`
- Fixture: expand `tests/fixtures/simple-ts/` with tsconfig paths, add `tsconfig.json`

**Consumes:**
- `ResolvedImport`, `UnresolvedImport` from `src/resolver/types.ts`
- `ArchyConfig` from `src/config/schema.ts`

**Produces:**
- `resolveTypeScriptImport(importPath: string, fromFile: string, config: ArchyConfig): ResolvedImport | UnresolvedImport` — resolves relative imports, tsconfig paths aliases, bare specifiers (node_modules -> mark as external)
- `resolvePythonImport(importPath: string, fromFile: string, config: ArchyConfig): ResolvedImport | UnresolvedImport` — resolves relative imports (`from .foo`), absolute imports within project, marks third-party as external
- `ResolvedImport`: `{ resolvedPath: string, symbols: string[] }`
- `UnresolvedImport`: `{ rawPath: string, reason: string }` — logged as a warning

**Steps:**

- [ ] Add `tsconfig.json` to `tests/fixtures/simple-ts/` with path aliases (`@/*` -> `src/*`)
- [ ] Write failing tests for TS resolution: resolves relative imports (`./foo`), resolves tsconfig path aliases (`@/utils/helper`), resolves barrel imports (`index.ts` fallback), marks `node_modules` imports as external, returns `UnresolvedImport` for broken paths
- [ ] Implement `resolveTypeScriptImport` — read tsconfig.json `paths`/`baseUrl`, resolve relative paths, handle `.ts`/`.tsx`/`.js`/`.jsx`/`index.ts` fallbacks
- [ ] Run tests, verify passing
- [ ] Write failing tests for Python resolution: resolves relative imports (`from .utils import helper`), resolves absolute imports within project root, marks third-party packages as external
- [ ] Implement `resolvePythonImport` — detect project root via `pyproject.toml`/`setup.py`, resolve relative imports by directory traversal, resolve absolute imports by matching to file paths under project root
- [ ] Verify all tests pass
- [ ] Commit: `feat: add import path resolution for TypeScript and Python`

---

## Task 7: @archy Annotation Parser

**Summary:** Parse `@archy` comment directives from tree-sitter-extracted comments. During the same AST pass that extracts structural nodes, comments containing `@archy` are parsed into typed directives and associated with the nearest subsequent code entity.

**Files:**
- Create: `src/annotations/parser.ts` — `parseAnnotations(tree: Tree, source: string, filePath: string): AnnotationResult`
- Test: `tests/annotations/parser.test.ts`
- Fixture: Create `tests/fixtures/annotated/` with annotated code in TS and Python

**Consumes:**
- `ArchyDirective`, `AnnotationEdge` from `src/annotations/types.ts`

**Produces:**
- `parseAnnotations(tree: Tree, source: string, filePath: string): AnnotationResult`
- `AnnotationResult`: `{ edges: AnnotationEdge[], nodeMetadata: NodeMetadata[], parseErrors: AnnotationParseError[] }`
- Parses directives: `connects-to`, `triggers`, `reads-from`, `writes-to`, `exposes`, `layer`, `description`, `group`
- Associates each directive with the nearest following code entity (function/class/variable declaration)

**Steps:**

- [ ] Create `tests/fixtures/annotated/` with TS files containing `@archy` comments: `connects-to` with `via`, `triggers`, `description`, `layer`, `group`, malformed annotations
- [ ] Add Python files with `# @archy` comments to the same fixture
- [ ] Write failing tests: extracts `connects-to` directive with target and via, extracts `triggers` directive, extracts `description` directive (quoted string), extracts `layer` and `group` directives, associates directive with next code entity, returns parse error for malformed directive, handles multiple directives on same entity
- [ ] Implement `parseAnnotations` — walk AST comment nodes, regex match `@archy`, parse directive + args, find nearest subsequent sibling declaration node, associate directive with file:symbol ID
- [ ] Write failing test: annotations in Python comments (`# @archy`) parse identically to TS comments (`// @archy`)
- [ ] Verify language-agnostic parsing (both `//` and `#` comment styles)
- [ ] Verify all tests pass
- [ ] Commit: `feat: add @archy in-code annotation parser`

---

## Task 8: Externals Loader

**Summary:** Parse `.archy/externals.yaml` to load external boundary declarations (devices, services, APIs that live outside the codebase).

**Files:**
- Create: `src/externals/loader.ts` — `loadExternals(projectRoot: string): ExternalNode[]`
- Test: `tests/externals/loader.test.ts`
- Fixture: Add `.archy/externals.yaml` to `tests/fixtures/annotated/`

**Consumes:**
- `ExternalNode`, `ExternalInterface` from `src/externals/types.ts`

**Produces:**
- `loadExternals(projectRoot: string): ExternalNode[]` — reads `.archy/externals.yaml`, validates, returns typed external nodes. Returns empty array if file doesn't exist.

**Steps:**

- [ ] Add `.archy/externals.yaml` to `tests/fixtures/annotated/` with a device and a service external
- [ ] Write failing tests: loads externals from YAML, validates `ext:` prefix on IDs, returns empty array when file missing, throws on invalid YAML, parses interfaces correctly
- [ ] Implement `loadExternals` in `src/externals/loader.ts`
- [ ] Verify all tests pass
- [ ] Commit: `feat: add externals.yaml loader for boundary declarations`

---

## Task 9: Graph Builder & Module Detection

**Summary:** The core assembly step. Takes raw extracted nodes/edges (structural), resolved imports, annotation edges (semantic), and external nodes — merges them into the unified `ArchyGraph`. Assigns module membership to every node. Detects warnings.

**Files:**
- Create: `src/graph/builder.ts` — `buildGraph(raw: RawExtractionResult, resolved: ResolvedImport[], annotations: AnnotationResult, externals: ExternalNode[], config: ArchyConfig): ArchyGraph`
- Create: `src/graph/modules.ts` — `assignModules(nodes: ArchyNode[], config: ArchyConfig): ArchyModule[]`
- Create: `src/graph/hash.ts` — `computeSourceHash(files: string[]): string`, `computeConfigHash(configPath: string, externalsPath: string): string`
- Test: `tests/graph/builder.test.ts`
- Test: `tests/graph/modules.test.ts`
- Test: `tests/graph/hash.test.ts`
- Fixture: Create `tests/fixtures/multi-module/` with a multi-module TS project

**Consumes:**
- `RawExtractionResult` from `src/parser/engine.ts`
- `AnnotationResult` from `src/annotations/parser.ts`
- `ExternalNode` from `src/externals/loader.ts`
- `ResolvedImport` from `src/resolver/types.ts`
- `ArchyConfig` from `src/config/schema.ts`

**Produces:**
- `buildGraph(...)`: ArchyGraph — the fully merged, module-assigned, sorted graph
- `assignModules(nodes: ArchyNode[], config: ArchyConfig): ArchyModule[]` — directory-based default or config override
- `computeSourceHash(files: string[]): string` — SHA-256 of concatenated sorted file contents
- `computeConfigHash(configPath: string, externalsPath: string): string` — SHA-256 of config + externals content

**Steps:**

- [ ] Create `tests/fixtures/multi-module/` with `src/api/routes.ts`, `src/api/middleware.ts`, `src/services/queue.ts`, `src/workers/processor.ts`, `.archy.config.yaml` defining modules
- [ ] Write failing tests for `assignModules`: assigns module by directory path, respects config overrides, handles nested paths
- [ ] Implement `assignModules` — match each node's file path against configured module paths, fall back to top-level directory
- [ ] Write failing tests for `buildGraph`: merges structural nodes/edges with annotation edges, adds external nodes, resolves import edges to point to real file:symbol targets, sorts all collections by ID, deduplicates edges
- [ ] Implement `buildGraph` — merge all sources, replace raw import paths with resolved paths, assign modules, sort everything, collect warnings for unresolved imports
- [ ] Write failing tests for `computeSourceHash`: identical files produce same hash, different files produce different hash, file order doesn't matter (sorted internally)
- [ ] Implement `computeSourceHash` and `computeConfigHash`
- [ ] Write determinism test: build graph twice from same inputs, verify byte-identical JSON serialization
- [ ] Verify all tests pass
- [ ] Commit: `feat: add graph builder, module detection, and deterministic hashing`

---

## Task 10: Multi-Level View Filtering

**Summary:** Filter the unified graph into zoom levels (system/module/detail) for rendering and AI context. Each view is a projection of the same graph — not a separate data structure.

**Files:**
- Create: `src/graph/views.ts` — `filterView(graph: ArchyGraph, level: ViewLevel, options?: ViewOptions): ArchyGraph`
- Test: `tests/graph/views.test.ts`

**Consumes:**
- `ArchyGraph`, `ArchyModule` from `src/graph/types.ts`

**Produces:**
- `filterView(graph: ArchyGraph, level: ViewLevel, options?: ViewOptions): ArchyGraph`
- `ViewLevel`: `'system' | 'module' | 'detail'`
- `ViewOptions`: `{ moduleId?: string, fileId?: string }`
- System view: only module nodes + external nodes + cross-module edges
- Module view: all nodes in a specific module + edges to/from that module
- Detail view: all nodes in a specific file + their edges

**Steps:**

- [ ] Write failing tests for system view: collapses nodes into module-level, keeps only cross-module and external edges, aggregates edge counts between modules
- [ ] Implement system view in `filterView`
- [ ] Write failing tests for module view: includes all nodes in specified module, includes edges within the module and crossing its boundary, excludes nodes/edges from other modules
- [ ] Implement module view
- [ ] Write failing tests for detail view: includes all nodes in specified file, includes all edges involving those nodes
- [ ] Implement detail view
- [ ] Verify all tests pass
- [ ] Commit: `feat: add multi-level view filtering for graph projections`

---

## Task 11: Canonical JSON Output

**Summary:** Serialize the `ArchyGraph` into the canonical `archy.json` format. This must be deterministic — same graph always produces byte-identical JSON.

**Files:**
- Create: `src/output/json.ts` — `serializeOutput(graph: ArchyGraph, metadata: OutputMetadata): string`
- Test: `tests/output/json.test.ts`

**Consumes:**
- `ArchyGraph` from `src/graph/types.ts`
- `ArchyOutput` from `src/output/schema.ts`

**Produces:**
- `serializeOutput(graph: ArchyGraph, metadata: OutputMetadata): string` — returns deterministic JSON string with 2-space indentation
- `OutputMetadata`: `{ version: string, sourceHash: string, configHash: string, languages: string[], warnings: Warning[] }`
- JSON keys sorted within each object, arrays sorted by ID
- `generated_at` is NOT included in the serialized output by default (breaks determinism) — stored separately or passed as option

**Important determinism note:** The `generated_at` timestamp must be excluded from the hash comparison in `archy check` / `archy diff`. Two runs of the same code produce the same output except for the timestamp. The comparison logic ignores this field.

**Steps:**

- [ ] Write failing tests: serializes a graph to valid JSON matching `ArchyOutput` schema, produces identical output for identical input, sorts node array by ID, sorts edge array by `from` then `to`, sorts module array by ID, excludes `generated_at` from deterministic comparison
- [ ] Implement `serializeOutput` with sorted keys via custom `JSON.stringify` replacer, sorted collections
- [ ] Write round-trip test: serialize then parse, verify graph is identical
- [ ] Verify all tests pass
- [ ] Commit: `feat: add deterministic JSON serialization for archy.json`

---

## Task 12: Mermaid Renderer

**Summary:** Convert an `ArchyGraph` (at any view level) into Mermaid diagram syntax. System-level renders as a flowchart with subgraphs per module. Module-level renders as a detailed flowchart. External nodes render with dashed styling.

**Files:**
- Create: `src/output/mermaid.ts` — `renderMermaid(graph: ArchyGraph, options?: MermaidOptions): string`
- Test: `tests/output/mermaid.test.ts`

**Consumes:**
- `ArchyGraph`, `ArchyNode`, `ArchyEdge` from `src/graph/types.ts`

**Produces:**
- `renderMermaid(graph: ArchyGraph, options?: MermaidOptions): string` — returns valid Mermaid flowchart syntax
- `MermaidOptions`: `{ direction?: 'TB' | 'LR', title?: string }`
- Module boundaries render as `subgraph` blocks
- External nodes render with `:::dashed` styling
- Edge labels show type (calls, triggers, connects-to, etc.)
- Semantic edges render with dotted lines to distinguish from structural

**Steps:**

- [ ] Write failing tests: renders empty graph as minimal valid Mermaid, renders nodes with correct IDs and labels, renders structural edges as solid arrows, renders semantic edges as dotted arrows, renders external nodes with dashed style, wraps module members in subgraph blocks, produces deterministic output
- [ ] Implement `renderMermaid` — node ID sanitization (Mermaid-safe characters), subgraph grouping by module, edge rendering with labels, external node styling
- [ ] Write snapshot test: render the `multi-module` fixture graph and compare against expected Mermaid output
- [ ] Verify Mermaid output is valid by checking basic syntax rules (starts with `flowchart`, valid arrow syntax)
- [ ] Verify all tests pass
- [ ] Commit: `feat: add Mermaid diagram renderer`

---

## Task 13: Validation & Drift Detection

**Summary:** Implement the three types of drift detection: structural staleness, annotation anchor validity, and unannotated cross-module connections. These power `archy validate` and `archy check`.

**Files:**
- Create: `src/validation/anchors.ts` — `validateAnchors(graph: ArchyGraph, externals: ExternalNode[]): AnchorValidationResult`
- Create: `src/validation/drift.ts` — `detectDrift(currentJson: string, committedJsonPath: string): DriftResult`
- Create: `src/validation/cross-module.ts` — `findUnannotatedCrossModuleEdges(graph: ArchyGraph): CrossModuleViolation[]`
- Test: `tests/validation/anchors.test.ts`
- Test: `tests/validation/drift.test.ts`
- Test: `tests/validation/cross-module.test.ts`

**Consumes:**
- `ArchyGraph` from `src/graph/types.ts`
- `ExternalNode` from `src/externals/types.ts`

**Produces:**
- `validateAnchors(graph: ArchyGraph, externals: ExternalNode[]): AnchorValidationResult` — checks that `ext:` targets in semantic edges exist in externals list
- `detectDrift(currentJson: string, committedJsonPath: string): DriftResult` — compares generated output to committed file, ignoring `generated_at`, returns `{ isDrifted: boolean, addedNodes: string[], removedNodes: string[], addedEdges: string[], removedEdges: string[] }`
- `findUnannotatedCrossModuleEdges(graph: ArchyGraph): CrossModuleViolation[]` — finds structural edges crossing module boundaries that don't have a corresponding semantic edge

**Steps:**

- [ ] Write failing tests for `validateAnchors`: passes when all ext: targets exist, fails when ext: target is missing, returns detailed error with the offending annotation location
- [ ] Implement `validateAnchors`
- [ ] Write failing tests for `detectDrift`: no drift when files are identical (ignoring timestamp), detects added nodes, detects removed edges, detects modified edges
- [ ] Implement `detectDrift` — parse both JSON, compare graphs structurally
- [ ] Write failing tests for `findUnannotatedCrossModuleEdges`: returns empty for fully annotated graph, returns violations for cross-module structural edges without matching semantic edges, doesn't flag intra-module edges
- [ ] Implement `findUnannotatedCrossModuleEdges`
- [ ] Verify all tests pass
- [ ] Commit: `feat: add drift detection and validation checks`

---

## Task 14: CLI Commands

**Summary:** Wire everything together into the CLI using `commander`. Each command orchestrates the pipeline stages built in Tasks 2-13. This is the integration layer.

**Files:**
- Create: `src/index.ts` — CLI entry point with `commander` program definition
- Create: `src/cli/commands/generate.ts` — `archy generate [--dry-run]`
- Create: `src/cli/commands/render.ts` — `archy render --format mermaid [--level] [--module]`
- Create: `src/cli/commands/validate.ts` — `archy validate`
- Create: `src/cli/commands/diff.ts` — `archy diff`
- Create: `src/cli/commands/check.ts` — `archy check [--exit-code]`
- Create: `src/cli/commands/init.ts` — `archy init`
- Test: `tests/cli/generate.integration.test.ts`
- Test: `tests/cli/render.integration.test.ts`

**Consumes:** Everything from Tasks 2-13.

**Produces:**
- `archy generate` — runs full pipeline: config -> scope -> parse -> resolve -> annotate -> externals -> build graph -> serialize -> write `archy.json`
- `archy generate --dry-run` — runs pipeline but doesn't write, outputs to stdout
- `archy render --format mermaid --level system` — reads `archy.json`, filters to view, renders Mermaid, outputs to stdout
- `archy validate` — runs anchor validation and cross-module checks, reports results
- `archy diff` — runs `generate --dry-run` and diffs against committed `archy.json`
- `archy check` — same as diff but exits with code 1 if drifted (CI gate)
- `archy init` — detects languages, generates default config, runs first generate

**Steps:**

- [ ] Set up `commander` program in `src/index.ts` with version, description, and subcommands
- [ ] Implement `generate` command — orchestrate full pipeline, write `archy.json`, print summary (node/edge/module counts)
- [ ] Implement `generate --dry-run` — same pipeline, output JSON to stdout instead of file
- [ ] Write integration test: run generate against `tests/fixtures/multi-module/`, verify `archy.json` is created with expected structure
- [ ] Implement `render` command — read `archy.json`, filter to requested view, render Mermaid to stdout
- [ ] Write integration test: generate then render against fixture, verify valid Mermaid output
- [ ] Implement `validate` command — load graph + externals, run `validateAnchors` + `findUnannotatedCrossModuleEdges`, print results, exit code based on enforcement config
- [ ] Implement `diff` command — generate dry-run, compare against committed file, print human-readable diff summary
- [ ] Implement `check` command — same as diff, exit 1 if drifted
- [ ] Implement `init` command — scan for languages by extension, generate default `.archy.config.yaml`, create `.archy/` directory, run first generate, print Mermaid diagram to stdout
- [ ] Add `bin` field to `package.json` pointing to `dist/index.js`
- [ ] Verify `npx . generate` works from the project root on a fixture
- [ ] Verify all tests pass
- [ ] Commit: `feat: add CLI commands (generate, render, validate, diff, check, init)`

---

## Task 15: End-to-End Integration & Documentation

**Summary:** Run the full tool against a realistic test project to verify everything works together. Write a README with usage instructions. Polish edge cases found during integration testing.

**Files:**
- Create: `README.md`
- Create: `tests/fixtures/e2e-project/` — a more realistic multi-file project with annotations
- Modify: any files with bugs found during integration

**Consumes:** All previous tasks

**Produces:**
- `README.md` with: what Archy is, installation, quick start, configuration reference, annotation syntax reference, CLI reference, CI integration example
- Passing end-to-end test that exercises the full pipeline on a realistic fixture
- Verified `npx archy init && npx archy generate && npx archy render --format mermaid --level system` workflow

**Steps:**

- [ ] Create `tests/fixtures/e2e-project/` with: multiple TS files across 3+ modules, `@archy` annotations including `connects-to`, `triggers`, `layer`, `.archy/externals.yaml` with 1-2 external nodes, `.archy.config.yaml` with scope, modules, and enforcement config
- [ ] Write end-to-end test: run full generate pipeline, verify output JSON matches expected structure, verify Mermaid rendering produces valid output, verify validate catches intentionally broken annotations, verify check detects drift when code is modified
- [ ] Run end-to-end test, fix any integration bugs found
- [ ] Write `README.md` with installation, quick start, full CLI reference, annotation syntax reference, config reference, CI example (GitHub Actions snippet)
- [ ] Test `npm pack` and verify the package installs and runs correctly
- [ ] Verify all tests pass (unit + integration + e2e)
- [ ] Commit: `feat: add e2e integration tests and README documentation`

---

## Task Dependency Graph

```
Task 1 (scaffolding + types)
  ├── Task 2 (config + scope)
  │     └── Task 3 (parser engine)
  │           ├── Task 4 (TS extractor)
  │           ├── Task 5 (Python extractor)
  │           └── Task 7 (annotation parser)
  ├── Task 6 (import resolution) — depends on Tasks 4, 5
  ├── Task 8 (externals loader)
  └── Task 9 (graph builder) — depends on Tasks 3-8
        ├── Task 10 (view filtering)
        ├── Task 11 (JSON output)
        ├── Task 12 (Mermaid renderer) — depends on Task 10
        └── Task 13 (validation) — depends on Task 11
              └── Task 14 (CLI commands) — depends on Tasks 9-13
                    └── Task 15 (e2e + docs) — depends on Task 14
```

**Parallelizable groups:**
- Tasks 4, 5, 7, 8 can run in parallel (independent extractors)
- Tasks 10, 11 can run in parallel (independent output stages)
- Task 6 can start as soon as Tasks 4, 5 are done

---

## Estimated Effort

| Task | Complexity | Est. Time |
|------|-----------|-----------|
| 1. Scaffolding + Types | Low | 1-2 hours |
| 2. Config System | Low | 1-2 hours |
| 3. Parser Engine | Medium | 2-3 hours |
| 4. TS Extractor | High | 4-6 hours |
| 5. Python Extractor | Medium | 3-4 hours |
| 6. Import Resolution | High | 4-6 hours |
| 7. Annotation Parser | Medium | 2-3 hours |
| 8. Externals Loader | Low | 1 hour |
| 9. Graph Builder | High | 4-6 hours |
| 10. View Filtering | Medium | 2-3 hours |
| 11. JSON Output | Low | 1-2 hours |
| 12. Mermaid Renderer | Medium | 3-4 hours |
| 13. Validation | Medium | 2-3 hours |
| 14. CLI Commands | Medium | 3-4 hours |
| 15. E2E + Docs | Medium | 3-4 hours |
| **Total** | | **~35-50 hours** |
