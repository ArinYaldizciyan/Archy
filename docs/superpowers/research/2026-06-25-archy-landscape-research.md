# Archy: Deep Landscape Research & Feasibility Analysis

**Date**: 2026-06-25  
**Status**: Research Complete  
**Ticket**: SHV-129  

## Executive Summary

The space Archy targets — deterministic architecture extraction from source code — is **extremely active in 2026**. Multiple well-funded, well-adopted tools exist. However, none of them combine all of Archy's proposed capabilities into one tool. The key question is: **should Archy build from scratch, integrate with existing tools, or pivot its focus?**

---

## 1. Existing Tools (Ranked by Relevance to Archy)

### 1.1 Graphify — ⭐ 63K+ stars, YC S26
- **What it does**: Multi-stage pipeline: detect → extract (AST + LLM) → build (NetworkX graph) → cluster (Leiden communities) → analyze → report → export (HTML/JSON/Obsidian)
- **How it works**: Pass 1 is deterministic tree-sitter AST extraction (functions, classes, imports, calls). Pass 2 uses LLM for semantic enrichment of docs/papers/diagrams. Pass 3 builds NetworkX graph with Leiden community clustering.
- **Export formats**: `graph.json`, `graph.html` (interactive), `GRAPH_REPORT.md`
- **Git integration**: Git hooks (`post-commit`, `post-checkout`) auto-rebuild graph. Union-merge driver for `graph.json`. SHA256-based cache for incremental rebuilds.
- **Language support**: 33 languages via tree-sitter
- **Platform support**: Claude Code, Codex, Cursor, Gemini CLI, Copilot CLI, and 6+ more
- **Token efficiency**: 71.5x fewer tokens per query vs. raw file reading
- **Key limitation**: **Not a CI enforcement tool**. No architectural constraint definition. No annotation system for semantic connections. The LLM enrichment pass makes it non-deterministic for non-code content. No `diff` or `check` commands for CI gates.

