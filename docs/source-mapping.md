# Source Mapping Configuration

The source mapping file, `.opencrane/config.yaml` by default, is the control center for documentation processing in an OpenCrane project. It defines which documentation sources exist, where they come from, and how they should be processed.

## Purpose

The source mapping serves four critical functions:

1. **Documentation Discovery**: Tracks all documentation sources
2. **Output Control**: Determines which directories get `llms-full.txt` files generated
3. **Source Attribution**: Provides GitHub URLs for proper attribution in generated content
4. **Search Filtering**: Each path key becomes a chunk's `source_name`, exposing it as a `source_names` filter on the MCP `search_docs` tool so agents can scope queries to specific sources. The match is performed by prefix-matching a chunk's `source_url` against the entry's `url` and `docs_url`

## File Structure

Each entry in the mapping follows this structure:

```yaml
sources:
  my-project:                            # Path key: the source name
    url: https://github.com/my-org/my-project
    docs_path: docs                       # Path within the repository
    manual: false                         # Entry created/updated automatically vs manually maintained

  content-guidelines/writing:            # Local source (no fetching)
    local: true
```

Field Descriptions:

- Key (for example `my-project`): The source name. OpenCrane stores a fetched source in `.opencrane/sources/{KEY}`. For a local source, the key is the directory path relative to the workspace root
- url: Source repository URL for attribution and fetching. For a `type: llmstxt` source, the URL or local path of the `llms-full.txt` file
- type: `github` (the default) or `llmstxt`. An `llmstxt` source is an existing `llms-full.txt` bundle. The fetch writes it to `.opencrane/llmstxt/{KEY}/llms-full.txt`, and stale cleanup never removes it
- docs_url: Base URL of the published documentation site. When set, OpenCrane builds page URLs from it instead of from `url`
- docs_path: Path within the source repository where docs are located (empty string means root). If you leave it out, the fetch uses `docs`
- sha, tag, release, branch: Pin the fetch to one commit, tag, release, or branch. The fetch reads these fields only on entries with `manual: true`
- manual: `false` = entry auto-generated and updated during fetch, `true` = manually added and maintained mapping
- local: When `true`, the path key points to a local directory in the workspace — no fetching occurs and the pipeline reads directly from this path

## Configuring the Source Mapping File Path

Set the `MAPPING_FILE` environment variable to use a different file. The default is `.opencrane/config.yaml`. The CLI still reads the `extensions` and `section_anchor_style` keys from `.opencrane/config.yaml`, and `opencrane add` still writes new sources there.

## Automatic Updates During Fetching

> [!CAUTION]
> When auto-discovery no longer returns its repository (it lost the discovery topic, its topics could not be read, or the token cannot see it), `opencrane fetch` removes its entry from the source mapping file. It also deletes its local source directory and its generated `llmstxt/` output. To keep a source that does not carry the topic, set `manual: true` on its entry.

When documentation is fetched, the mapping file is automatically updated:

