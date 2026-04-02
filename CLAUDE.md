# Calendar Fetcher

Fetches artifacts (docs, transcripts, recordings, slides, sheets) from Google Calendar meetings and exports them as portable local files. Installed via `pipx`. Uses the `gws` CLI for all Google API calls — no direct API client libraries.

## Architecture

```
CLI (__main__.py)
 └─ handle_fetch
     ├─ calendar_client.list_events    (paginated Calendar API query)
     ├─ calendar_client.parse_event    (extract attachments, description links, Meet code)
     └─ exporter.process_events
          └─ per event:
               ├─ meet_client.get_all_meet_data  (transcripts, recordings, participants)
               ├─ drive_client.*                  (export docs/sheets/slides)
               └─ metadata.write_metadata         (.meta.json sidecar)
```

All Google API calls go through `gws.run_gws()` which shells out to the `gws` CLI. There are no Google client library dependencies.

## Key Design Decisions

### Flat output directory

All artifacts and metadata land in a single directory. Filenames encode date, meeting title, document title, and document ID. The `naming.build_filename` function truncates slugs to stay within the 255-byte filesystem limit.

### gws CLI path restrictions

The `gws drive files export` and `gws drive files download` commands validate that `--output` paths are within the current working directory. Absolute paths outside CWD are rejected with exit code 3. Wrapper scripts must `cd` to the output directory before invoking calendar-fetcher.

### Attendees vs Participants

Calendar attendees and Meet participants are **different data** from different APIs. Neither replaces the other.

| Source | API | What it captures | Key fields |
|--------|-----|-----------------|------------|
| Calendar `attendees[]` | Calendar API | Who was **invited** | `email`, `responseStatus`, `comment`, `optional` |
| Meet `participants` | Meet API | Who actually **joined** | `displayName`, `user` (numeric ID), join/leave times |

Meet participants have no email — only a display name and numeric user ID. Calendar attendees have no join/leave times.

### Pagination

Google API list endpoints have default page size limits (Calendar: 250, Meet participants: 25). `calendar_client.list_events` handles pagination via `nextPageToken`. Other list calls (`meet_client.get_participants`, `drive_client.list_folder_files`) do not yet paginate — low risk for typical usage but will silently truncate if limits are exceeded.

### Path resolution

All user-supplied path args (`--output-dir`, `--log-file`) are resolved via `Path.expanduser().resolve()` in `main()` before any processing. Without this, `~/path` creates a literal `~` directory and relative paths resolve against process CWD.

### Lookback anchoring

`--lookback-days` counts back from the **last run date**, not today. On first run (no state), it counts back from today. State is stored in `fetcher-state.json` (`modified_on` ISO timestamp), written after each successful non-dryrun fetch. This ensures no gap between runs regardless of weekends or missed days, while preserving intentional overlap for catching updated attachments.

`--date` and `--lookback-days` are mutually exclusive (argparse enforced).

### Logging

When `--log-file` is set, logs go to a `RotatingFileHandler` (2MB, 1 backup) **instead of** stderr — not both. The only stdout output is the final summary line via `print()`. Logs append across runs.

## Subcommands

| Command | Purpose |
|---------|---------|
| `fetch` | Query calendar, download artifacts, write metadata |
| `refresh-metadata` | Re-extract fields from stored API responses (no API calls) |
| `status` | Count meetings and downloaded files |
| `list-calendars` | Show accessible calendars |

## File Layout

| File | Purpose |
|------|---------|
| `*.md` | Exported Google Docs (markdown) |
| `*.csv` | Exported Google Sheets (one per tab) |
| `*.pdf` | Exported Google Slides |
| `*.meta.json` | Per-meeting metadata sidecar (event data, artifacts, raw API responses) |
| `index.json` | Summary index of all meetings and downloaded files |
| `fetcher-state.json` | Last run timestamp for lookback anchoring |
| `fetch.log` | Log file (when `--log-file` is used) |
