# calendar-fetcher

[![Test](https://github.com/jewzaam/calendar-fetcher/actions/workflows/test.yml/badge.svg)](https://github.com/jewzaam/calendar-fetcher/actions/workflows/test.yml) [![Coverage](https://github.com/jewzaam/calendar-fetcher/actions/workflows/coverage.yml/badge.svg)](https://github.com/jewzaam/calendar-fetcher/actions/workflows/coverage.yml) [![Lint](https://github.com/jewzaam/calendar-fetcher/actions/workflows/lint.yml/badge.svg)](https://github.com/jewzaam/calendar-fetcher/actions/workflows/lint.yml) [![Format](https://github.com/jewzaam/calendar-fetcher/actions/workflows/format.yml/badge.svg)](https://github.com/jewzaam/calendar-fetcher/actions/workflows/format.yml) [![Type Check](https://github.com/jewzaam/calendar-fetcher/actions/workflows/typecheck.yml/badge.svg)](https://github.com/jewzaam/calendar-fetcher/actions/workflows/typecheck.yml) [![Mutation](https://github.com/jewzaam/calendar-fetcher/actions/workflows/mutation.yml/badge.svg)](https://github.com/jewzaam/calendar-fetcher/actions/workflows/mutation.yml)
[![Python 3.12+](https://img.shields.io/badge/python-3.12+-blue.svg)](https://www.python.org/downloads/) [![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

Fetch and download artifacts from Google Calendar meetings into a flat, searchable directory with structured metadata.

## Overview

- Query Google Calendar for events within a configurable lookback window
- Collect attachments, description links, and Google Meet transcripts/recordings
- Export Google Docs as markdown, Sheets as CSV, Slides as PDF
- Generate per-meeting JSON metadata with raw API responses
- Cache participant identities to reduce API calls across runs
- Optionally upload results to a Google Drive folder

## Installation

### Development

```bash
git clone https://github.com/jewzaam/calendar-fetcher.git
cd calendar-fetcher
make install-dev
```

### From Git (pipx)

```bash
pipx install git+https://github.com/jewzaam/calendar-fetcher.git
```

### From Git (pip)

```bash
pip install git+https://github.com/jewzaam/calendar-fetcher.git
```

### Prerequisites

- Python 3.12+
- [gws CLI](https://github.com/jewzaam/gws-cli) installed and on PATH
- Google Cloud project with Calendar, Drive, Docs, Meet, and People APIs enabled

### Authentication

Authentication is managed via `gws auth login`. The tool checks for required scopes on startup and exits with an error if they are insufficient.

```bash
# Download-only mode (read-only scopes):
gws auth login --readonly -s calendar,docs,drive,meet

# With Drive upload (requires drive write):
gws auth login -s calendar,docs,drive,meet
```

If you change scopes, log out first: `gws auth logout`.

## Usage

```bash
# Fetch last 7 days from primary calendar
calendar-fetcher fetch

# Fetch specific date
calendar-fetcher fetch --date 2026-04-01

# Fetch last 30 days from a specific calendar
calendar-fetcher fetch --calendar-id work@example.com --lookback-days 30

# Dry run (show what would be downloaded)
calendar-fetcher fetch --dryrun

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
| `--quiet` / `-q` | | suppress non-essential output |
| `--log-file` | | write log output to file |

## Development

```bash
make check       # format, lint, typecheck, test, coverage
make test        # run tests only
make help        # show all targets
```