**Sources**: [Graphify GitHub](https://github.com/safishamsi/graphify), [Graphify Website](https://graphify.net/), [Augment Code coverage](https://www.augmentcode.com/learn/graphify-63k-stars-knowledge-graphs)

### 1.2 Codebase-Memory (MCP) — ⭐ 14.5K stars
- **What it does**: Persistent tree-sitter knowledge graph exposed via MCP protocol. Single C binary, zero dependencies, 158 languages.
- **Performance**: Sub-ms queries, indexes average repo in milliseconds. 83% answer quality, 10x fewer tokens, 2.1x fewer tool calls vs. file exploration.
- **Pipeline**: Parse (tree-sitter AST) → store (SQLite knowledge graph) → expose (14 MCP tools). Background file watcher with XXH3 content hashing for incremental re-indexing.
- **Integration**: Works with 11 agents (Claude Code, Codex CLI, Gemini CLI, Cursor, Zed)
- **Research backing**: Peer-reviewed preprint [arxiv:2603.27277](https://arxiv.org/abs/2603.27277), evaluated across 31 real-world repositories
- **Key limitation**: **Runtime query engine, not a versioned artifact.** No git-diffable output. No CI enforcement. No annotation system.

**Sources**: [GitHub](https://github.com/DeusData/codebase-memory-mcp), [arXiv paper](https://arxiv.org/abs/2603.27277)

### 1.3 Structurizr / C4 Model — Established, widely adopted
- **What it does**: "Architecture as code" using a text-based DSL. Produces C4 diagrams (Context, Container, Component, Code) from versioned `.dsl` files.
- **Approach**: **Top-down** — humans write the architecture model, tool renders it. The opposite of Archy's bottom-up extraction.
- **Git integration**: DSL files version cleanly in git, support code review workflows.
- **Key limitation**: **Manual authoring required.** Doesn't extract architecture from code. Drifts from reality unless manually maintained.

**Sources**: [Structurizr](https://structurizr.com/), [DSL Docs](https://docs.structurizr.com/dsl)

### 1.4 Axivion Suite (Qt) — Commercial, enterprise
- **What it does**: Architecture model written as Python code, verified by static analysis in CI/CD pipeline. Every commit checked against architectural model.
- **Approach**: Define architecture as code → map to source → enforce in CI. Violations blocked before merge.
- **Languages**: C, C++, C#, Java (enterprise focus)
- **Key insight**: **This is the closest to Archy's enforcement vision**, but it's commercial, enterprise-focused, and requires manual architecture definition (top-down).
- **Key limitation**: **Not open source.** Top-down (requires manual architecture definition). No auto-extraction from code. No multi-language lightweight CLI.

**Sources**: [Axivion Architecture Verification](https://www.qt.io/quality-assurance/axivion-architecture-verification), [Architecture as Code blog](https://www.qt.io/software-insights/architecture-as-code-a-developer-friendly-approach-to-architecture-verification)

### 1.5 dependency-cruiser — ⭐ ~6K stars, mature
- **What it does**: Validates and visualizes JS/TS dependencies against configurable rules. CI enforcement of dependency constraints.
- **Key strength**: Battle-tested rule engine. Catches circular deps, forbidden paths, orphan modules.
- **Key limitation**: **JS/TS only.** Dependency graph != architecture. No semantic layer. No annotation system.

**Sources**: [GitHub](https://github.com/sverweij/dependency-cruiser)

### 1.6 skott — ⭐ ~3K stars
- **What it does**: JS/TS dependency graph with interactive web visualization, circular dependency detection, unused dependency detection.
- **Key strength**: Rich visualization (embedded web app, SVG, PNG, JSON export). Graph API with DFS/BFS traversal.
- **Key limitation**: **JS/TS only.** Dependency-focused, not architecture-focused.

**Sources**: [GitHub](https://github.com/antoine-coulon/skott)

### 1.7 Swark — VS Code extension
- **What it does**: LLM-powered architecture diagram generation via GitHub Copilot. Outputs Mermaid.
- **Key limitation**: **Non-deterministic.** Can't version or diff reliably. Token-limited (adjusts files to fit LLM context window).

**Sources**: [GitHub](https://github.com/swark-io/swark), [Medium article](https://medium.com/@ozanani/introducing-swark-automatic-architecture-diagrams-from-code-cb5c8af7a7a5)

### 1.8 Drift — Early stage
- **What it does**: Detects architectural erosion from AI-generated code. Static analysis for pattern fragmentation and architecture violations.
- **Key limitation**: **Python only.** Detection, not generation or enforcement.

**Sources**: [GitHub](https://github.com/sauremilk/drift)

---

## 2. Academic Research (Peer-Reviewed & Preprints)

### 2.1 Repository Intelligence Graph (RIG) / SPADE
- **Paper**: [arxiv:2601.10112](https://arxiv.org/abs/2601.10112) (Jan 2026)
- **Authors**: Cherny-Shahar & Yehudai (Blavatnik Foundation funded)
- **Key contribution**: Deterministic, evidence-backed architectural map from build/test artifacts. Exposed as LLM-friendly JSON.
- **Currently**: CMake-based only. Open source at [Greenfuze/Spade](https://github.com/Greenfuze/Spade).
- **Relevance to Archy**: **Identical philosophy** (deterministic + versioned + JSON + LLM-friendly), but focused on build structure, not code architecture. Could be complementary rather than competitive.

### 2.2 SSAR (ICSE 2026)
- **Paper**: Presented at ICSE 2026 Research Track
- **Authors**: Ding, Mo, Wu, Song (Central China Normal University)
- **Key contribution**: Architecture recovery integrating semantic similarity + structural dependencies. Improves accuracy 5-90.9% over existing techniques.
- **Relevance**: Validates that combining semantic + structural analysis produces better architecture recovery. Supports Archy's multi-layer approach.

### 2.3 CIAO — Code In Architecture Out
- **Paper**: [arxiv:2604.08293](https://arxiv.org/abs/2604.08293) (Apr 2026)
- **Key contribution**: LLM-based system-level architecture documentation from GitHub repos. Follows ISO/IEC/IEEE 42010, C4 model templates. Evaluated with 22 developers.
- **Relevance**: Shows the LLM approach works for documentation but **not for enforcement or versioning**. Complements but doesn't replace Archy.

### 2.4 LLM-based Automated Architecture View Generation
- **Paper**: [arxiv:2603.21178](https://arxiv.org/abs/2603.21178) (Mar 2026)
- **Key finding**: Evaluated 4,137 generated views across 340 repos. **LLMs consistently exhibit granularity mismatches** — they operate at code level rather than architectural abstractions. 22.6% clarity failure rate even with the best approach.
- **Relevance**: **Strong validation for Archy's deterministic approach.** LLMs are not reliable enough for architecture generation. Static analysis is more appropriate.

### 2.5 ArchAgent
- **Paper**: [arxiv:2601.13007](https://arxiv.org/abs/2601.13007) (Jan 2026)
- **Key contribution**: Agent-based framework combining static analysis + LLM synthesis for legacy architecture recovery.
- **Relevance**: Hybrid approach. Shows static analysis alone isn't always sufficient for legacy code, but Archy's target (active projects with evolving architecture) is different from legacy recovery.

### 2.6 RAD-AI
- **Paper**: [arxiv:2603.28735](https://arxiv.org/abs/2603.28735) (Mar 2026, accepted ANGE 2026 @ IEEE ICSA)
- **Key contribution**: Architecture documentation extensions for AI-augmented ecosystems. Extends arc42/C4 with AI-specific sections. EU AI Act compliance.
- **Relevance**: Shows the need for **new architecture documentation approaches** beyond traditional frameworks. Archy's annotation-based approach is aligned with this direction.

### 2.7 Codified Context
- **Paper**: [arxiv:2602.20478](https://arxiv.org/abs/2602.20478) (Feb 2026)
- **Key contribution**: 3-component infrastructure for AI agent context: constitution, domain experts, knowledge base. Tested across 283 sessions on a 108K-line C# project.
- **Relevance**: Validates the need for structured, persistent context for AI agents. Archy's `archy.json` as context is aligned.

### 2.8 Comparison of Static Analysis Architecture Recovery Tools (Springer, peer-reviewed)
- **Paper**: [Springer](https://link.springer.com/article/10.1007/s10664-025-10686-2) (Jun 2025)
- **Key finding**: Combining 4 individual tools achieved F1-score of 0.91 for microservice architecture recovery.
- **Relevance**: Supports the idea that **no single tool is sufficient** — combining approaches yields the best results. Integration strategy for Archy.

---

## 3. Gap Analysis: What No Tool Does Today

| Capability | Graphify | Codebase-Memory | dependency-cruiser | Axivion | Structurizr | **Archy (proposed)** |
|---|---|---|---|---|---|---|
| Auto-extract from source | ✅ | ✅ | ✅ | ❌ (manual model) | ❌ (manual DSL) | ✅ |
| Deterministic output | ✅ (code only) | ✅ | ✅ | ✅ | ✅ | ✅ |
| Git-diffable versioned artifact | ✅ (graph.json) | ❌ | ❌ | ✅ (Python model) | ✅ (DSL) | ✅ |
| In-code semantic annotations | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| CI enforcement / gate | ❌ | ❌ | ✅ (JS only) | ✅ (enterprise) | ❌ | ✅ |
| Multi-language | ✅ (33 langs) | ✅ (158 langs) | ❌ (JS/TS) | Partial (C/C++/Java/C#) | N/A | ✅ (TS+Python MVP) |
| Architecture diff (`archy diff`) | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ |
| External boundary declarations | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Staleness detection | ❌ | N/A (runtime) | ❌ | ✅ | ❌ | ✅ |
| Mermaid/D2 rendering | ❌ (HTML only) | ❌ | SVG | Commercial | ✅ | ✅ |
| AI context optimized | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |

**The unique combination Archy offers that nobody else does:**
1. **Bottom-up extraction** (auto from code) + **semantic enrichment** (in-code `@archy` annotations) + **CI enforcement** (archy check) + **versioned diffable artifact** (archy.json in git)
2. In-code annotations that travel with the code — nobody does this

---

## 4. Integration Possibilities

### Option A: Archy as a standalone tool (current design)
- Build everything from scratch
- Full control over the pipeline
- Highest effort, most complete vision
- Risk: competing with Graphify (63K stars) on the extraction side

### Option B: Archy as a layer on top of Graphify
- Use Graphify's `graph.json` as input instead of doing tree-sitter extraction ourselves
- Add: annotation system, CI enforcement, diffing, Mermaid rendering, external declarations
- Pro: Skip the hardest part (tree-sitter extraction, import resolution)
- Con: Dependency on Graphify's output format. Graphify uses LLM for some enrichment (non-deterministic). Graphify is Python-based; Archy is TypeScript.
- **Feasibility**: Moderate. Would need to parse Graphify's JSON output and build enforcement/annotation layers on top.

### Option C: Archy as a layer on top of Codebase-Memory
- Use Codebase-Memory's knowledge graph as the structural layer
- Add: annotation system, CI enforcement, JSON artifact export, diffing, Mermaid rendering
- Pro: Codebase-Memory is fully deterministic (no LLM), has 158 language support, blazing fast
- Con: Codebase-Memory is a C binary with SQLite storage — would need to shell out or use MCP protocol. Its graph schema may not match Archy's needs.
- **Feasibility**: Low-moderate. Protocol mismatch (MCP runtime vs. CLI artifact generation). Would require significant adapter work.

### Option D: Archy focuses exclusively on what's unique
- Skip competing on the extraction layer entirely
- Build only: annotation system (`@archy` comments), CI enforcement engine, architecture diffing, and a thin bridge to import structural data from any source (Graphify, Codebase-Memory, or its own lightweight parser)
- **This makes Archy an "architecture governance tool" rather than an "architecture extraction tool"**
- Pro: Smallest scope, most differentiated, avoids the hardest technical problems (tree-sitter extraction, import resolution)
- Con: Depends on external tools for the structural layer. May feel incomplete as a standalone tool.

---

## 5. Feasibility Assessment

### What's proven to work:
- Tree-sitter extraction of code structure (validated by Graphify, Codebase-Memory, SSAR, multiple papers)
- Deterministic JSON artifacts in git (validated by RIG/SPADE, Graphify's graph.json)
- CI enforcement of architectural rules (validated by dependency-cruiser, Axivion)
- LLMs are NOT reliable for deterministic architecture generation (validated by [arxiv:2603.21178](https://arxiv.org/abs/2603.21178) — 22.6% clarity failure rate)

### What's genuinely novel about Archy:
- **In-code `@archy` annotations** — nobody does this. The closest analog is Structurizr's DSL, but that's a separate file, not in-code.
- **Combined bottom-up extraction + semantic annotations + CI enforcement** — this specific combination doesn't exist.
- **Architecture diffing** as a first-class feature — Axivion has it but is enterprise/commercial.

### Technical risks:
- **Tree-sitter call graph extraction**: 70-80% accuracy for dynamic languages. Well-understood limitation.
- **Import resolution**: rabbit hole. V1 should handle only relative imports and tsconfig paths.
- **Adoption**: Graphify has 63K stars. Competing on the same extraction territory would be very difficult.

---

## 6. Recommendations

### Strongest path: Option D (governance-focused) + lightweight extraction

Build Archy as an **architecture governance CLI** with:
1. A lightweight built-in extractor (tree-sitter, TS+Python) for standalone use
2. Import bridges for Graphify/Codebase-Memory graph data
3. The unique layers: `@archy` annotations, CI enforcement (`archy check`), architecture diffing (`archy diff`), external boundary declarations
4. Mermaid rendering for visualization

This approach:
- Avoids competing directly with Graphify (63K stars) on extraction
- Focuses on what's genuinely unique and validated by research
- Keeps the standalone experience complete (built-in extractor works)
- Allows integration with best-in-class tools for extraction
- Reduces scope significantly (skip the import resolution rabbit hole for integrated mode)

---

## Sources

### Tools
- [Graphify](https://github.com/safishamsi/graphify) — 63K+ stars, YC S26
- [Codebase-Memory](https://github.com/DeusData/codebase-memory-mcp) — 14.5K stars
- [Structurizr](https://structurizr.com/) — C4 model reference implementation
- [Axivion Suite](https://www.qt.io/quality-assurance/axivion-architecture-verification) — Enterprise AaC verification
- [dependency-cruiser](https://github.com/sverweij/dependency-cruiser) — JS/TS dependency validation
- [skott](https://github.com/antoine-coulon/skott) — JS/TS dependency graph
- [Swark](https://github.com/swark-io/swark) — LLM architecture diagrams
- [Drift](https://github.com/sauremilk/drift) — Architecture erosion detection
- [RIG/SPADE](https://github.com/Greenfuze/Spade) — Deterministic build/test architecture

### Academic Papers
- [Codebase-Memory](https://arxiv.org/abs/2603.27277) — Tree-sitter knowledge graphs via MCP
- [Repository Intelligence Graph](https://arxiv.org/abs/2601.10112) — Deterministic architectural map for LLM assistants
- [SSAR](https://conf.researchr.org/details/icse-2026/icse-2026-research-track/221/) — Architecture recovery (ICSE 2026)
- [CIAO](https://arxiv.org/abs/2604.08293) — Automated architecture documentation with LLMs
- [LLM Architecture View Generation](https://arxiv.org/abs/2603.21178) — Empirical evaluation of LLM architecture generation
- [ArchAgent](https://arxiv.org/abs/2601.13007) — Scalable legacy architecture recovery
- [RAD-AI](https://arxiv.org/abs/2603.28735) — Architecture documentation for AI ecosystems (IEEE ICSA 2026)
- [Codified Context](https://arxiv.org/abs/2602.20478) — Infrastructure for AI agents in complex codebases
- [Static Analysis Architecture Recovery Comparison](https://link.springer.com/article/10.1007/s10664-025-10686-2) — Peer-reviewed, Empirical Software Engineering (Springer)
- [Semantic-Enhanced Architecture Recovery with LLMs](https://conf.researchr.org/details/icse-2026/icse-2026-research-track/72/) — ICSE 2026
