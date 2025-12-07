# Changelog

All notable changes to this fork are documented here.

## [1.0.0-ollama] - 2024-12-07

### Added
- **Ollama Integration**: New embedding provider using local Ollama server
  - `getOllamaEmbedding()` - Fetches embeddings from Ollama REST API
  - `getEmbeddingDimension()` - Auto-detects embedding dimensions on startup
  - `OllamaEmbeddingFunction` - LanceDB-compatible embedding class
  
- **Configuration Options** at top of `index.ts`:
  - `OLLAMA_BASE_URL` - Ollama server address (default: `http://localhost:11434`)
  - `OLLAMA_EMBEDDING_MODEL` - Model to use (default: `nomic-embed-text`)

- **Relevance Scores**: Search results now include similarity scores

### Changed
- **Search Results**: Now exclude vector arrays, only return title + content
- **Content Preview**: Search results truncated to 500 chars (use `get-note` for full)
- **Result Format**: Added `rank`, `content_preview`, and `relevance` fields

### Removed
- `@huggingface/transformers` dependency
- `Xenova/all-MiniLM-L6-v2` in-process model loading

### Fixed
- **WindowServer Crash**: Eliminated macOS crash caused by HuggingFace transformers GPU initialization
- **Claude Overflow**: Search no longer returns 768-dimensional vectors that overwhelm Claude

---

## Migration from Original

1. Install Ollama: `brew install ollama`
2. Pull model: `ollama pull nomic-embed-text`
3. Start Ollama: `ollama serve`
4. Replace `index.ts` with the Ollama version
5. Run `bun install` (no new deps needed)
6. Re-index notes in Claude: "Index my Apple Notes"
