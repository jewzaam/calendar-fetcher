# Design: `consolidate` Subcommand

## Problem

NotebookLM works best when fed a single document containing all relevant content. Calendar-fetcher already downloads Google Docs (as markdown), Sheets (as CSV), and Slides (as PDF) from calendar event attachments. This subcommand takes those fetched Google Docs and consolidates them into a single output Google Doc with one tab per source document.

## Scope

**In scope:**
- Google Docs only (`application/vnd.google-apps.document`) — these are already exported as markdown (`.md`) by the `fetch` subcommand
- Plain text insertion into Google Doc tabs (markdown syntax visible but not rendered as formatting)
- Curated metadata header per tab (meeting context)
- Deduplication by Drive file ID
- Incremental updates (only rewrite tabs where source doc changed)

**Out of scope:**
- Google Sheets (stay as local CSV files)
- Google Slides (stay as local PDF files)
- Formatted insertion (the Docs API `insertText` treats all input as literal text — markdown is not interpreted)

## CLI Interface

```
calendar-fetcher consolidate --output-doc-id DOC_ID [--output-dir DIR] [--debug] [--quiet] [--log-file FILE]
```

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--output-doc-id` | yes | — | Google Doc ID to write tabs into |
| `--output-dir` | no | `calendar-artifacts` | Where fetched `.md` files and `.meta.json` sidecars live |
| `--debug` | no | off | Debug logging |
| `--quiet` | no | off | Suppress info output |
| `--log-file` | no | stderr | Write logs to file |

Prerequisite: `fetch` must have been run first. `consolidate` operates on local files only — it does NOT call the Calendar, Meet, or Drive APIs. It only calls the Docs API (batchUpdate).

## Auth

Requires `https://www.googleapis.com/auth/documents` scope (write, not readonly). The existing `check_auth_scopes()` in `gws.py` needs a new mode for this — the current scopes are all readonly except `drive` in upload mode.

Update `gws.py`:
- Add `DOCS_WRITE_SCOPE = "https://www.googleapis.com/auth/documents"` 
- Add a `consolidate: bool = False` parameter to `check_auth_scopes()` that requires the write scope

## Tab Structure

| Aspect | Value |
|--------|-------|
| Tab title | Source document's Drive file ID (stable, unique, ≤44 chars) |
| Tab content | Metadata header block + `---` separator + markdown file content |

### Metadata Header Format

```
Meeting: Weekly Engineering Sync
Date: 2026-03-26 13:25 EDT
Recurring: Yes
Attendees: Alice (accepted), Bob (declined), ~~Carol~~ (removed)
Source: https://docs.google.com/document/d/{file_id}/edit
---

{markdown content from local .md file}
```

The metadata is extracted from the `.meta.json` sidecar that references this file. Fields:
- **Meeting**: `summary` from the event metadata
- **Date**: `start` from the event metadata, formatted as date + time + timezone
- **Recurring**: "Yes" if `recurring_event_id` is not null, "No" otherwise
- **Attendees**: from `api_responses.calendar.attendees[]` — name + `(responseStatus)`. Strikethrough (`~~name~~`) for attendees with `responseStatus: "declined"` (preserves the information in markdown form)
- **Source**: link to the original Google Doc

When a doc is referenced by multiple meetings, use the **most recent** meeting's metadata (latest `start` date). This naturally handles meeting renames — the latest name wins.

## File ID to Metadata Mapping

To build the metadata header, the consolidator needs to find which meeting(s) reference each markdown file. The mapping is:

1. Scan all `.meta.json` files in the output directory
2. For each metadata file, iterate `artifacts[]`
3. If an artifact has `mime_type == "application/vnd.google-apps.document"` and `local_file` is set, map `drive_file_id → metadata`
4. If multiple metadata files reference the same `drive_file_id`, keep the one with the latest `start` date

This produces a `dict[str, dict]` mapping `file_id → meeting_metadata`.

## Update Logic

### Tracking State

Add a `consolidate_state` key to `fetcher-state.json`:

```json
{
  "modified_on": "2026-04-03T...",
  "consolidate_state": {
    "last_run": "2026-04-03T15:00:00Z",
    "output_doc_id": "1u47nG8k...",
    "tabs": {
      "file_id_abc": {
        "tab_id": "t.jrr8h4qmummg",
        "source_mtime": "2026-04-03T14:30:00Z"
      }
    }
  }
}
```

### Decision Flow Per File

For each local `.md` file with a known `drive_file_id`:

1. **Is there a tab for this file ID in the output doc?**
   - Check by matching tab title against the file ID
2. **Is the local file newer than what we last wrote?**
   - Compare local file mtime against `consolidate_state.tabs[file_id].source_mtime`

| Tab exists? | Local newer? | Action |
|-------------|-------------|--------|
| No | — | Create tab + insert content |
| Yes | Yes | Delete tab + create new tab + insert content |
| Yes | No | Skip (no change) |

Delete + recreate (rather than clear + rewrite) is simpler and self-healing — if the process crashes mid-update, the next run will see the tab is missing and recreate it.

### Reading Existing Tabs

At the start of each run, fetch the output doc to discover existing tabs:

```python
run_gws("docs", "documents", "get", params={
    "documentId": output_doc_id,
    "includeTabsContent": False,  # only need tab metadata, not content
})
```

Wait — `includeTabsContent` controls whether content is returned, but tab metadata (titles, IDs) might only be in `doc.tabs[]` when it's true. Need to verify. If tab metadata requires `includeTabsContent: true`, use it but ignore the content. The response will be larger but we only parse `tabs[].tabProperties`.

Build a map: `tab_title → tab_id` from the response.

