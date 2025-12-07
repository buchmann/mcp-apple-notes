# MCP Apple Notes - Ollama Edition

> A fork of [RafalWilinski/mcp-apple-notes](https://github.com/RafalWilinski/mcp-apple-notes) that uses **Ollama** for local embeddings instead of HuggingFace Transformers.

## Why This Fork?

The original implementation uses `@huggingface/transformers` with the `Xenova/all-MiniLM-L6-v2` model for generating embeddings. On some macOS systems, this causes the **WindowServer to crash**, resulting in a system reboot when indexing notes.

This fork replaces the HuggingFace pipeline with calls to a local **Ollama** server, which:
- ✅ Eliminates WindowServer crashes
- ✅ Runs embeddings in a separate process (Ollama server)
- ✅ Supports multiple embedding models
- ✅ Is more stable on macOS with Apple Silicon

---

## Changes from Original

### 1. Embedding Provider: HuggingFace → Ollama

| Aspect | Original | This Fork |
|--------|----------|-----------|
| Library | `@huggingface/transformers` | Ollama REST API |
| Model | `Xenova/all-MiniLM-L6-v2` | `nomic-embed-text` (configurable) |
| Execution | In-process (Node.js) | External server (Ollama) |
| GPU Usage | Metal/GPU via transformers.js | Metal/GPU via Ollama |

### 2. Search Results: Fixed Vector Flooding

The original search returned raw vector arrays (768+ numbers per result), which overwhelmed Claude. This fork:
- Excludes vector data from search results
- Truncates content previews to 500 characters
- Adds relevance scores for better context

---

## File Changes

### `index.ts` - Complete Rewrite of Embedding Logic

#### Removed Dependencies
```typescript
// REMOVED - causes WindowServer crash on macOS
import { pipeline } from "@huggingface/transformers";
const extractor = await pipeline("feature-extraction", "Xenova/all-MiniLM-L6-v2");
```

#### Added: Ollama Configuration (Lines 1-25)
```typescript
// ============================================
// OLLAMA CONFIGURATION - Change model here if needed
// ============================================
const OLLAMA_BASE_URL = "http://localhost:11434";
const OLLAMA_EMBEDDING_MODEL = "nomic-embed-text"; // or "all-minilm" or "mxbai-embed-large"
```

#### Added: Ollama Embedding Function (Lines 27-50)
```typescript
/**
 * Calls Ollama's /api/embeddings endpoint to generate vector embeddings
 * @param text - The text to embed
 * @returns Promise<number[]> - Array of embedding values
 */
async function getOllamaEmbedding(text: string): Promise<number[]> {
  const response = await fetch(`${OLLAMA_BASE_URL}/api/embeddings`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: OLLAMA_EMBEDDING_MODEL,
      prompt: text,
    }),
  });

  if (!response.ok) {
    throw new Error(`Ollama API error: ${response.status} ${response.statusText}`);
  }

  const data = await response.json();
  return data.embedding;
}
```

#### Added: Dynamic Embedding Dimension Detection (Lines 52-62)
```typescript
/**
 * Detects the embedding dimension by making a test call to Ollama
 * Different models have different dimensions:
 * - nomic-embed-text: 768
 * - all-minilm: 384
 * - mxbai-embed-large: 1024
 */
async function getEmbeddingDimension(): Promise<number> {
  const testEmbedding = await getOllamaEmbedding("test");
  return testEmbedding.length;
}
```

#### Added: Custom LanceDB Embedding Function Class (Lines 70-95)
```typescript
/**
 * Custom embedding function for LanceDB that uses Ollama
 * Registered as "ollama" for use with LanceSchema
 */
@register("ollama")
class OllamaEmbeddingFunction extends EmbeddingFunction<string> {
  toJSON(): object {
    return {};
  }

  ndims(): number {
    return EMBEDDING_DIM;  // Detected dynamically on startup
  }

  embeddingDataType(): Float {
    return new Float32();
  }

  async computeSourceEmbeddings(data: string[]): Promise<number[][] | Float32Array[]> {
    const embeddings: number[][] = [];
    for (const text of data) {
      const embedding = await getOllamaEmbedding(text);
      embeddings.push(embedding);
    }
    return embeddings;
  }
}
```

#### Modified: `search-notes` Tool (Lines 180-220)
```typescript
// BEFORE (Original) - Returns vectors, crashes Claude
const results = await table.search(queryEmbedding).limit(10).toArray();
return { content: [{ type: "text", text: JSON.stringify(results, null, 2) }] };

// AFTER (This Fork) - Clean results, no vectors
const results = await table
  .search(queryEmbedding)
  .select(["title", "content"])  // Exclude vector field
  .limit(10)
  .toArray();

// Format with truncation and relevance scores
const formattedResults = results.map((r: any, i: number) => ({
  rank: i + 1,
  title: r.title,
  content_preview: r.content.length > 500 
    ? r.content.substring(0, 500) + "... [truncated - use get-note for full content]"
    : r.content,
  relevance: r._distance ? `${(1 - r._distance).toFixed(2)}` : "N/A"
}));
```

---

## Function Reference

### Core Functions

| Function | Purpose | Returns |
|----------|---------|---------|
| `getOllamaEmbedding(text)` | Generate embedding vector for text | `Promise<number[]>` |
| `getEmbeddingDimension()` | Detect model's embedding size | `Promise<number>` |

### MCP Tools (Available to Claude)

| Tool | Description | Input |
|------|-------------|-------|
| `list-notes` | List all note titles | None |
| `index-notes` | Index all notes for semantic search | None |
| `search-notes` | Semantic search across notes | `{ query: string }` |
| `get-note` | Get full content of a note | `{ title: string }` |
| `create-note` | Create a new note | `{ title: string, content: string }` |

---

## Setup Instructions

### Prerequisites

1. **Ollama** installed and running
   ```bash
   # Install Ollama (if not already)
   brew install ollama
   
   # Pull an embedding model
   ollama pull nomic-embed-text
   
   # Start Ollama server (or use the app)
   ollama serve
   ```

2. **Bun** runtime installed
   ```bash
   brew install bun
   ```

### Installation

```bash
# Clone your fork
git clone https://github.com/YOUR_USERNAME/mcp-apple-notes.git
cd mcp-apple-notes

# Install dependencies
bun install

# Test the server
bun run index.ts
```

### Claude Desktop Configuration

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "apple-notes": {
      "command": "/Users/YOUR_USERNAME/.bun/bin/bun",
      "args": ["/Users/YOUR_USERNAME/mcp-apple-notes/index.ts"]
    }
  }
}
```

Restart Claude Desktop after saving.

---

## Configuration Options

### Changing the Embedding Model

Edit the top of `index.ts`:

```typescript
const OLLAMA_EMBEDDING_MODEL = "nomic-embed-text";  // Default, 768 dimensions
```

**Supported models:**

| Model | Dimensions | Notes |
|-------|------------|-------|
| `nomic-embed-text` | 768 | Good balance of quality/speed |
| `all-minilm` | 384 | Fastest, lower quality |
| `mxbai-embed-large` | 1024 | Highest quality, slower |
| `snowflake-arctic-embed` | 1024 | Alternative high-quality |

After changing models, **re-index your notes**:
```
"Hey Claude, please re-index my Apple Notes"
```

### Changing Ollama Server URL

If Ollama runs on a different host/port:

```typescript
const OLLAMA_BASE_URL = "http://localhost:11434";  // Default
// Change to:
const OLLAMA_BASE_URL = "http://192.168.1.100:11434";  // Remote server
```

---

## Troubleshooting

### "Ollama API error: 404"
- Model not pulled: `ollama pull nomic-embed-text`

### "Connection refused"
- Ollama not running: `ollama serve`

### Search returns no results
- Notes not indexed: Ask Claude to "index my Apple Notes"

### View logs
```bash
tail -f ~/Library/Logs/Claude/mcp-server-apple-notes.log
```

---

## Data Storage

| Path | Purpose |
|------|---------|
| `~/.mcp-apple-notes/data/` | LanceDB vector database |

To reset and re-index:
```bash
rm -rf ~/.mcp-apple-notes/data
# Then ask Claude to index notes again
```

---

## Contributing

1. Fork this repository
2. Create a feature branch: `git checkout -b feature/my-change`
3. Commit changes: `git commit -am 'Add feature'`
4. Push to branch: `git push origin feature/my-change`
5. Submit a Pull Request

---

## Credits

- Original project: [RafalWilinski/mcp-apple-notes](https://github.com/RafalWilinski/mcp-apple-notes)
- Ollama modifications: [YOUR_NAME]
- Built with [Model Context Protocol](https://modelcontextprotocol.io)

---

## License

See original repository for license terms.
