---
# Repo Mind Light - Shared Workflow
# Reusable Repo Mind Light integration for gh-aw workflows.
#
# This shared workflow bundles four concerns behind one local import:
# 1. incremental index preparation in a dedicated job
# 2. artifact handoff into the agent job
# 3. MCP server startup, readiness checks, and cleanup
# 4. prompt guidance telling the agent to use Repo Mind Light first

import-schema:
  config-yaml:
    type: string
    required: true
    description: >
      Full Repo Mind Light YAML configuration content. The workflow writes this
      to .repo-mind-light.config.yml in both the prep job and the agent job.
  image:
    type: string
    required: false
    description: >
      Repo Mind Light container image reference. Defaults to a pinned published
      Repo Mind Light image.
  cache-prefix:
    type: string
    required: false
    description: >
      Prefix used when deriving refreshed index cache keys.
  cache-restore-key:
    type: string
    required: false
    description: >
      Primary restore key used for the index cache restore step.
  container-name:
    type: string
    required: false
    description: >
      Docker container name used for the agent-job Repo Mind Light MCP server.
  artifact-name:
    type: string
    required: false
    description: >
      Base artifact name used for the prepared Repo Mind Light index. gh-aw's
      activation artifact prefix is added automatically to avoid collisions.

env:
  REPO_MIND_LIGHT_CACHE_PREFIX: ${{ github.aw.import-inputs.cache-prefix || 'repo-mind-light-index-' }}
  REPO_MIND_LIGHT_CACHE_RESTORE_KEY: ${{ github.aw.import-inputs.cache-restore-key || 'repo-mind-light-index-restore' }}
  REPO_MIND_LIGHT_CONTAINER_NAME: ${{ github.aw.import-inputs.container-name || 'repo-mind-light-mcp' }}
  REPO_MIND_LIGHT_ARTIFACT_NAME: ${{ github.aw.import-inputs.artifact-name || 'repo-mind-light-index' }}
  REPO_MIND_LIGHT_IMAGE: ${{ github.aw.import-inputs.image || 'ghcr.io/githubnext/repo-mind-light:v0.7.0@sha256:066f2b3646d238a865cd080234001aa1febec8a32970aa7cea84efc688135081' }}
  REPO_MIND_LIGHT_CONFIG_YAML: |
    ${{ github.aw.import-inputs.config-yaml }}

jobs:
  repo-mind-light-prep:
    name: Prepare Repo Mind Light index
    runs-on: ubuntu-latest
    needs: [activation]
    permissions:
      contents: read
      issues: read
      pull-requests: read
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6.0.2

      - name: Write Repo Mind Light config
        run: |
          printf '%s\n' "$REPO_MIND_LIGHT_CONFIG_YAML" > .repo-mind-light.config.yml

      - name: Restore Repo Mind Light index cache
        uses: actions/cache/restore@v5.0.5
        with:
          path: .index-store
          key: ${{ env.REPO_MIND_LIGHT_CACHE_RESTORE_KEY }}
          restore-keys: |
            ${{ env.REPO_MIND_LIGHT_CACHE_PREFIX }}

      - name: Ensure index directory exists
        run: |
          mkdir -p .index-store
          chmod -R a+rwX .index-store

      - name: Pull Repo Mind Light image
        run: docker pull "$REPO_MIND_LIGHT_IMAGE"

      - name: Run incremental indexing
        id: incremental-index
        env:
          COPILOT_TOKEN: ${{ secrets.COPILOT_GITHUB_TOKEN }}
          GITHUB_TOKEN: ${{ github.token }}
          REPO_MIND_LIGHT_CONFIG: /tmp/repo-mind-light.config.yml
        run: |
          set -euo pipefail
          result_json_dir="$RUNNER_TEMP/repo-mind-light-index"
          result_json_path="$result_json_dir/result.json"

          test -n "$COPILOT_TOKEN" || {
            echo "COPILOT_GITHUB_TOKEN secret is required" >&2
            exit 1
          }

          rm -rf "$result_json_dir"
          mkdir -p "$result_json_dir"
          chmod -R a+rwX "$result_json_dir"

          docker run --rm \
            -v "$PWD/.index-store:/var/lib/repo-mind-light/index" \
            -v "$result_json_dir:/tmp/repo-mind-light-result" \
            -v "$PWD/.repo-mind-light.config.yml:/tmp/repo-mind-light.config.yml:ro" \
            -e COPILOT_TOKEN \
            -e GITHUB_TOKEN \
            -e REPO_MIND_LIGHT_CONFIG \
            "$REPO_MIND_LIGHT_IMAGE" \
            index --result-json /tmp/repo-mind-light-result/result.json

          test -f "$result_json_path" || {
            echo "Repo Mind Light did not write the structured result JSON" >&2
            exit 1
          }

          refreshed="$(jq -r '.refreshed | if type == "boolean" then . else error("missing refreshed") end' "$result_json_path")"
          echo "changed=${refreshed}" >> "$GITHUB_OUTPUT"

          if test "$refreshed" = 'true'; then
            cache_key="$(jq -re --arg prefix "$REPO_MIND_LIGHT_CACHE_PREFIX" '
              .last_refreshed_at
              | if type == "string" and length > 0
                then $prefix + (split("T")[0])
                else error("missing last_refreshed_at")
                end
            ' "$result_json_path")"
            echo "cache_key=${cache_key}" >> "$GITHUB_OUTPUT"
          fi

      - name: Save refreshed Repo Mind Light index cache
        if: steps.incremental-index.outputs.changed == 'true'
        uses: actions/cache/save@v5.0.5
        with:
          path: .index-store
          key: ${{ steps.incremental-index.outputs.cache_key }}

      - name: Upload Repo Mind Light index artifact
        uses: actions/upload-artifact@v7.0.1
        with:
          name: ${{ needs.activation.outputs.artifact_prefix }}${{ env.REPO_MIND_LIGHT_ARTIFACT_NAME }}
          path: .index-store
          if-no-files-found: error
          include-hidden-files: true
          retention-days: '1'

