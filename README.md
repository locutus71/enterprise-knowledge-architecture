# Enterprise Knowledge Architecture

A self-improving knowledge architecture for AI agents. 3 memory layers, hybrid search, automated validation, multi-LLM orchestration. No framework, no vendor lock-in. SQLite, Python, open-source models.

**Paper:** [Epistemic Validation in AI Knowledge Systems](https://zenodo.org/records/) (Zenodo, 2026)

## Architecture Overview

| Component | Description | Document |
|---|---|---|
| 3-Layer Memory | Hot Memory, Deep Storage, Semantic Search | [architecture/3-layer-memory.md](architecture/3-layer-memory.md) |
| Hybrid Search | 70% Vector + 30% BM25, Temporal Decay | [architecture/hybrid-search.md](architecture/hybrid-search.md) |
| Hook System | 5-stage Session Lifecycle | [architecture/hook-system.md](architecture/hook-system.md) |
| Validation Layer | 6 Gates, Zero Trust on Agent Knowledge | [architecture/validation-layer.md](architecture/validation-layer.md) |
| Knowledge Loop | Self-Improving Knowledge Cycle | [architecture/knowledge-loop.md](architecture/knowledge-loop.md) |
| Multi-LLM | Auto-Router, Cross-LLM Benchmarking | [architecture/multi-llm-orchestration.md](architecture/multi-llm-orchestration.md) |
| Scope Isolation | Data Separation with Cross-Scope Coach | [architecture/scope-isolation.md](architecture/scope-isolation.md) |
| Timestamp Hook | Time Awareness for AI Agents | [architecture/timestamp-hook.md](architecture/timestamp-hook.md) |

## Interactive Documentation

- [deep-dive.html](deep-dive.html) — Full interactive architecture documentation (open in browser)

## Lessons Learned

- [architecture/lessons-learned.md](architecture/lessons-learned.md) — What went wrong and what we learned

## Priority Declaration

See [PRIORITY.md](PRIORITY.md) for authorship and public evidence.

## License

MIT License. See [LICENSE](LICENSE).

---
Author: Holger Woelfle | 2025-2026
