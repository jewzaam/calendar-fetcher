# calendar-fetcher

Fetch and download artifacts from Google Calendar meetings into a flat, searchable directory with structured metadata.

## Prerequisites

- Python 3.12+
- [gws CLI](https://github.com/jewzaam/gws-cli) installed and on PATH
- Google Cloud project with Calendar, Drive, Docs, Meet, and People APIs enabled

## Installation

```bash
make install-dev
```

## Authentication

Authentication is managed via `gws auth login`. The tool checks for required scopes on startup and exits with an error if they are insufficient.

```bash
# Download-only mode (read-only scopes):
gws auth login --readonly -s calendar,docs,drive,meet

# With Drive upload (requires drive write):
gws auth login -s calendar,docs,drive,meet
```

If you change scopes, log out first: `gws auth logout`.

## Usage

### Fetch meeting artifacts

```bash
# Fetch last 7 days from primary calendar
calendar-fetcher fetch

# Fetch specific date
calendar-fetcher fetch --date 2026-04-01

# Fetch last 30 days from a specific calendar
calendar-fetcher fetch --calendar-id work@example.com --lookback-days 30

# Dry run (show what would be downloaded)
calendar-fetcher fetch --dryrun

# Upload results to Google Drive
calendar-fetcher fetch --drive-folder-id FOLDER_ID
```

### Other commands

```bash
# Regenerate metadata from stored API responses
calendar-fetcher refresh-metadata

# Show processing stats
calendar-fetcher status

# List available calendars
calendar-fetcher list-calendars
```

### Options

| Option | Default | Description |
|--------|---------|-------------|
| `--calendar-id` | `primary` | calendar to query |
| `--output-dir` | `calendar-artifacts` | local directory for downloads |
| `--lookback-days` | `7` | days to look back |
| `--date` | | process only a specific date |
| `--drive-folder-id` | | upload results to this Drive folder |
| `--debug` | | enable debug logging |
| `--dryrun` | | show what would be downloaded |
| `--quiet` | | suppress non-essential output |
| `--log-file` | | write log output to file |

## Development

```bash
make check       # format, lint, typecheck, test, coverage
make test         # run tests only
make help         # show all targets
```

## License

Apache-2.0
