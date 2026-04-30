# repo-mind-light-aw

`repo-mind-light-aw` is a companion repository for Repo Mind Light that packages the GitHub Agentic Workflows integration as a reusable shared workflow.

This repository intentionally contains workflow packaging and documentation only. It does not contain the Repo Mind Light implementation itself.

## What This Repository Provides

- A reusable gh-aw shared workflow for Repo Mind Light
- Repo Mind Light index preparation and artifact handoff
- MCP server startup, preload readiness checks, and cleanup
- Shared prompt guidance that tells agents to use Repo Mind Light first
- Shared Repo Mind Light tool timeout configuration

## Repository Layout

- `.github/workflows/shared/repo-mind-light.md`: the shared gh-aw import

## When To Use It

Use this shared workflow when you want a gh-aw workflow to query repository context through Repo Mind Light without duplicating the setup in every consumer workflow.

It is a good fit for workflows such as:

- issue triage
- incident or escalation context gathering
- PR review assistants that need repo-local historical context
- diagnostics workflows that should search repository-specific evidence before broader exploration

## Usage

Import the shared workflow from a consuming gh-aw workflow:

```yaml
imports:
  - uses: githubnext/repo-mind-light-aw/.github/workflows/shared/repo-mind-light.md@main
    with:
      config-yaml: |
        slug: ${{ github.repository }}
        store_path: /var/lib/repo-mind-light/index
        refresh_if_older_than: 1d
        query:
          preload_query_sources_on_startup: true
```

Then add consumer-specific permissions, GitHub tool settings, safe outputs, and task instructions in your own workflow.

## Inputs

The shared workflow accepts these inputs:

- `config-yaml`: full Repo Mind Light configuration content. Required.
- `image`: optional Repo Mind Light container image override.
- `cache-prefix`: optional prefix for refreshed cache keys.
- `cache-restore-key`: optional restore key for the index cache.
- `container-name`: optional Docker container name for the MCP server.
- `artifact-name`: optional artifact base name for the prepared index.

## Runtime Requirements

Consuming workflows should provide:

- `COPILOT_GITHUB_TOKEN` so Repo Mind Light can call Copilot model endpoints
- GitHub read permissions appropriate for the repository data being indexed
- Docker support on the runner used by the workflow

The shared workflow expects to use the public Repo Mind Light image published at `ghcr.io/githubnext/repo-mind-light` unless the `image` input overrides it.

## What The Shared Workflow Does

At a high level, the import does four things:

1. Refreshes or reuses a Repo Mind Light index in a dedicated prep job.
2. Uploads that index as an artifact and restores it into the agent job.
3. Starts the Repo Mind Light MCP server and waits for preload readiness.
4. Adds prompt instructions telling the agent to query Repo Mind Light before broader repo exploration.

## Validation

To validate the shared workflow locally:

```bash
gh aw compile .github/workflows/shared/repo-mind-light.md
```

To validate end-to-end behavior, import it into a consumer workflow and compile that consumer workflow as well.

## Relationship To Repo Mind Light

This repository is the gh-aw integration surface for Repo Mind Light.

The underlying Repo Mind Light implementation, indexing behavior, and container image live in the companion Repo Mind Light project. Keeping the integration layer separate makes it easier to reuse from workflows without exposing implementation-specific project contents in this repository.
