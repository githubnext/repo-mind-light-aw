# repo-mind-light-aw

`repo-mind-light-aw` is the agentic workflow surface for Repo Mind Light. It packages the GitHub Agentic Workflows integration as a reusable shared workflow.

This repository intentionally contains workflow packaging and documentation only. It does not contain the Repo Mind Light implementation itself.

The content here is written for two audiences at the same time:

- humans who need to understand what the integration does and when to use it
- agents that need enough operational detail to compose or modify gh-aw workflows correctly

## What This Repository Provides

- A reusable gh-aw shared workflow for Repo Mind Light
- Repo Mind Light index preparation and artifact handoff
- MCP server startup, preload readiness checks, and cleanup
- Shared prompt guidance that tells agents to use Repo Mind Light first
- Shared Repo Mind Light tool timeout configuration

## What Repo Mind Light Does

Repo Mind Light provides holistic repository understanding over GitHub repository data.

It is designed to answer questions such as:

- what parts of the repository are relevant to this feature or problem
- what historical issues, pull requests, or discussions are related
- what architectural or implementation decisions shaped the current state
- what code areas are likely to matter for a given symptom or topic

At the workflow level, this integration exposes one MCP server named `repo-mind` with one allowed tool:

- `query`: takes a focused natural-language query and returns retrieved repository context grounded in the prepared Repo Mind Light index

The final answer can combine evidence from:

- indexed issues
- indexed pull requests
- indexed discussions
- indexed wiki pages (when wiki indexing is enabled)
- repository content already captured in the Repo Mind Light index
- GitHub code search results used to mine relevant code snippets

Repo Mind Light does not just retrieve raw matches. It also uses embeddings and a generative model to synthesize a final answer from the retrieved evidence.

The intended usage pattern is:

1. prepare or restore an index for the target repository
2. start the Repo Mind Light MCP server against that index
3. ask one focused `query`
4. ask at most one follow-up `query` if a specific gap remains
5. only then fall back to broader GitHub or repository exploration

## Repository Layout

- `.github/workflows/shared/repo-mind-light.md`: the shared gh-aw import
- `LICENSE`: MIT license for this repository

## When To Use It

Use this shared workflow when you want a gh-aw workflow to query repository context through Repo Mind Light without duplicating the setup in every consumer workflow.

It is a good fit for workflows such as:

- holistic repository understanding
- high-level architectural analysis
- historical investigation across issues, pull requests, discussions, and code
- workflows that need repository-specific context before taking another action
- workflows that need help locating relevant subsystems, past decisions, or strongly related code

It is less about narrow issue-routing automation and more about building a high-value repository context layer for an agent.

## Usage

Import the shared workflow from a consuming gh-aw workflow:

```yaml
imports:
  - uses: githubnext/repo-mind-light-aw/.github/workflows/shared/repo-mind-light.md@main
    with:
      config:
        yaml: |
          slug: ${{ github.repository }}
          store_path: /var/lib/repo-mind-light/index
          refresh_if_older_than: 1d
          query:
            preload_query_sources_on_startup: true
```

Then add consumer-specific permissions, GitHub tool settings, safe outputs, and task instructions in your own workflow.

For stability-sensitive consumers, prefer pinning the import to a commit SHA or release tag instead of `@main`.

## Cost And Model Considerations

Using Repo Mind Light adds extra cost beyond the base gh-aw workflow run.

That cost exists because Repo Mind Light:

- computes embedding vectors during indexing and refresh
- uses a generative model to produce the final `query` answer

This is why the shared workflow requires `COPILOT_GITHUB_TOKEN`.

Consumer workflows can still use another outer agent runtime such as Claude or Codex.

The important constraint is not the outer workflow engine. The important constraint is that Repo Mind Light itself depends on Copilot-backed model access for embeddings and answer generation.

So even when the surrounding workflow uses a non-Copilot engine, `COPILOT_GITHUB_TOKEN` still must be available to Repo Mind Light or indexing and query synthesis will not work.

## Inputs

The shared workflow accepts these inputs:

- `config`: Repo Mind Light configuration object. Required.
- `config.yaml`: full Repo Mind Light configuration content stored under the required `yaml` property.
- `image`: optional Repo Mind Light container image override.
- `cache-prefix`: optional prefix for refreshed cache keys.
- `cache-restore-key`: optional restore key for the index cache.
- `container-name`: optional Docker container name for the MCP server.
- `artifact-name`: optional artifact base name for the prepared index.
- `run-if`: optional GitHub Actions expression evaluated as the prep job condition. Defaults to `true`. Pass an event or label condition to skip Repo Mind Light for irrelevant triggers.

These inputs are intentionally narrow. Most behavioral tuning happens inside `config.yaml`, which is passed through directly to Repo Mind Light.

## Repo Mind Light Config Schema

The most important underlying Repo Mind Light configuration fields are:

```yaml
slug: owner/repo
store_path: /var/lib/repo-mind-light/index
chat_model: claude-sonnet-4.6
refresh_if_older_than: 1d

conversations:
  keep_count: 1000
  issue_state: all              # null | none | open | closed | all
  pr_state: all                 # null | none | open | closed | merged | all
  discussion_state: all         # null | none | all
  issue_labels: null
  pr_labels: null
  discussion_categories: null
  ignore_bot_authored: true

wiki: null                      # optional; set to {} or a config object to enable wiki indexing

query:
  preload_query_sources_on_startup: true
  graph_rag_zero:
    top_k_initial: 1000
    max_chunks: 200
    relative_dropoff_threshold: 0.1
    knn_neighbors: 5
    max_cluster_size: 10
  code_search:
    enabled: true
    limit: 50
```

Important schema notes for workflow authors and agents:

- `slug` is the only required Repo Mind Light field.
- `refresh_if_older_than` controls whether cached indexes are reused or refreshed.
- `conversations` controls issue, pull request, and discussion indexing. The legacy `indexing` key is still accepted by Repo Mind Light, but new workflows should use `conversations`.
- `issue_state` accepts `open`, `closed`, `all`, `none`, or `null`.
- `pr_state` accepts `open`, `closed`, `merged`, `all`, `none`, or `null`.
- `discussion_state` accepts `all`, `none`, or `null`.
- `issue_labels`, `pr_labels`, and `discussion_categories` are optional any-of filters.
- `ignore_bot_authored: true` is usually a good default for support and triage workflows.
- Wiki indexing is opt-in. `wiki: null` disables wiki indexing and wiki query sources.
- `wiki: {}` enables GitHub wiki indexing with Repo Mind Light's default path filters for common text formats and skipped sidebar/footer pages.
- `wiki.include_paths` keeps only wiki pages matching at least one glob, and `wiki.exclude_paths` skips pages matching any configured glob.
- File extension filtering should be expressed with `wiki.include_paths` patterns such as `*.md` and `*.markdown`.
- Wiki pages are refreshed incrementally and re-embedded only when their Git blob SHA changes.
- `query.preload_query_sources_on_startup: true` reduces first-query latency by warming preloadable sources after server startup.
- `query.code_search.enabled` lets Repo Mind Light use its code-search-backed retrieval path when available.

Example wiki indexing configuration:

```yaml
wiki:
  include_paths:
    - "*.md"
    - "*.markdown"
    - "*.rst"
    - "*.txt"
    - "*.textile"
    - "*.org"
  exclude_paths:
    - "_Sidebar.*"
    - "_Footer.*"
```

## Query Behavior

The `query` tool is meant for targeted repository-context retrieval, not for broad speculative searching.

Good query patterns include:

- an exact user-visible error message
- a feature or workflow name
- a suspected subsystem or ownership area
- a request for similar incidents or regressions
- a request for files, classes, or modules relevant to a concrete symptom

Agents should prefer precise, discriminating prompts over broad prompts such as "tell me about this repo".

## Retention, Caching, And Artifacts

This shared workflow has several storage behaviors that matter operationally:

- The prepared index is uploaded as a GitHub Actions artifact with `retention-days: 1`.
- The local working directory uses `.index-store` as the mounted index path during workflow execution.
- Refreshed cache entries are saved only when the Repo Mind Light indexer reports at least one refreshed source in its structured `--result-json` output.
- Cache keys are derived from the refresh date and are intentionally immutable.
- Cache eviction after that point is controlled by GitHub Actions cache retention policy, not by this repository.

This means the workflow gives consumers short-lived artifact handoff plus reusable cache restore behavior, but it does not implement custom cache pruning.

## Runtime Requirements

Consuming workflows should provide:

- `COPILOT_GITHUB_TOKEN` so Repo Mind Light can call Copilot model endpoints for embeddings and final answer generation
- GitHub read permissions appropriate for the repository data being indexed
- Docker support on the runner used by the workflow

The shared workflow expects to use the public Repo Mind Light image published at `ghcr.io/githubnext/repo-mind-light` unless the `image` input overrides it.

By default, the shared workflow tracks `ghcr.io/githubnext/repo-mind-light:latest` so it picks up the current published release. Consumers that need stricter reproducibility should override `image` with an explicit tag or digest.

Consumers that only need Repo Mind Light for some events can pass a GitHub Actions expression through `run-if`. The expression is applied to the Repo Mind Light prep job; when it is false, the dependent agent job is skipped too.

```yaml
imports:
  - uses: githubnext/repo-mind-light-aw/.github/workflows/shared/repo-mind-light.md@main
    with:
      run-if: >-
        github.event_name == 'workflow_dispatch' ||
        contains(github.event.issue.labels.*.name, 'support-escalation') ||
        contains(github.event.issue.labels.*.name, 'internal-feedback')
      config:
        yaml: |
          slug: ${{ github.repository }}
          store_path: /var/lib/repo-mind-light/index
          refresh_if_older_than: 1d
```

Consumers should also ensure:

- the runner can bind port `8000` for the Repo Mind Light MCP server
- `jq` and `curl` are available on the runner image used by gh-aw jobs
- enough disk space exists for the prepared index and temporary result files

## What The Shared Workflow Does

At a high level, the import does four things:

1. Refreshes or reuses a Repo Mind Light index in a dedicated prep job.
2. Uploads that index as an artifact and restores it into the agent job.
3. Starts the Repo Mind Light MCP server and waits for preload readiness.
4. Adds prompt instructions telling the agent to query Repo Mind Light before broader repo exploration.

More concretely:

- the prep job writes the config file, restores `.index-store`, and runs `repo-mind-light index --result-json ...`
- the structured JSON result determines whether any source refreshed and therefore whether a refreshed cache key should be saved
- the agent job downloads the prepared index artifact, starts the MCP server, and waits on `/preload-status`
- the import exposes only the `query` tool to the agent from the Repo Mind Light server
- the import sets shared tool timeout values of `600` seconds for startup and tool execution

## Operational Constraints

Agents and workflow authors should assume the following constraints:

- this repository contains integration logic, not the private or implementation-specific source for Repo Mind Light
- the shared workflow assumes Repo Mind Light is enabled for the whole consumer workflow unless the consumer passes `run-if`
- event-specific gating policy belongs in the consuming workflow and can be passed through the `run-if` input
- the import is designed for repository-scoped retrieval, not for cross-repository discovery by itself
- the MCP server startup path depends on a valid prepared index; if no index exists or indexing fails, the consumer workflow should fail fast

## Validation

To validate the shared workflow locally:

```bash
gh aw compile .github/workflows/shared/repo-mind-light.md
```

To validate end-to-end behavior, import it into a consumer workflow and compile that consumer workflow as well.

For real validation, also exercise:

- one run that reuses a cache
- one run that forces a refresh
- one query that succeeds quickly
- one query against a low-signal input so the consumer workflow can verify its fallback behavior

## Relationship To Repo Mind Light

This repository is the gh-aw integration surface for Repo Mind Light.

## License

MIT. See [LICENSE](./LICENSE).
