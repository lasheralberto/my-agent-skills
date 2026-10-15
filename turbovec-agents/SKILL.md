---
name: turbovec-agents
description: >
  Build retrieval-augmented agents (RAG, semantic memory, tool/skill routing)
  using turbovec (https://github.com/RyanCodrai/turbovec) as the vector
  search backend instead of FAISS/Pinecone/Chroma. turbovec is a
  TurboQuant-based vector index that compresses embeddings 4-16x and needs no
  training phase, so it's well suited for agent memory that grows online
  (conversation history, tool docs, knowledge chunks). Use when the user
  mentions "turbovec", asks to add semantic memory/RAG to an agent, or wants
  a lightweight local vector index for an agent without standing up an
  external vector DB.
license: MIT
---

# turbovec agents

turbovec is a vector index (Python/Rust) built on Google's TurboQuant
algorithm: online ingestion (no training step), 4-16x compression vs raw
float32, SIMD-optimized search, and allowlist-filtered search. It is a
storage/search primitive, not an agent framework — this skill is about
wiring it into an agent's retrieval loop correctly.

## When to reach for turbovec instead of FAISS/Chroma/Pinecone

- Agent memory needs to grow continuously (chat turns, scraped docs) without
  re-training/re-indexing — turbovec's online `add()` fits this.
- Memory must stay local/private with no external service.
- Embedding count is large enough that RAM matters (10M docs: ~4GB vs 31GB
  float32) — relevant for long-running agents with big tool/doc corpora.
- Need per-search allowlist filtering (e.g. restrict retrieval to the
  current user's docs, or current conversation's docs) without a separate
  metadata filter pass.

If the corpus is small (a few thousand vectors) or already lives in a
hosted vector DB the user is happy with, don't migrate them to turbovec
just because it's available — that's premature.

## Install

```bash
pip install turbovec      # Python agents
cargo add turbovec        # Rust agents
```

## Core pattern for an agent's retrieval tool

1. Pick an embedding model and fix `dim` and `bit_width` up front — these
   are set at index construction and not easily changed later.
2. Use `IdMapIndex` (not bare `TurboQuantIndex`) whenever memory items need
   stable external IDs (message IDs, doc IDs) so you can delete/update
   individual entries (`remove(id)` is O(1)).
3. Wrap add/search behind a small `AgentMemory` class — the agent's tool
   should call `memory.search(query_text, k, allowlist=...)`, not touch the
   index directly. Embedding generation belongs in that wrapper too, so the
   index dimension and the embedding model stay coupled in one place.
4. Persist with `index.write(path)` after ingestion batches (or on a
   schedule) — there's no auto-persistence.
5. Use `allowlist` for scoping instead of filtering results after the fact
   in Python — it's designed to do this at search time without recall loss.

```python
from turbovec import IdMapIndex

class AgentMemory:
    def __init__(self, embed_fn, dim, bit_width=4, path=None):
        self.embed_fn = embed_fn
        self.index = IdMapIndex(dim=dim, bit_width=bit_width)
        if path:
            self.index.load(path)  # check actual API name before relying on this

    def add(self, doc_id, text):
        self.index.add_with_ids(self.embed_fn([text]), [doc_id])

    def search(self, query_text, k=5, allowlist=None):
        vec = self.embed_fn([query_text])[0]
        scores, ids = self.index.search(vec, k=k, allowlist=allowlist)
        return list(zip(ids, scores))
```

Expose `AgentMemory.search` as the agent's retrieval tool (function-calling
schema: `query: string`, optional `k`, optional `scope`/`allowlist`).

## Choosing bit_width

- `bit_width=4` is the default tradeoff point in turbovec's own benchmarks
  (16x compression on 1536-dim vectors, strong recall). Start here.
- Drop to lower bit widths only if memory pressure is the actual bottleneck
  and you've measured recall degradation is acceptable for the agent's use
  case (e.g. recall matters more for fact retrieval than for loose semantic
  chat-history recall).

## Gotchas

- This is a fast-moving young project — **verify exact method/class names
  (`write`/`load`/`add_with_ids`/etc.) against the installed version's
  README or docstrings before generating code**, don't assume the snippet
  above is byte-for-byte current.
- No training phase means no IVF-style clustering speedup curve to tune —
  don't port FAISS `nlist`/`nprobe` tuning habits over, they don't apply.
- SIMD kernels are platform-specific (NEON/AVX-512BW) — if deploying agents
  to mixed-arch infra (e.g. ARM Lambda vs x86 dev box), confirm the wheel
  for the target platform actually ships optimized kernels rather than a
  slow fallback.