1. Script discovers repositories with the configured topic via GitHub API
2. For each repository, adds or updates an entry in the source mapping file
   - Sets `manual: false` for auto-discovered sources
   - Preserves `manual: true` entries (won't overwrite)
3. **Removes stale entries** - The fetch cleans up each auto-generated entry (`manual: false`) that auto-discovery no longer returns:
   - Removes entry from the source mapping file (only if `manual: false`)
   - Deletes local source directory (e.g., `.opencrane/sources/repo-name/`)
   - Deletes generated output directory (e.g., `.opencrane/llmstxt/repo-name/`)
   - The fetch never removes manual entries (`manual: true`), local entries (`local: true`), or `llmstxt` entries, and keeps their directories

This ensures the mapping always reflects the current state of available documentation sources and prevents stale entries from accumulating.

## Role in llms-full.txt Generation

The source mapping is the filter that controls which directories get `llms-full.txt` files generated. This prevents file explosion and gives explicit control over output structure.

### Path-Based Output Location

The key (relative path) in the mapping determines exactly where the generated file will be placed:

Mapping entry:
```yaml
my-project:
  url: https://github.com/my-org/my-project
  docs_path: docs
  manual: false
```

Generated output:
```
.opencrane/llmstxt/my-project/llms-full.txt
```

The output path mirrors the source path, making it easy to understand which source a file came from.

### Content Inclusion Rules

When generating `llms-full.txt` for a mapped path:

1. Start at mapped directory: `.opencrane/sources/my-project/`
2. Recursively collect ALL markdown: Includes `guides/`, `releases/`, `technical-reference/`, etc.
3. Generate single file: All content goes into one `llms-full.txt` at the mapped path
4. No subdirectory files: Even if `guides/` has markdown, no `llms-full.txt` is created there

Example:

```
Source structure:
  .opencrane/sources/my-project/
    README.md
    installation.md
    guides/
      mesh.md
    releases/
      1.0.0.md
    technical-reference/
      config.md

Generated output:
  .opencrane/llmstxt/my-project/llms-full.txt  ← Contains ALL 5 files

NOT generated:
  .opencrane/llmstxt/my-project/guides/llms-full.txt  ✗
  .opencrane/llmstxt/my-project/releases/llms-full.txt  ✗
```

## Building page URLs

The `url` and `docs_url` fields are used to construct per-page source URLs. These URLs are **not** injected into `llms-full.txt` headings — the bundle stays clean. Instead, they are written to the companion `llms.txt` index (one `- [title](page_url)` link per page) and joined back to each chunk during chunking. See [Generating bundles](llms-generation.md) and [Chunking](chunking.md).

**`url` (GitHub) URL construction:**
1. Take `url`: `https://github.com/my-org/my-project`
2. Add `/blob/main`. OpenCrane always links to the `main` branch, even when the fetch used a release or a pinned ref
3. Add `docs_path` if present: `/docs`
4. Add file's relative path from mapped directory: `/guides/setup.md`
5. Result: `https://github.com/my-org/my-project/blob/main/docs/guides/setup.md`

**`docs_url` (rendered docs site):** when set, `docs_url` takes precedence over `url` and produces a per-page URL on the published docs site (e.g. `https://docs.example.com/product/guides/setup`), so agents can point users to rendered docs rather than raw GitHub files.

This ensures every piece of content is traceable back to its source page.

Mapping:
```yaml
my-project:
  url: https://github.com/my-org/my-project
  docs_path: docs
```

File location: `.opencrane/sources/my-project/guides/setup.md`

Generated entry in `llms.txt`:
```markdown
## my-project
- [Setup Guide](https://github.com/my-org/my-project/blob/main/docs/guides/setup.md)
```

### `docs_url` for external `llmstxt` sources

For a pre-existing external `llmstxt` bundle, `docs_url` plays a different role depending on whether a companion `llms.txt` was fetched (see [Fetching](fetching.md)):

- **With a companion `llms.txt`**: the companion's real per-page URLs are used; `docs_url` is not needed for URL resolution.
- **Without a companion `llms.txt`**: the `llms` step synthesizes one index entry per H1 page, all carrying the base `docs_url` (the same URL repeated for every page). If no `docs_url` is set, those chunks get no `source_url`.

## Source Types

### Auto-Generated Entries (`manual: false`)
- External repositories discovered via GitHub API
- Mapping entry automatically created and updated during fetch operations
- Documentation fetched when `opencrane fetch` is run (locally or via CI/CD)

### Manual Entries (`manual: true`)
- External repositories manually added to the mapping
- Mapping entry manually created and maintained
- Not discovered or updated automatically, but still fetched from GitHub
- The fetch operation respects this flag and won't overwrite manually configured entries

### Local Entries (`local: true`)
- Documentation that already exists in the workspace (e.g., content in the same repo)
- No fetching occurs — the pipeline reads directly from the local path
- The path key is the local directory path relative to workspace root
- `url` is optional. If you set it, OpenCrane uses it to build the GitHub page URLs for the local files
- Never removed by stale cleanup
- Examples: Content guidelines, writing standards, templates that live alongside the OpenCrane project

```yaml
sources:
  # Local content — no fetching, reads from workspace root
  content-guidelines/writing:
    local: true

  # Remote content — fetched from GitHub
  my-project:
    url: https://github.com/my-org/my-project
    docs_path: docs
    manual: true
```
