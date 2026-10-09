# Generating Embeddings

After chunking documentation, generate vector embeddings for semantic search:

```bash
opencrane embed --config yourproject.config:YourConfig
```

## What happens

- Loads chunks from `.opencrane/chunks.json`
- Uses Nomic Embed model (nomic-ai/nomic-embed-text-v1.5)
- Generates vector embeddings (dimensions depend on model)
- Processes in batches to avoid memory issues
- Saves to `.opencrane/embeddings.json`

## Collection Schema

When loaded into Milvus, each chunk becomes a vector with fields:
- `chunk_id` (VARCHAR, primary key)
- `embedding` (FLOAT_VECTOR, dimensions match embedding model)
- `content` (VARCHAR, up to 65KB)
- `source_file` (VARCHAR)
- `source_name` (VARCHAR)
- `chunk_type` (VARCHAR: prose/code_snippet/crd_definition/openapi_spec/json_schema/yaml_content/list_item/table_row)
- `metadata_json` (VARCHAR)
- `token_count` (INT64)
- `line_start` (INT64)
- `list_id` (VARCHAR)
- `table_id` (VARCHAR)

The `index` step creates an `AUTOINDEX` vector index with the cosine metric and `INVERTED` indexes on `list_id` and `table_id`, then loads the collection into memory.
