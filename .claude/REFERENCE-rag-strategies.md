# RAG Strategy Reference

Retrieval strategies mapped to architectural systems. Combine 3-5 per use case.

## Strategies

| Strategy | How it works | Maps to | Best for |
|----------|-------------|---------|----------|
| **Reranking** | Two-step: pull many chunks, cross-encoder selects most relevant | General retrieval quality | Almost everything — default include |
| **Agentic RAG** | Agent chooses search strategy per query (semantic search vs full doc read vs graph traversal) | Observe's context gathering (already chooses Read vs Grep vs WebSearch) | Varied query types against same knowledge base |
| **Knowledge graph search** | Entity-relationship traversal alongside vector similarity | Sovereign graph system; topology-protocol | Interconnected concepts, primitive-to-concept mapping |
| **Contextual retrieval** | LLM enriches each chunk with how it fits the broader document (Anthropic research) | Skill context layers (skills carry context about how they fit the system) | Large document collections where chunks lose meaning in isolation |
| **Hierarchical RAG** | Search small/precise, return big/contextual (parent-child chunk relationships) | Pyramid summaries (survey low-res, expand where signal) | Dream state, long documents, run archives |
| **Self-reflective RAG** | Grade search results, retry with refined query if quality is low | Quality gates + self-eval (validate output, retry if insufficient) | High-stakes retrieval where accuracy matters |
| **Query expansion** | LLM refines query to be more specific before search | Orient's path statement refinement | Vague or broad user queries |
| **Multi-query** | LLM generates multiple query variants, search in parallel | Stochastic consensus (multiple attempts, aggregate results) | Ambiguous queries where multiple angles help |
| **Context-aware chunking** | Embedding model finds natural document boundaries vs arbitrary splits | Compaction (preserve structure during compression) | Data preparation for any knowledge base |
| **Late chunking** | Embed full document first, then chunk the embeddings (preserves full context) | Compaction as quotient map (compress while preserving separability) | Documents where every chunk needs full document context |
| **Fine-tuned embeddings** | Domain-specific embedding model trained on your data (sentiment vs semantic) | Sovereign model registry (domain-specific models) | Specialized domains (legal, medical, domain-specific similarity) |
| **CAG (Cache Augmented Generation)** | Skip retrieval entirely — preload full knowledge base into KV cache; no chunking, no similarity search, just front-load everything | CLAUDE.md, skill definitions, dream state sections (we already do this) | Small, stable knowledge bases that fit in context; latency-critical; avoids all chunking/retrieval failure modes |

## Recommended combinations by system

| System | Recommended strategies |
|--------|----------------------|
| **Knowledge graph (topology-protocol)** | Knowledge graph search + reranking + hierarchical RAG |
| **Primitive whiteboard (cross-org retrieval)** | Agentic RAG + reranking + contextual retrieval |
| **Run archives (CBR/K-lines)** | Hierarchical RAG + reranking + self-reflective RAG |
| **Observe (context gathering)** | Agentic RAG + query expansion + reranking |
| **Sovereign data hub (general)** | Reranking + context-aware chunking + agentic RAG (starter combo) |

## Infrastructure

- **Storage:** PostgreSQL + PG Vector (already in sovereign data hub)
- **Chunking:** Dockling library for hybrid/context-aware chunking
- **Knowledge graphs:** Graphiti library
- **Reference:** Cole Medin's RAG strategies repo

## Security Notes

- **Vector embeddings are invertible.** Vectors are NOT hashes — they can be approximately inverted to recover original text with 90-100% accuracy using inversion + correction models (e.g., Vec2Text). PII scanners don't detect sensitive data in vector form. When building vector search, treat the vector store as containing plaintext-equivalent sensitive data.
- **RAG context leaks through conversation.** Longer conversations increase probability of extracting injected context. Fresh context windows per task (our shadow clone model) limits exposure surface.
- **System prompts are always extractable.** Every major model has had its system prompt stolen. Design CLAUDE.md and skill definitions with the assumption they are public. Never put secrets in system prompts.
- **Application-layer encryption** before data enters the vector store is more secure than transparent database encryption (which only protects powered-off servers). Consider for sovereign data hub.
- Reference: Patrick Walsh, "Exploiting Shadow Data in AI Models and Embeddings" (DEF CON)

## Notes

- Reranking is the default include for almost every implementation
- Context-aware chunking pays off at data preparation time for all strategies
- Fine-tuned embeddings deferred until sovereign model registry is active
- Late chunking is the most complex but preserves the most context — evaluate when document-level context loss becomes a measurable retrieval problem
