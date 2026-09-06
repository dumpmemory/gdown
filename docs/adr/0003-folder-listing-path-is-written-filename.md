# 3. Folder Listing `path` is the written filename, probing Google-native files only

Date: 2026-09-06

## Status

Accepted

## Context

A folder Listing (`--json`, `download_folder(skip_download=True)`) reports one
`{url, path}` entry per file from the embedded folder view alone, so it costs no
per-file requests. The folder view shows an ordinary file's full Drive filename
but only the extensionless Drive name of a Google-native file (Docs, Sheets,
Slides), whose written filename gains the export extension (`.docx`, `.xlsx`,
`.pptx`).

The folder parser can tell the two apart by link host, but download planning
guessed from whether the name contained a dot: a dotted name was trusted as-is
and an undotted name was resolved from the download response. A Google-native
file named `report.v2` exported as `report.v2.pptx` was therefore written as
`report.v2` (GitHub #498). Fixing the guess forces a decision on what a folder
Listing `path` promises for a Google-native file. Three options:

- Listing `path` is the written filename. Each Google-native entry costs one
  request to read the export filename from `Content-Disposition`; ordinary
  entries stay unprobed.
- Listing `path` is the Drive name as shown in the folder view. No extra
  requests, but a Google-native entry's `path` is not the filename that will
  be written, so `CONTEXT.md`'s definition of `path` would need an exception
  and callers could not predict on-disk names from the Listing.
- Every entry is probed and `path` is always the response filename. Uniform,
  but doubles requests for an all-ordinary folder for no naming gain, and lets
  a mismatched response header rename an ordinary file.

The single-file Listing already resolves its `path` from the response, and
already errors out rather than emit a name it cannot resolve.

## Decision

A folder Listing `path` is the filename that would be written, relative to the
download root and including nested directories. For a Google-native file that
is the export filename, resolved with one request per such file through the
same probe the single-file Listing uses. Ordinary files are written under, and
listed as, their folder-view Drive filename and are never probed. A failed
probe fails the whole Listing.

Download planning routes on the parser's native/ordinary distinction, not on
the presence of a dot in the name. `GoogleDriveFileToDownload` keeps its
`(id, path, local_path)` shape. Choosing an export format for files inside a
folder is a separate feature and not part of this decision.

## Consequences

- `CONTEXT.md`'s definition of `path` stays literally true for every entry:
  what a Listing prints is what a download writes.
- A folder Listing costs one extra request per Google-native file. Folders of
  ordinary files remain request-free beyond the folder view itself.
- A Google-native file's Listing `path` changes from the bare Drive name to the
  export filename. Anyone parsing folder `--json` output for those entries sees
  the new, correct name.
- A probe failure (permission, quota) makes the folder Listing fail rather than
  print a partial or misleading list, matching the single-file rule.
- Ordinary files keep the listed name even when the response header differs,
  so nested paths and resume continue to key on the folder view.
