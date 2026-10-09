# Generating Embeddings

After chunking documentation, generate vector embeddings for semantic search:

```bash
opencrane embed
```

## What happens

- Loads chunks from `.opencrane/chunks.json`
- Uses Nomic Embed model (nomic-ai/nomic-embed-text-v1.5) by default. To use a different model, set the `EMBEDDING_MODEL` environment variable
- Generates vector embeddings. The Milvus collection stores 768-dimension vectors, which matches the default model. A model with a different dimension produces vectors that the `index` step cannot load
- Processes in batches to avoid memory issues
- Saves to `.opencrane/embeddings.json`

## Collection Schema

When loaded into Milvus, each chunk becomes a vector with fields:
- `chunk_id` (VARCHAR, primary key, `max_length` 64)
- `embedding` (FLOAT_VECTOR, 768 dimensions)
- `content` (VARCHAR, `max_length` 65,535)
- `source_file` (VARCHAR, `max_length` 512)
- `source_name` (VARCHAR, `max_length` 256)
- `chunk_type` (VARCHAR, `max_length` 32: prose/code_snippet/crd_definition/openapi_spec/json_schema/yaml_content/list_item/table_row)
- `metadata_json` (VARCHAR, `max_length` 65,535)
- `token_count` (INT64)
- `line_start` (INT64)
- `list_id` (VARCHAR, `max_length` 64)
- `table_id` (VARCHAR, `max_length` 64)

The `index` step creates an `AUTOINDEX` vector index with the cosine metric and `INVERTED` indexes on `list_id` and `table_id`, then loads the collection into memory.
