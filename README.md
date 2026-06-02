# Awesome LLM Wiki [![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

> A curated list of foundational blueprints, functional frameworks, and technical guides for building compounding, AI-compiled knowledge bases.

Inspired by a paradigm shift in software development engineering, this architecture treats large language models not as ephemeral chat engines, but as stateful knowledge compilers. Instead of parsing fragmented document partitions dynamically at runtime via traditional RAG loops, these systems leverage autonomous agents to read static source files and systematically construct an interlinked, persistent Markdown knowledge topology.

---

## Contents

- [Foundations](#foundations)
- [Articles and Guides](#articles-and-guides)
- [Tools and Plugins](#tools-and-plugins)
- [Videos](#videos)

---

## Foundations

Foundational whitepapers, conceptual architectures, and structural blueprints outlining static agentic compilation.

- [Andrej Karpathy's LLM Wiki Gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) - The foundational idea file laying out the core pattern, operations, and architecture for compounding AI knowledge bases.
- [Farza's Personal Wiki Skill](https://gist.github.com/farzaa/c35ac0cfbeb957788650e36aabea836d) - A functional blueprint for implementing an LLM wiki compiler using Claude Code skills, including commands for ingestion, absorption, and automated cleanup.
- [LLM Wiki v2](https://gist.github.com/rohitg00/2067ab416f7bbe447c1977edaaa681e2) - An architectural extension of Karpathy's blueprint focused on scale, memory lifecycles, confidence decay, and typed knowledge graphs.

## Articles and Guides

Technical examinations, exhaustive architectural deep-dives, and detailed workflow overviews.

- [Commonplace](https://zby.github.io/commonplace/) - A technical framework establishing the theory of deploy-time learning for bounded AI observers, detailing semantic distillation operations, and providing automated CLI workspace management skills.
- [How to Build an LLM Knowledge Base (DAIR.AI)](https://academy.dair.ai/blog/how-to-build-an-llm-knowledge-base) - A practical workshop guide defining a standard local directory architecture and outlining repeatable agentic compiler patterns.
- [Karpathy's LLM Wiki: The Complete Guide (Agentpedia Codes)](https://antigravity.codes/blog/karpathy-llm-wiki-idea-file) - An exhaustive breakdown analyzing the three-layer architecture, comparing static compilation vs. traditional RAG, and detailing prompt configurations.
- [LLM Knowledge Bases (DAIR.AI)](https://academy.dair.ai/blog/llm-knowledge-bases-karpathy) - A detailed breakdown of the four-phase compilation pipeline (Ingest, Compile, Query, Lint) with architecture diagrams and implementation workflows.
- [Self-Authoring LLM Knowledge Bases](https://www.ricardodecal.com/projects/self-authoring-llm-knowledge-base/) - A technical conceptualization extending the compilation loop to live developer conversations, transforming ephemeral terminal and editor interactions into structured, persistent memory.
- [What Karpathy's LLM Wiki is Missing (And How to Fix It)](https://dev.to/penfieldlabs/what-karpathys-llm-wiki-is-missing-and-how-to-fix-it-1988) (Penfield Labs) - A deep architectural critique outlining solutions for token scaling limits in file-based context stores, featuring code patterns for semantic deduplication and pre-commit syntax hooks to protect structural integrity.

## Tools and Plugins

Software utilities, automation scripts, and development plugins for structuring and maintaining LLM wikis.

- [Atomic](https://github.com/kenforthewin/atomic) - An open-source, self-hosted personal knowledge base built in Rust that transforms freeform markdown notes into a semantically linked graph, featuring asynchronous chunking pipelines via sqlite-vec, auto-generated tag wikis with inline citations, an integrated MCP server, and a force-directed canvas.
- [Beever Atlas](https://github.com/Beever-AI/beever-atlas) ([Website](https://docs.beever.ai/atlas)) - An open-source, self-hostable conversational knowledge compiler and MCP server that transforms Slack, Discord, and Teams chat streams into a structured Neo4j knowledge graph and an auto-generated Markdown wiki with granular permission mirroring.
- [ByteRover](https://github.com/campfirein/byterover-cli) ([Website](https://www.byterover.dev/)) - An open-source, file-based local memory engine and interactive CLI tool that compiles codebase interactions into a hierarchical Context Tree, featuring agent-native curation, an adaptive knowledge lifecycle layer, sub-100ms hybrid text retrieval, and multi-IDE MCP portability.
- [browzy.ai](https://github.com/VihariKanukollu/browzy.ai) - An open-source, self-hosted TypeScript knowledge compiler designed to ingest messy digital data streams and compile them into a structured, self-organizing personal memory layer.
- [Cabinet](https://runcabinet.com/) - A free, open-source, file-based AI knowledge workspace that implements Karpathy's compilation loop, featuring git-backed auto-commits, scheduled agent automation cron-jobs, an integrated browser terminal, and embedded HTML application injection.
- [claude-obsidian](https://github.com/AgriciDaniel/claude-obsidian) - An open-source Claude Code plugin and knowledge engine that builds compounding Obsidian vaults, featuring hot-cache context persistence, multi-agent batch ingestion, automated 8-category vault linting, and spatial canvas orchestration.
- [Engram](https://github.com/NoobAIDeveloper/engram) - An open-source Claude Code skill suite that captures digital touchpoints and social threads, automatically parsing and compiling them into an interlinked, structured Obsidian knowledge vault.
- [Enzyme](https://enzyme.garden/) - A local-first memory indexer that compiles folder structures, backlinks, and tags into pre-computed concept "catalysts," providing sub-millisecond local context lookups and automated skill integration for Claude Code and Codex.
- [GitNexus](https://github.com/abhigyanpatwari/GitNexus) ([Website](https://gitnexus.vercel.app/)) - A zero-server, client-side code intelligence engine that compiles entire repositories into a structured knowledge graph and automated markdown wiki, utilizing local WebAssembly databases and an MCP server to provide deep architectural awareness to coding agents.
- [Graphify](https://github.com/safishamsi/graphify) - A multi-modal knowledge graph compilation engine that handles codebase AST parsing via tree-sitter alongside media transcription, producing localized agentic subgraphs to yield up to a 71.5x token efficiency gain.
- [Klore](https://github.com/vbarsoum1/llm-wiki-compiler) - A Python-based CLI knowledge compiler that structures multi-source research inputs into markdown files, featuring native configuration rule injectors for Cursor, Windsurf, and Copilot, alongside a dedicated Claude Code slash-command plugin.
- [Link](https://github.com/gowtham0992/link) ([Website](https://gowtham0992.github.io/link/)) - An open-source local memory engine and MCP server for terminal agents that compiles assets into markdown vaults, featuring built-in graph visualizations, automated structural health healing, and rigorous local security sanitization.
- [Linkly AI](https://github.com/LinklyAI/linkly-ai-cli) ([Website](https://linkly.ai/)) - A lightweight local document search engine and MCP server that compiles filesystem data into an AI-ready context layer, featuring progressive outline indexing, fast multilingual fuzzy matching, and deep regex terminal grep filtering.
- [LLM Wikid](https://github.com/shannhk/llm-wikid) - A multi-phase shell compilation framework for Obsidian vaults that implements automatic inbound media extraction, programmatic categorization routing, and mandatory cognitive bias countermeasure modules.
- [LLM Wiki (Nash Su)](https://github.com/nashsu/llm_wiki) - A cross-platform Tauri desktop application that turns multi-format documents into interlinked markdown vaults, featuring two-step chain-of-thought ingestion, interactive Louvain community graphs, and an async human-in-the-loop review system.
- [LLM Wiki (nvk)](https://github.com/nvk/llm-wiki) - An open-source core engine and CLI toolkit implementing whole-topic archive lifecycle management, deep workspace linting with structural auto-repair capabilities, and platform-specific path environment diagnostics.
- [llmwiki (Atomic Memory)](https://github.com/atomicmemory/llm-wiki-compiler) - A TypeScript CLI tool and MCP server that compiles raw text into structured markdown wikis, featuring paragraph-level source provenance tracking, multi-provider model routing, and a rule-based workspace linter.
- [llmwiki (Lucas Astorian)](https://github.com/lucasastorian/llmwiki) - An open-source Python engine and local web dashboard that indexes directories into a local SQLite repository, serving a specialized MCP adapter to automate Claude-driven wiki compilation and citation tracking.
- [LLM Wiki (Dom Leca)](https://github.com/domleca/llm-wiki) ([Forum Post](https://forum.obsidian.md/t/new-plugin-llm-wiki-turn-your-vault-into-a-queryable-knowledge-base-privately/113223)) - A native Obsidian community plugin implementing Karpathy's compilation pattern locally via Ollama (Qwen 2.5 + Nomic Embed), featuring real-time event-driven background extraction, multi-modal hybrid search, and persistent natural language chat interfaces.
- [Memora](https://github.com/agentic-box/memora) - A lightweight open-source MCP memory server that decomposes markdown files into structural semantic fragments, featuring automated tool schema sanitization, real-time graph visualizations, and automated LLM-driven deduplication.
- [nohmitaina](https://nohmitaina.com/) - A local-first macOS desktop Markdown editor built to implement Karpathy's LLM Wiki pattern natively alongside Claude Code or Codex, featuring automated background concept extraction, cross-reference mapping, and workspace contradiction linting.
- [obsidian-knowledge](https://github.com/crypdick/obsidian-knowledge) - An open-source Python automation toolkit that tracks file configurations and packs directory trees within local Obsidian vaults into streamlined context frames optimized for terminal coding agents.
- [Obsidian Second Brain](https://github.com/eugeniughelbur/obsidian-second-brain) ([Deep Dive](https://theaioperator.io/p/i-rebuilt-karpathys-llm-wiki-heres)) - A powerful cross-CLI skill suite for Claude Code, Codex, and Gemini that updates, reconciles, and rewrites existing vault notes dynamically to enforce a compounding local knowledge graph, featuring 34 terminal commands and write-time document validation.
- [OpenKB](https://github.com/VectifyAI/OpenKB) ([Website](https://pageindex.ai/)) - An open-source Python CLI knowledge base framework that compiles multi-format documents into interlinked markdown vaults using a specialized tree-based index for vectorless long-document retrieval.
- [Pieces](https://pieces.app/) - A local-first developer context and snippet manager driven by an on-device Long-Term Memory (LTM) engine, exposing workflow history, auto-tagged codebases, and structural metadata to external agents via an integrated MCP server.
- [QMD](https://github.com/tobi/qmd) - A local-first, mini CLI search engine and MCP server for markdown knowledge bases that combines BM25 full-text filtering, vector semantic search, and on-device LLM re-ranking.
- [SwarmVault](https://github.com/swarmclawai/swarmvault) ([Website](https://www.swarmvault.ai/)) - A local-first RAG knowledge base compiler and MCP server that maps files into an interlinked Markdown wiki and SQLite-backed knowledge graph, featuring automated linting, local graph visualizations, and a compounding "file-back" exploratory architecture.
- [Tolaria](https://tolaria.md/) - An open-source, Git-first desktop markdown app and native MCP server engine built with Tauri and Rust, implementing structural file conventions, automatic AGENTS.md generation, and secure local file boundaries for agent processing.
- [Wiki Builder (DAIR.AI)](https://github.com/dair-ai/dair-academy-plugins/tree/main/plugins/wiki-builder) - An open-source Claude Code plugin path that automates directory scaffolding, handles multi-flavor workspace indexing, and leverages localized markdown configuration files to govern agent compilation boundaries.
- [Wikiwise](https://github.com/TristanH/wikiwise) ([Website](https://wiki-wise.com/)) - A native macOS Swift application that wraps markdown directories into a fully browsable personal wiki interface, featuring a file-watcher compilation engine, cross-link indexing graph panels, and an embedded agent shell panel.

## Videos

Visual walkthroughs, conceptual code execution guides, and theoretical video essays.

- [How To Build LLM Wiki In Obsidian?](https://www.youtube.com/watch?v=QbjAQFJJyt0) (Wanderloots) - The definitive video tutorial mapping out the core 3-tier local memory architecture, showcasing how to build a file-based ingestion pipeline, implement a Git-backed maintenance loop, and deploy an agentic vault firewall wrapper.
- [Karpathy's LLM Wiki: What It Means & How to Build One](https://www.youtube.com/watch?v=zVEb19AwkqM) (Tonbi's AI Garage) - A practical video guide on bootstrapping an LLM Wiki from scratch inside Claude Code, demonstrating automated multi-agent ingestion loops, backfill routines for external web research, and visual chart integrations.
- [Why LLM Wiki? Future Of Knowledge For Agentic AI & Humans](https://www.youtube.com/watch?app=desktop&v=n4EVksU_EOs) (Wanderloots) - A visual guide explaining the mechanics of nodes, edges, and triples, the token-efficiency of GraphRAG over standard RAG, and a workflow for sandboxing human vs. agentic Obsidian vaults.

---

## Contributing

Contributions are welcome! Please ensure all pull requests strictly follow the formatting guidelines specified in the repository workflow files.
