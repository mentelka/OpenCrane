# Chunking Documentation for RAG

The Structure-Aware Hybrid Chunker processes documentation into semantically meaningful chunks optimized for Retrieval-Augmented Generation (RAG) systems.

> If you are authoring documentation that will be indexed by OpenCrane, see [docs/authoring-guide.md](authoring-guide.md) for how to structure your markdown so chunks are high quality and retrievable.

```bash
opencrane chunk --config yourproject.config:YourConfig
```

This command writes the chunks to `.opencrane/chunks.json`.

## How Document Boundaries and `source_url` are Handled

The chunker reads **both** the clean `llms-full.txt` bundle and its companion `llms.txt` index (see [Generating bundles](llms-generation.md)). Because `llms-full.txt` no longer carries URLs in its headings, each chunk's page `source_url` is recovered by joining the clean content back to the index:

1. **Split into source blocks** — `llms-full.txt` is split on `======` separators. The Nth block aligns with the Nth `## {source}` section in `llms.txt`.
2. **Split into pages** — within a source block, pages are split on the `<!-- opencrane:page -->` sentinel when present, otherwise on `#` H1 lines (fence-aware, so a `#` comment inside a fenced code block is not mistaken for a page boundary). The separator is a collision-proof HTML comment, not a dash rule, so a markdown thematic break (`---`, `-----`) in page content never mis-splits the page.
3. **Positional, title-validated join** — each page's URL is taken from the next entry in that source's index list, and validated against the page's leading H1 title (case-insensitive, whitespace-normalized). On a mismatch the join scans ahead within the same source's remaining entries for a matching title and realigns; if no match is found, `source_url` is left unset and a warning is logged.
4. **Assign to sub-chunks** — the resolved page URL is attached to every chunk produced from that page.

Because matching is scoped per source, duplicate page titles across sources are harmless — each page resolves to its own source's URL.

Example:
```
Input (llms-full.txt):          Input (llms.txt):
  # Setup Guide                   ## my-source
  Paragraph content...            - [Setup Guide](https://docs.example.com/setup)
  ## Prerequisites
  More content...

Output (.opencrane/chunks.json):
  {
    "chunk_type": "prose",
    "metadata": {
      "source_url": "https://docs.example.com/setup"
    },
    "content": "More content..."
  }
```

**Legacy fallback.** When no companion `llms.txt` is present (an older bundle that still has inline URL markers, or a bundle without an index), the chunker falls back to the previous marker-based behavior: `### https://...` boundary markers are treated as document boundaries and the source URL is extracted from the injected markers. This keeps old bundles resolvable until the next `build`.

This design ensures that each chunk maintains full context through its header hierarchy while document boundaries remain clear for processing.

## Tables

Every Markdown table becomes one `table_row` chunk per data row, rendered as natural-language `Column: value.` lines. Each row chunk carries the section heading and lead-in sentence from the source so it is independently retrievable. Rows self-organize via a shared `table_id` field and `sibling_ids` (full list_item parity) — no separate overview chunk is produced.

To fetch the full table from a row chunk, call `get_table_members(table_id=...)` using the `table_id` field in the row chunk's metadata. This returns all sibling row chunks ordered by `row_index`.

A table with no heading or lead-in sentence in the source still produces `table_row` chunks, but those chunks have no `breadcrumb_path` or `table_caption` metadata. Add a heading and a lead-in sentence in the source to make those chunks retrievable by semantic search.

The fixture pair `tests/fixtures/markdown_with_table.md` and `tests/fixtures/expected_table_chunks.json` is a generated baseline that shows the chunks produced for a representative table document. Regenerate `expected_table_chunks.json` with `ChunkSerializer.serialize_chunks` when chunking behavior changes intentionally.

## Architecture

The chunker uses a **Strategy Pattern** for extensible format support. It allows adding new formats (JSON, XML, etc.) without modifying core logic, using `ProcessingStrategy` interface. See `CodeChunkingStrategy` for reference implementation.

Available Strategies:
1. **YamlChunkingStrategy** - Detects and processes YAML content; delegates structured YAML specs (e.g., CRDs, OpenAPI) to tree walkers for structured chunking, falls back to generic yaml_content type for other YAML
2. **CodeChunkingStrategy** - Fenced code blocks with language detection; auto-detects CRDs and OpenAPI specifications in YAML code blocks and passes them to the built-in CRD and OpenAPI tree walkers. Fenced `yaml` and `yml` blocks go to `YamlChunkingStrategy` first, which runs every configured tree walker.
3. **TableChunkingStrategy** - Markdown tables; emits one `table_row` chunk per data row (natural-language rendered, self-linked via `table_id` + `sibling_ids`), and delegates non-table regions to the list and prose strategies
4. **ListChunkingStrategy** - Markdown lists; emits one chunk per list item
5. **ProseChunkingStrategy** - Markdown/text with hierarchical headers (fallback strategy)

