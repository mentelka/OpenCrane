# Fetch Documentation Sources

OpenCrane can automatically fetch documentation files from GitHub repositories. Documentation is fetched from the **latest release** of each repository, or from the default branch when a repository has no releases. An entry with `manual: true` can pin a `branch`, `tag`, `release`, or `sha` instead. OpenCrane ignores these fields on auto-discovered entries. The primary mechanism for fetching is through a scheduled GitHub Actions workflow that runs on a defined cadence to keep documentation up-to-date.

## GitHub Actions Workflow

A typical `update-docs.yml` workflow fetches documentation from one or more organizations or repositories:

- **Auto-discovery**: Repos tagged with a specific topic (e.g., `"documentation"`) are discovered automatically
- **Manual repositories**: Specific repos can be added in your source mapping config with `manual: true`

**Auto-discovery configuration**: The `AUTO_DISCOVERY_ORGS` environment variable controls which organizations have auto-discovery turned on. It takes a comma-separated list, for example `AUTO_DISCOVERY_ORGS=my-org,other-org`, and is empty by default. The `--org` flag also turns on auto-discovery for the organization it names. Each fetch discovers repositories in one organization only: the one that `--org` or the `ORG_NAME` environment variable names. Auto-discovery runs only if that organization is in `AUTO_DISCOVERY_ORGS`.

> [!CAUTION]
> If `ORG_NAME` names an organization that is not in `AUTO_DISCOVERY_ORGS` and you do not pass `--org`, the fetch discovers no repositories in it. It then deletes the local source directory and the `llmstxt/` output of every auto-discovered entry from that organization, and removes the entries from the source mapping config. Pass `--org`, or add the organization to `AUTO_DISCOVERY_ORGS`.

The workflow makes sure that your project always operates on the latest docs.

## Automatic Cleanup

> [!CAUTION]
> When a repository loses the discovery topic, `opencrane fetch` deletes its local source directory and its generated `llmstxt/` output. To keep a source that does not carry the topic, set `manual: true` on its entry.

The fetch process automatically cleans up stale documentation sources:

- **Stale Detection**: Repositories that **lose the discovery topic** are identified and removed
- **Mapping Cleanup**: Removes entries from your source mapping config (only auto-generated entries with `manual: false`)
- **Directory Cleanup**: Deletes both source directories and generated output (`llmstxt/`)
- **Manual Entry Protection**: Entries marked as `manual: true` are never automatically removed
- **Local Entry Protection**: Entries marked as `local: true` are never fetched or removed (they reference local filesystem paths)
- **Org Filtering**: The `--org` flag filters which auto-discovered repos are processed - auto-discovered entries from other orgs are skipped (not removed). Entries with `manual: true` are fetched whatever their org
- **Failure Protection**: Repos whose files **fail to download** (network errors, no files) are not removed
- **What counts as stale**: An auto-discovered repository is stale when the discovery no longer returns it. That happens when the repository lost the topic, when OpenCrane could not read its topics after retries, or when the repository was deleted, renamed, or is not visible to the token

This prevents accumulation of outdated documentation.

## Local Execution

Running locally is intended **only for testing purposes** or in exceptional cases where manual fetching is required (e.g., troubleshooting issues, one-off updates, or when the automated workflow fails). It requires setting up a GitHub token and Python environment locally.

### Fetch from a specific organization

> [!CAUTION]
> This command deletes the local source directory and the generated `llmstxt/` output of every auto-discovered repository in the organization that lost the discovery topic. To keep such a source, set `manual: true` on its entry before you run the command.

```bash
# Fetch from an organization (auto-discovers repos with the configured topic)
opencrane fetch --config yourproject.config:YourConfig --org my-org
```

### Fetch a single repository

Use `--source {PATH_KEY}` to restrict the fetch to one entry from your source mapping config, or pass a comma-separated list of path keys. The path key is the top-level key under `sources:` in that file, for example `my-repo`. `--repo` is an alias for `--source`. For entries with `manual: true`, the org filter does not apply, so no `--org` flag is needed. To refresh an auto-discovered entry, also pass `--org` with its organization.

> [!CAUTION]
> If a source you name is auto-discovered and lost the discovery topic, this command deletes its local source directory and its generated `llmstxt/` output. To keep the source, set `manual: true` on its entry before you run the command.

```bash
# Fetch only one manual entry, no --org needed
opencrane fetch --config yourproject.config:YourConfig --source my-repo
```

This is useful when you need to refresh a single source without re-fetching the entire set of repositories.

## Companion `llms.txt` for external `llmstxt` sources

When a source is added as a pre-existing `llmstxt` bundle (a URL or local path to an `llms-full.txt`), fetch also tries to retrieve the upstream **companion `llms.txt`** index next to it — the standard `llms.txt` file that carries real per-page URLs. It is saved alongside the bundle in `.opencrane/llmstxt/<name>/llms.txt`.

- **Remote sources**: the companion URL is derived by swapping a trailing `llms-full.txt` for `llms.txt`, or, when that does not apply, by appending `/llms.txt` to the source's `docs_url`. If the companion returns a 404 or any error, fetch proceeds without it — no hard failure.
- **Local sources**: a sibling `llms.txt` next to the source file is copied when present.

When a companion index is available, the `chunk` step uses its per-page URLs so each chunk carries its specific page `source_url`. When it is absent, the `llms` step synthesizes an index from the source's `docs_url` (the base URL, repeated for every page) — see [Source mapping](source-mapping.md). If the source has no `docs_url` either, its pages get no URL.

Companion entries may include the standard optional `: description` suffix after the link (`- [Title](url): some text`); it is parsed and ignored. GitBook-style companions often list source-file URLs ending in `.md` (or `/index.md`). When the source has a `docs_url` configured, those URLs are normalized to the rendered docs-site page (the extension is stripped and `/index.md` maps to its parent path), matching how `docs_url` sources resolve page URLs. Without a `docs_url`, companion URLs are used verbatim.

### Generate LLM bundles

```bash
opencrane llms --config yourproject.config:YourConfig
```