## Google Docs API Operations

All via `gws docs documents batchUpdate --params '{"documentId":"DOC_ID"}' --json '{"requests":[...]}'`.

Key discoveries from prototyping (documented in `~/source/gws-cli-notes/NOTES.md`):

| Operation | Request | Notes |
|-----------|---------|-------|
| Create tab | `addDocumentTab` | `tabProperties.title` max 50 chars. Response includes `tabProperties.tabId`. |
| Delete tab | `deleteTab` | Takes `tabId`. |
| Insert text | `insertText` | `endOfSegmentLocation.tabId` targets a tab. **Plain text only** — markdown is not interpreted. |
| **CLI flag** | `--json` | Request body goes in `--json`, NOT `--body` (which doesn't exist). `--params` is for URL/path parameters only. |

### gws.py Changes

Add a `run_gws_write()` helper or extend `run_gws()` to support the `--json` flag:

```python
def run_gws_write(
    *args: str,
    params: dict | None = None,
    json_body: dict | None = None,
) -> dict:
    """Run a gws CLI command with a JSON request body."""
    cmd = ["gws", *args]
    if params is not None:
        cmd.extend(["--params", json.dumps(params)])
    if json_body is not None:
        cmd.extend(["--json", json.dumps(json_body)])
    # ... same error handling as run_gws
```

Or add `json_body` as an optional parameter to the existing `run_gws()`.

## New Module: `consolidator.py`

### Public API

```python
def consolidate(output_dir: Path, output_doc_id: str) -> None:
    """Consolidate fetched Google Docs into a single output document."""
```

### Internal Functions

```python
def _scan_doc_artifacts(output_dir: Path) -> dict[str, DocInfo]:
    """Scan .meta.json files and .md files, return map of file_id → DocInfo."""

def _read_existing_tabs(output_doc_id: str) -> dict[str, str]:
    """Read output doc, return map of tab_title → tab_id."""

def _build_metadata_header(meta: dict, file_id: str) -> str:
    """Build the curated metadata header string from meeting metadata."""

def _create_tab(output_doc_id: str, title: str, content: str) -> str:
    """Create a new tab and insert content. Returns tab_id."""

def _delete_tab(output_doc_id: str, tab_id: str) -> None:
    """Delete a tab from the output doc."""

def _read_consolidate_state(output_dir: Path) -> dict:
    """Read consolidate state from fetcher-state.json."""

def _write_consolidate_state(output_dir: Path, state: dict) -> None:
    """Write consolidate state to fetcher-state.json."""
```

### DocInfo Dataclass

```python
@dataclass
class DocInfo:
    file_id: str
    local_path: Path        # path to the .md file
    local_mtime: float      # file modification time
    meeting_summary: str     # from metadata
    meeting_start: str       # from metadata
    recurring: bool          # from metadata
    attendees: list[dict]    # from metadata api_responses.calendar.attendees
```

## Files to Create/Modify

| File | Action | What |
|------|--------|------|
| `calendar_fetcher/consolidator.py` | **Create** | New module with consolidation logic |
| `calendar_fetcher/__main__.py` | Modify | Add `consolidate` subcommand + `handle_consolidate()` |
| `calendar_fetcher/gws.py` | Modify | Add `--json` support to `run_gws()` or add `run_gws_write()`. Add `DOCS_WRITE_SCOPE`. Update `check_auth_scopes()`. |
| `calendar_fetcher/config.py` | Modify | Add consolidation constants if needed |
| `tests/test_consolidator.py` | **Create** | Unit tests for the new module |

## Implementation Order

1. **`gws.py`** — Add `--json` body support and docs write scope check
2. **`consolidator.py`** — Core logic: scan artifacts, read tabs, build headers, create/delete/update tabs
3. **`__main__.py`** — Wire up `consolidate` subcommand with argument parsing
4. **`tests/test_consolidator.py`** — Unit tests (mock `run_gws` calls)
5. **Manual test** — Run against real calendar data and verify output doc

## Verification

1. Run `calendar-fetcher fetch --calendar-id <ID> --lookback-days 30` to populate local files
2. Run `calendar-fetcher consolidate --output-doc-id <DOC_ID>`
3. Open output doc and verify:
   - One tab per unique Google Doc attachment
   - Tab titles are file IDs
   - Each tab has metadata header (meeting name, date, recurring, attendees)
   - Content below the header matches the local `.md` file
   - Re-running skips unchanged files
   - After modifying a source doc and re-fetching, re-running updates only the changed tab

## Open Questions (Resolved)

| Question | Resolution |
|----------|------------|
| Plain text vs markdown? | Markdown — preserves strikethrough for declined attendees. Inserted as literal text (not rendered). |
| Sheets/Slides in output doc? | No — Docs only. Sheets and Slides stay as local files. |
| Tab cleanup (delete default Tab 1)? | No — leave it. |
| Tab naming? | Drive file ID (stable, unique, <50 chars). |
| Multi-meeting references? | One tab per doc. Metadata uses the most recent meeting. |
| Update strategy? | Delete tab + recreate. Self-healing on crash. |
| Metadata header? | Curated: meeting title, date, recurring flag, attendees with status. |

## GWS CLI Notes

Key discoveries documented in `~/source/gws-cli-notes/NOTES.md`:

- `batchUpdate` uses `--json` for request body, NOT `--body`
- `--params` is for URL/path parameters only (e.g., `documentId`)
- `addDocumentTab` returns `tabProperties.tabId` in the response
- Tab title max length is 50 characters
- `insertText` is **plain text only** — markdown syntax is not interpreted as formatting
- Write operations require `https://www.googleapis.com/auth/documents` scope