mcp-servers:
  repo-mind:
    url: http://127.0.0.1:8000/mcp
    allowed:
      - query

tools:
  timeout: 600
  startup-timeout: 600

pre-agent-steps:
  - name: Download Repo Mind Light index artifact
    uses: actions/download-artifact@v8.0.1
    with:
      name: ${{ needs.activation.outputs.artifact_prefix }}${{ env.REPO_MIND_LIGHT_ARTIFACT_NAME }}
      path: .index-store

  - name: Ensure Repo Mind Light directories exist
    run: |
      mkdir -p .index-store /tmp/gh-aw/mcp-logs
      chmod -R a+rwX .index-store /tmp/gh-aw/mcp-logs

  - name: Write Repo Mind Light config
    run: |
      printf '%s\n' "$REPO_MIND_LIGHT_CONFIG_YAML" > .repo-mind-light.config.yml

  - name: Start Repo Mind Light MCP server
    env:
      COPILOT_TOKEN: ${{ secrets.COPILOT_GITHUB_TOKEN }}
      GITHUB_TOKEN: ${{ github.token }}
    run: |
      set -euo pipefail
      docker rm -f "$REPO_MIND_LIGHT_CONTAINER_NAME" >/dev/null 2>&1 || true
      docker pull "$REPO_MIND_LIGHT_IMAGE"

      docker run -d \
        --name "$REPO_MIND_LIGHT_CONTAINER_NAME" \
        -p 8000:8000 \
        -v "$PWD/.index-store:/var/lib/repo-mind-light/index" \
        -v "$PWD/.repo-mind-light.config.yml:/tmp/repo-mind-light.config.yml:ro" \
        -e COPILOT_TOKEN \
        -e GITHUB_TOKEN \
        -e REPO_MIND_LIGHT_CONFIG=/tmp/repo-mind-light.config.yml \
        "$REPO_MIND_LIGHT_IMAGE"

  - name: Capture initial Repo Mind Light logs
    run: |
      docker logs "$REPO_MIND_LIGHT_CONTAINER_NAME" \
        > /tmp/gh-aw/mcp-logs/repo-mind-light-startup.log 2>&1 || true

  - name: Wait for Repo Mind Light preload readiness
    timeout-minutes: 10
    run: |
      set -euo pipefail
      status_file='/tmp/gh-aw/mcp-logs/repo-mind-light-preload-status.json'
      status_url='http://127.0.0.1:8000/preload-status'

      for attempt in $(seq 1 120); do
        if ! docker ps --format '{{.Names}}' | grep -Fxq "$REPO_MIND_LIGHT_CONTAINER_NAME"; then
          echo 'Repo Mind Light MCP container exited before preload completed.' >&2
          docker logs "$REPO_MIND_LIGHT_CONTAINER_NAME" >&2 || true
          exit 1
        fi

        if response="$(curl --silent --fail --max-time 5 "$status_url" 2>/dev/null)"; then
          printf '%s\n' "$response" > "$status_file"
          ready="$(jq -r '.ready' "$status_file")"
          state="$(jq -r '.state' "$status_file")"
          echo "Repo Mind Light preload state: ${state}"

          if test "$ready" = 'true'; then
            echo 'Repo Mind Light preload completed.'
            exit 0
          fi

          if test "$state" = 'failed' || test "$state" = 'cancelled'; then
            echo "Repo Mind Light preload ended in unexpected state: ${state}" >&2
            docker logs "$REPO_MIND_LIGHT_CONTAINER_NAME" >&2 || true
            exit 1
          fi
        fi

        sleep 5
      done

      echo 'Timed out waiting for Repo Mind Light preload readiness.' >&2
      docker logs "$REPO_MIND_LIGHT_CONTAINER_NAME" >&2 || true
      exit 1

post-steps:
  - name: Capture Repo Mind Light logs
    if: always()
    run: |
      mkdir -p /tmp/gh-aw/mcp-logs
      if docker ps -a --format '{{.Names}}' | grep -Fxq "$REPO_MIND_LIGHT_CONTAINER_NAME"; then
        docker logs "$REPO_MIND_LIGHT_CONTAINER_NAME" \
          > /tmp/gh-aw/mcp-logs/repo-mind-light.log 2>&1 || true
      fi

  - name: Stop Repo Mind Light MCP server
    if: always()
    run: docker rm -f "$REPO_MIND_LIGHT_CONTAINER_NAME" || true
---

# Repo Mind Light

Use Repo Mind Light for repository-context retrieval before falling back to broad repo exploration.

## Required Usage Pattern

1. Make one focused `query` request first.
2. If the first result is sufficient, do not issue more Repo Mind Light queries.
3. Make at most one follow-up `query` only when the first result leaves a specific gap that matters to the task.
4. Use other tools only after Repo Mind Light has established the relevant local context.

## Notes

- Repo Mind Light is available at the `repo-mind` MCP server with the `query` tool.
- Startup and runtime logs are written under `/tmp/gh-aw/mcp-logs/` for debugging.
- This shared import assumes Repo Mind Light is enabled for the whole workflow. If a consumer needs event-specific gating, keep that decision in the consumer workflow or introduce a higher-level abstraction.
