# Test Plan

> Testing strategy for calendar-fetcher.

## Overview

**Project:** calendar-fetcher
**Primary functionality:** Fetch and download Google Calendar meeting artifacts as portable documents with structured metadata.

## Testing Philosophy

All Google API interactions go through the `gws` CLI, which is mocked at the subprocess boundary. No tests make real network calls.

Key principles:

- Mock external boundaries (subprocess/gws CLI), test internal logic directly
- Every module has a corresponding test file
- Error paths are tested — API failures, missing fields, inaccessible files

## Test Categories

### Unit Tests

| Module | Coverage | Notes |
|--------|----------|-------|
| `config.py` | Constants only | No behavior to test |
| `gws.py` | Core runner, auth, retry, timeout | Mocks subprocess.run |
| `models.py` | Dataclass construction, defaults | Pure data |
| `naming.py` | slugify, build_filename, extract_date | Pure functions |
| `calendar_client.py` | Event listing, attachment extraction, description link parsing, Meet code extraction | Mocks run_gws |
| `meet_client.py` | Conference records, transcripts, recordings, participants, People API resolution | Mocks run_gws |
| `drive_client.py` | File metadata, doc/sheet/slide export, folder listing, upload | Mocks run_gws |
| `metadata.py` | Sidecar building, index generation, refresh | Uses tmp_path |
| `exporter.py` | Artifact processing, event orchestration | Mocks client modules |
| `__main__.py` | Argument parsing, handler dispatch | Mocks application modules |

### Integration Tests

None currently. All Google API calls are mocked. Integration testing requires a real gws CLI session with authenticated credentials.

## Untested Areas

| Area | Reason |
|------|--------|
| Real Google API calls | Requires live credentials and network access |
| Upload functionality | Placeholder implementation pending gws CLI investigation |
| Multi-tab Google Sheets export | Drive API limitation — exports entire spreadsheet, not individual tabs |
| End-to-end CLI execution | Handlers tested individually, not as full CLI invocations |

## Bug Fix Testing Protocol

All bug fixes follow TDD:

1. Write a failing test that exposes the bug
2. Verify the test fails
3. Implement the fix
4. Verify the test passes
5. Commit test and fix together with issue reference

## Coverage Goals

**Target:** 80%+ line coverage

Focus on testing behavior and error paths, not configuration constants.

## Running Tests

```bash
make test        # run all tests
make coverage    # run with coverage report
make check       # format + lint + typecheck + test + coverage
```

## Test Data

- Generated programmatically in fixtures (mock API responses as dicts)
- File operations use pytest `tmp_path` fixture
- No external test data files needed

## Maintenance

1. **Adding features**: Write tests for new functionality
2. **Fixing bugs**: Follow TDD protocol (test first, then fix)
3. **Refactoring**: Existing tests should pass without modification
4. **Removing features**: Remove associated tests

## Changelog

| Date | Change | Rationale |
|------|--------|-----------|
| 2026-04-01 | Initial test plan | Project creation |