Strategies are evaluated in order; first match wins.

## Data Models

### Chunk Structure

#### Top-Level Fields

##### `chunk_id` (string)
- Purpose: Unique identifier for each chunk
- Format: 64-character hex string
- Usage: Primary key in Milvus, cross-reference via `neighbor_chunks`, re-hydration, context expansion
- Generation: Hashed during chunking from the chunk content, source file, chunk type, and metadata, so the same input produces the same ID on every run

##### `chunk_type` (string, enum)
- Purpose: Identifies semantic type of chunk content
- Values:
  - `"prose"` - Markdown/text with hierarchical headers
  - `"code_snippet"` - Fenced code blocks with language detection
  - `"crd_definition"` - Kubernetes Custom Resource Definition properties
  - `"openapi_spec"` - OpenAPI specification elements
  - `"json_schema"` - JSON Schema definition properties
  - `"yaml_content"` - Generic YAML configuration (not a recognized structured format)
  - `"list_item"` - A single Markdown list item
  - `"table_row"` - A single Markdown table data row
- Usage: Route chunks to appropriate processing logic and templates

##### `content` (string or object)
- Purpose: Actual chunk content
- Format:
  - String for prose and code_snippet chunks
  - JSON object/array for structured schema chunks (parsed YAML)
- Important: YAML chunks store structured JSON, not strings, for semantic querying

##### `source_file` (string)
- Purpose: Relative path to source file from workspace root. Typically the llms-full file.
- Example: `"llmstxt/llms-full.txt"`
- Usage: Track chunk origins for debugging and filtering

##### `source_name` (string, optional)
- Purpose: Human-readable source identifier matching a path key in `.opencrane/config.yaml` sources.
- Example: `"MicrosoftDocs/microsoft-style-guide"`
- Resolved at chunking time by matching the chunk's `metadata.source_url` against the `url` / `docs_url` of each configured source (longest prefix wins).
- Value is `null` when no configured source matches the chunk's URL.
- Stored as a top-level Milvus field, enabling fast scalar filtering via the `source_names` parameter on `search_docs`.

##### `token_count` (integer)
- Purpose: Number of tokens in chunk content
- Encoding: `cl100k_base` (tiktoken) - used by GPT-4, GPT-3.5-turbo, text-embedding-ada-002
- Usage: Enforce chunk size limits, estimate context window usage

#### Metadata Object

All chunks include a `metadata` object with type-specific fields:

##### Universal Metadata (all chunk types)

