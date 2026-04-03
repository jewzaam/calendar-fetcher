# NotebookLM Integration Notes

## Goal

Feed fetched Google Docs (Gemini meeting notes, transcripts) into NotebookLM as sources for AI-powered analysis and summarization.

## Current Approach: Consolidate into Google Doc

The `consolidate` subcommand merges fetched docs into a single Google Doc with one tab per source. This was designed as a workaround to reduce source count for NotebookLM (3 consolidated docs instead of 84 individual ones).

### Problems Encountered

- **Google Docs character limit**: 1.02M characters per doc ([source](https://support.google.com/drive/answer/37603?hl=en)). 30 days of Ansible Engineering calendar data produces ~6.6M chars across 131 unique docs. Even filtered to gemini notes + transcripts only (84 docs, 2.35M chars), that requires 3 Google Docs.
- **Base64 images in markdown exports**: Google Docs markdown export embeds images as base64 data URIs. One doc was 1.3MB with only 74KB of text. Fixed by stripping `data:image` content before insertion.
- **ARG_MAX / MAX_ARG_STRLEN**: The `gws` CLI takes `--json` as a command-line argument. Linux limits individual argv strings to 128KB (`MAX_ARG_STRLEN = 32 * PAGE_SIZE`). Fixed by chunking `insertText` calls to 100KB per chunk.
- **API quota**: Google Docs API has a per-minute write quota. Fixed with retry + backoff (5s, 10s, 15s steady).
- **Precondition failures**: Docs API returns "Precondition check failed" on concurrent edits (e.g., doc open in browser during consolidation). Fixed by catching errors per-doc and continuing.

## Better Approach: Direct NotebookLM API

NotebookLM has an official Enterprise API (alpha) that supports adding Google Docs as sources directly. This eliminates the Google Doc middleman entirely.

### NotebookLM Limits

| Limit | Free | Plus | Pro | Ultra |
|-------|------|------|-----|-------|
| Sources per notebook | 50 | 100 | 300 | 600 |
| Notebooks | 100 | 200 | 500 | — |
| Per-source limit | 500K words / 200MB | same | same | same |
| Notes per notebook | 1,000 | same | same | same |
| Daily chat queries | 50 | — | 500 | 5,000 |

Source: [NotebookLM FAQ](https://support.google.com/notebooklm/answer/16269187?hl=en), [Limits overview](https://elephas.app/blog/notebooklm-source-limits)

### API Details

- **Service**: `discoveryengine.googleapis.com`
- **Create notebook**: `notebooks.create`
- **Add sources**: `notebooks.sources.batchCreate` — supports Google Docs via Drive
- **Auth**: `gcloud auth print-access-token` (not OAuth client credentials like gws)
- **Docs**: [Create notebooks](https://docs.cloud.google.com/gemini/enterprise/notebooklm-enterprise/docs/api-notebooks), [Add sources](https://docs.cloud.google.com/gemini/enterprise/notebooklm-enterprise/docs/api-notebooks-sources)

### gws CLI Status

`gws` lists a "Discovery" service but reports "could not fetch API schema." The NotebookLM Enterprise API uses `discoveryengine.googleapis.com` which is not currently supported by `gws`. Integration would require either:

1. Adding `discoveryengine` support to `gws`
2. Calling the API directly via `curl` / `requests`
3. Using an unofficial client like [notebooklm-py](https://github.com/teng-lin/notebooklm-py) or [nblm-rs](https://github.com/K-dash/nblm-rs)

## Data Profile (Ansible Engineering, 30 days)

| Category | Unique docs | Characters (after image stripping) |
|----------|-------------|-----------------------------------|
| Gemini notes (`notes-by-gemini`) | 76 | 2,054,062 |
| Transcripts (`transcript`) | 8 | 296,038 |
| Human meeting notes | 47 | 4,258,939 |
| **Total** | **131** | **6,609,128** |

Gemini + transcripts alone (84 docs) fit within Plus tier's 100-source limit per notebook. Each doc is well under the 500K word per-source limit.

## Recommendation

The consolidate-to-Google-Doc approach works but fights multiple platform limits. Direct NotebookLM API integration is the cleaner path once `gws` supports `discoveryengine` or a direct API client is viable.