###### `source_url` (string, URL, optional)
- Purpose: Link back to the specific documentation page the chunk came from
- Format: The page URL resolved via the `llms.txt` index join (see [How Document Boundaries and `source_url` are Handled](#how-document-boundaries-and-source_url-are-handled))
- Example: `"https://github.com/org/repo/blob/main/docs/configuration.md"` or a rendered docs-site page URL such as `"https://docs.example.com/configuration"`
- Usage: Provide users with source documentation link in RAG responses
- Present in: All chunk types when a page URL can be resolved

###### `original_format` (string, optional)
- Purpose: Original serialization format of content
- Value: `"yaml"` for YAML chunks
- Usage: Inform re-hydration process about expected output format
- Present in: YAML chunks (crd_definition, openapi_spec, json_schema)

###### `schema_type` (string, enum, optional)
- Purpose: High-level schema category for YAML content
- Values:
  - `"k8s_crd"` - Kubernetes Custom Resource Definition
  - `"openapi"` - OpenAPI specification
  - `"json_schema"` - JSON Schema
- Usage: Tell the schema formats apart
- Present in: YAML chunks (crd_definition, openapi_spec, json_schema)

###### `section_anchor` (string, optional)
- Purpose: In-page anchor slug of the section the chunk comes from. A direct section link is `{source_url}#{section_anchor}`.
- Example: `"prerequisites"`
- Present in: Prose, list item, and table row chunks that have a `source_url` and sit under a heading of level 2 or deeper. Structured YAML chunks have no anchor.
- Turn it off with `section_anchor_style: none` in `.opencrane/config.yaml`.

##### Prose Chunks (`chunk_type: "prose"`)

###### `breadcrumb_path` (string, optional)
- Purpose: Location of the chunk in the page
- Format: `page title > section`, where the section is the first heading of level 2 or deeper in the chunk. Only the page title when the chunk has no such heading.
- Example: `"Setup Guide > Prerequisites"`
- Present in: Prose chunks whose `source_url` is listed in the `llms.txt` index

##### Code Chunks (`chunk_type: "code_snippet"`)

###### `language` (string, required)
- Purpose: Programming language of code block
- Format: Language identifier from fenced code block (e.g., ```python)
- Examples: `"python"`, `"javascript"`, `"bash"`, `"yaml"`, `"json"`
- Usage: Syntax highlighting, language-specific filtering, code validation

###### `tab_value` (string, optional)
- Purpose: Tab identifier for parallel instructions
- Format: Value attribute from `<Tab>` component
- Usage: Filter code examples by implementation type
- Present in: Code blocks within `<Tabs>` components, when a custom chunking strategy sets it. The built-in strategies do not set it.

###### `tab_label` (string, optional)
- Purpose: Human-readable tab label
- Format: Label attribute from `<Tab>` component
- Usage: Display tab context in RAG responses
- Present in: Code blocks within `<Tabs>` components, when a custom chunking strategy sets it. The built-in strategies do not set it.

##### CRD Chunks (`chunk_type: "crd_definition"`)

**Chunking Strategy**: Property-based recursive chunking with an 800-token limit:
- Each `spec.properties` field is evaluated for token count
- If ≤ 800 tokens: chunk as-is with nested content
- If > 800 tokens AND has nested `properties`: recurse into nested properties, create separate chunks
- If > 800 tokens AND has `items.properties` (array): recurse into array item properties, create separate chunks
- If > 800 tokens AND no splittable structure: keep as single chunk (e.g., maps with `additionalProperties`)

Examples:
- `spec.replicas` (50 tokens) → single chunk with full definition
- `spec.config` (1200 tokens, nested properties) → split into `spec.config.database`, `spec.config.cache` chunks
- `spec.volumes` (900 tokens, array items with properties) → split into `spec.volumes.items.name`, `spec.volumes.items.path` chunks

**Multi-Version Support**: All CRD versions are processed. Each version generates separate chunks with `crd_version` metadata.

###### `breadcrumb_path` (string)
- Purpose: Exact location in YAML tree structure
- Format: Dot-separated path with array indices
- Example: `"spec.versions[0].schema.openAPIV3Schema.properties.spec.replicas"`
- Usage: Reconstruct YAML tree during re-hydration, precise property lookup

###### `logical_parent` (string)
- Purpose: Parent node in tree hierarchy
- Format: Dot-separated path of the parent node. It does not always equal `breadcrumb_path` without its final segment: a top-level property keeps the trailing `properties` segment.
- Example: `"spec.versions[0].schema.openAPIV3Schema.properties.spec.properties"` (parent of replicas)
- Usage: Group siblings, identify neighbor relationships

###### `neighbor_chunks` (array of chunk IDs)
- Purpose: Track sibling chunks at same tree level
- Format: Array of `chunk_id` values
- Definition: Neighbors = chunks sharing the same `logical_parent`, at any depth
- Example: `spec.replicas` and `spec.image` share a parent → Both reference each other's chunk IDs. A property that is split, such as `spec.config`, has no chunk of its own, so it is not a neighbor.
- Empty Array: No neighbors when only child under parent exists
- Usage: Context expansion - fetch neighbors to provide additional related information

###### `crd_kind` (string)
- Purpose: Kubernetes resource kind
- Example: `"MyResource"`
- Usage: Identify which CRD this chunk belongs to

###### `crd_api_version` (string)
- Purpose: Full API group and version
- Format: `{group}/{version}`
- Example: `"mygroup.example.com/v1"`
- Usage: Distinguish between different API versions of same resource

###### `crd_version` (string)
- Purpose: Version identifier (shorter form)
- Example: `"v1"`
- Usage: Quick version filtering, distinguish chunks from different CRD versions

###### `crd_property_path` (string)
- Purpose: Property path starting at `spec.`
- Format: Starts from `spec.`. For a recursively split property, includes the full nested path
- Example: `"spec.replicas"`, `"spec.config.database"`
- Usage: Display concise property paths in RAG responses, identify top-level property chunks

##### OpenAPI Chunks (`chunk_type: "openapi_spec"`)

**Chunking Strategy**: Element-based recursive chunking with an 800-token limit:
- Top-level elements (`info`, `security`, `tags`) → single chunks
- Servers → one chunk for each server
- Path operations (GET, POST, etc.) → separate chunks per method
- Component schemas:
  - If ≤ 800 tokens: chunk schema as-is
  - If > 800 tokens AND has nested `properties`: recurse into properties, create separate chunks
  - If > 800 tokens AND has `items.properties` (array schema): recurse into array item properties

Examples:
- `components.schemas.User` (200 tokens) → single chunk
- `components.schemas.Order` (1500 tokens, nested properties) → split into `Order.customer`, `Order.items` chunks
- Path operation `/users.post` with large request/response → single chunk (atomic operation unit)

###### `breadcrumb_path` (string)
- Purpose: Exact location in OpenAPI tree structure
- Format: Dot-separated path
- Examples:
  - `"info"`
  - `"paths./groups/{id}/access_requests.get"`
  - `"components.schemas.API_Entities_Badge"`
- Usage: Reconstruct OpenAPI spec during re-hydration

###### `logical_parent` (string)
- Purpose: Parent node in tree hierarchy
- Examples:
  - `"root"` (parent of info, security, tags)
  - `"servers"` (parent of each server chunk)
  - `"paths./groups/{id}/access_requests"` (parent of get, post operations)
  - `"components.schemas"` (parent of schema definitions)
- Usage: Group siblings, identify neighbor relationships

###### `neighbor_chunks` (array of chunk IDs)
- Purpose: Track sibling chunks at same tree level
- Format: Array of `chunk_id` values
- Examples:
  - `info`, `security`, `tags` (all have parent "root") → All are neighbors
  - Several server chunks (all have parent "servers") → They are neighbors
  - GET and POST at same path → They are neighbors
- Empty Array: Single child under parent (e.g., one server)
- Usage: Context expansion for related API elements

###### `openapi_version` (string)
- Purpose: OpenAPI specification version
- Examples: `"3.0.1"`, `"3.1.0"`
- Usage: Ensure compatibility with spec version

###### `openapi_element` (string, enum)
- Purpose: Top-level OpenAPI element type
- Values: `"info"`, `"servers"`, `"security"`, `"tags"`, `"paths"`, `"components"`
- Usage: Categorize chunks by OpenAPI structure

###### `server_url` (string, optional)
- Purpose: Server base URL
- Present in: Server chunks
- Example: `"https://www.example.com/api/v4"`
- Usage: Display available API endpoints

###### `endpoint_path` (string, optional)
- Purpose: API endpoint path
- Present in: Path operation chunks
- Example: `"/groups/{id}/access_requests"`
- Usage: Filter by endpoint, display in RAG responses

###### `http_method` (string, optional)
- Purpose: HTTP method for operation
- Present in: Path operation chunks
- Values: `"get"`, `"post"`, `"put"`, `"delete"`, `"patch"`, `"options"`, `"head"`, `"trace"`
- Usage: Filter by operation type, display method in responses

###### `schema_name` (string, optional)
- Purpose: Schema definition name
- Present in: Schema chunks
- Example: `"API_Entities_AccessRequester"`
- Usage: Reference schemas, resolve $ref links

###### `component_type` (string, optional)
- Purpose: Component category
- Present in: Component chunks
- Values: `"schemas"`, `"securitySchemes"`, `"responses"`, `"parameters"`, `"examples"`, `"requestBodies"`, `"headers"`, `"links"`, `"callbacks"`
- Usage: Organize components by type

###### `security_scheme_name` (string, optional)
- Purpose: Security scheme identifier
- Present in: Security scheme chunks
- Example: `"ApiKeyAuth"`
- Usage: Reference security schemes

##### JSON Schema Chunks (`chunk_type: "json_schema"`)

**Chunking Strategy**: Property and definition-based recursive chunking with an 800-token limit:
- Root metadata (title, description) → single chunk if present
- Properties:
  - If ≤ 800 tokens: chunk property as-is with nested content
  - If > 800 tokens AND has nested `properties`: recurse into nested properties
  - If > 800 tokens AND has `items.properties` (array): recurse into array item properties
- Definitions (`$defs` or `definitions`): same recursive logic as properties

Examples:
- `properties.username` (30 tokens) → single chunk
- `properties.config` (1000 tokens, nested) → split into `config.database`, `config.cache` chunks
- `$defs.address` (900 tokens, array items) → split into `address.items.street`, `address.items.city` chunks

## Extending with Custom Strategies

### Adding a New Processing Strategy

To add support for a new content type (e.g., JSON, XML, custom markdown components):

1. **Create strategy class**:

   ```python
   from pathlib import Path
   from typing import List
   from opencrane.rag.services.base_strategy import ProcessingStrategy
   from opencrane.rag.services.utils.chunk_id_generator import generate_unique_chunk_id
   from opencrane.shared.models.chunk import Chunk
   from opencrane.shared.utils.token_counter import get_token_count

   class MyCustomStrategy(ProcessingStrategy):
       def can_process(self, node) -> bool:
           """Check if this strategy can handle the node."""
           # Return True if this strategy handles the node
           return hasattr(node, 'text') and node.text.startswith('{{custom}}')

       def process(self, node, source_file: Path) -> List[Chunk]:
           """Process node into chunks."""
           chunks = []

           # Extract content
           content = node.text.removeprefix('{{custom}}').strip()

           # chunk_type must be one of the values in the chunk_type Literal in
           # opencrane/shared/models/chunk.py. Prose chunks accept
           # only these metadata keys: source_url, breadcrumb_path,
           # section_anchor, tab_value, tab_label, and key.
           metadata = {}

           chunk = Chunk(
               chunk_id=generate_unique_chunk_id(content, str(source_file), "prose", metadata),
               content=content,
               source_file=str(source_file),
               chunk_type="prose",
               metadata=metadata,
               token_count=get_token_count(content)
           )

           chunks.append(chunk)
           return chunks
   ```

2. **Register the strategy in your config subclass**:

   ```python
   # .opencrane/extensions.py
   from opencrane import OpenCraneConfig
   from opencrane.rag.services.yaml_chunker import YamlChunkingStrategy
   from opencrane.rag.services.code_chunker import CodeChunkingStrategy
   from opencrane.rag.services.table_chunker import TableChunkingStrategy
   from opencrane.rag.services.list_chunker import ListChunkingStrategy
   from opencrane.rag.services.prose_chunker import ProseChunkingStrategy
   from my_strategies import MyCustomStrategy

   class Config(OpenCraneConfig):
       chunking_strategies = [
           YamlChunkingStrategy(),   # Priority 1: YAML (delegates to tree walkers)
           MyCustomStrategy(),       # Priority 2: Your custom strategy
           CodeChunkingStrategy(),   # Priority 3: Fenced code blocks
           TableChunkingStrategy(),  # Priority 4: Markdown tables
           ListChunkingStrategy(),   # Priority 5: Markdown lists
           ProseChunkingStrategy(),  # Priority 6: Prose (fallback)
       ]
   ```

   OpenCrane loads `.opencrane/extensions.py` only when `.opencrane/config.yaml` sets `extensions: extensions.py`, and the class must be named `Config`. Otherwise, pass the class with `--config`.

   Put `my_strategies.py` in `.opencrane/`, next to `extensions.py`. When OpenCrane loads `extensions.py`, it adds that directory to the Python import path, so the import works.

   **Important**: Strategy order matters! First matching strategy wins. Place specific strategies before general ones.

### Adding a Tree Walker for YAML Standards

To add support for new YAML-based specifications (for example, AsyncAPI or AWS CloudFormation templates):

1. **Create tree walker class**:

   ```python
   from typing import List, Dict, Any
   from opencrane.walkers import YamlTreeWalker
   from opencrane.shared.models.chunk import Chunk
   from opencrane.shared.utils.token_counter import get_token_count
   import yaml

   class AsyncAPITreeWalker(YamlTreeWalker):
       """Walk AsyncAPI specification trees and generate element-based chunks."""

       @classmethod
       def can_handle(cls, doc: dict) -> bool:
           """Return True for AsyncAPI documents."""
           return "asyncapi" in doc and "channels" in doc

       def __init__(self, yaml_dict: Dict[str, Any], source_url: str,
                    source_file=None, original_yaml_file: str | None = None):
           """Initialize AsyncAPI tree walker."""
           super().__init__(yaml_dict, source_url, source_file, original_yaml_file)
           self.asyncapi_version = self._extract_asyncapi_version()

       def _extract_asyncapi_version(self) -> str:
           """Extract AsyncAPI version from spec."""
           return self.yaml_dict.get("asyncapi", "unknown")

       def walk(self) -> List[Chunk]:
           """Walk AsyncAPI tree and generate chunks for each element."""
           self.chunks = []

           # Process top-level elements
           if "info" in self.yaml_dict:
               self._process_info(self.yaml_dict["info"])

           if "channels" in self.yaml_dict:
               self._process_channels(self.yaml_dict["channels"])

           if "components" in self.yaml_dict:
               self._process_components(self.yaml_dict["components"])

           # Assign neighbor relationships
           self._assign_neighbor_relationships()

           return self.chunks

       def _process_info(self, info: Dict[str, Any]) -> None:
           """Process info section."""
           yaml_str = yaml.dump(info, default_flow_style=False)
           token_count = get_token_count(yaml_str)
           source_file = self.source_file or self.source_url
           metadata = {
               "source_url": self.source_url,
               "breadcrumb_path": "info",
               "logical_parent": "root",
               "neighbor_chunks": [],
               "original_format": "yaml",
               "schema_type": "asyncapi",
               "asyncapi_version": self.asyncapi_version,
               "asyncapi_element": "info"
           }

           chunk = Chunk(
               chunk_id=self._generate_chunk_id(
                   content=info,
                   chunk_type="asyncapi_spec",
                   metadata=metadata,
                   source_file=source_file
               ),
               content=info,
               source_file=source_file,
               chunk_type="asyncapi_spec",
               token_count=token_count,
               metadata=metadata
           )

           self.chunks.append(chunk)

       def _process_channels(self, channels: Dict[str, Any]) -> None:
           """Process channels section (message topics/queues)."""
           # Similar implementation to _process_info
           # Create chunks for each channel
           pass

       def _process_components(self, components: Dict[str, Any]) -> None:
           """Process reusable components (schemas, messages, etc)."""
           # Similar implementation
           pass

       def _assign_neighbor_relationships(self) -> None:
           """Identify and assign sibling relationships."""
           # Group chunks by logical_parent
           parent_groups: Dict[str, List[Chunk]] = {}
           for chunk in self.chunks:
               parent = chunk.metadata.get("logical_parent", "")
               if parent not in parent_groups:
                   parent_groups[parent] = []
               parent_groups[parent].append(chunk)

           # Set neighbors for each group
           for parent, siblings in parent_groups.items():
               if len(siblings) <= 1:
                   continue
               for chunk in siblings:
                   neighbor_ids = [s.chunk_id for s in siblings if s.chunk_id != chunk.chunk_id]
                   chunk.metadata["neighbor_chunks"] = neighbor_ids
   ```

2. **Register the walker in your config subclass**:

   ```python
   # .opencrane/extensions.py
   from opencrane import OpenCraneConfig
   from my_walkers import AsyncAPITreeWalker

   class Config(OpenCraneConfig):
       yaml_tree_walkers = [
           *OpenCraneConfig.yaml_tree_walkers,
           AsyncAPITreeWalker,
       ]
   ```

   As with strategies, OpenCrane loads this file only when `.opencrane/config.yaml` sets `extensions: extensions.py`.

   The walker runs on every YAML block that `YamlChunkingStrategy` handles, including fenced `yaml` and `yml` blocks and unlabeled blocks that parse as YAML.

3. **Add new chunk type to models**:

   Ensure your new chunk type is recognized. This step changes the OpenCrane package itself, not only your configuration:
   - Add `"asyncapi_spec"` to the `chunk_type` `Literal` in `opencrane/shared/models/chunk.py`. Without it, creating the chunk fails validation.
   - Document the new chunk type and its metadata fields in `docs/metadata-schema.md` and in `opencrane/mcp/metadata-schema.md`, which the `get_metadata_schema` MCP tool returns
   - To let `get_metadata_schema(chunk_type=...)` return the new section, add the chunk type to `_CHUNK_TYPE_SECTION_HEADINGS` in `opencrane/mcp/server.py`

### Key Concepts

- **Strategy Pattern**: Each strategy handles specific content types (YAML, code, prose)
- **Tree Walkers**: Specialized processors for structured YAML formats (CRDs, OpenAPI, AsyncAPI)
- **Priority Order**: Strategies execute in order; first match wins (specific before general)
- **Neighbor Relationships**: Tree walkers identify sibling chunks for context expansion
- **Chunk Types**: Use descriptive types (`asyncapi_spec`, `cloudformation_template`) for filtering and routing
