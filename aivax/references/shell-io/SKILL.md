---
name: aivax-shell-io
description: Work with the per-account shell file sandbox — list, download, inspect, publish, upload, and delete sandboxed files and directories. Load when the agent must read or mutate files inside the shell sandbox scoped by account and X-Shell-User-Id.
---

# Shell I/O

This sub-skill owns the shell file sandbox. The sandbox is an isolated filesystem scoped by the authenticated account and the `X-Shell-User-Id` header. Different shell user IDs never see each other's files, even under the same account.

The same sandbox backs the gateway's virtual Bash (`parameters.enableBash`) when `bashOptions.allowDataPersistence` is on: the shell user ID here must equal the end user's external ID used in the conversation, and the files appear to the model under `/home/workspace`. Without persistence, or without an external user ID, the model's shell filesystem is temporary and nothing appears here.

All routes share the prefix `/api/v1/shell/io` and require the `X-Shell-User-Id` header. All file and directory operations take a required `path` query parameter naming a path inside the sandbox.

## Operating Files

- `situations/publish-and-share.md`: upload content and optionally generate a temporary public download URL.

## Discovery Workflow

```text
GET /api/v1/shell/io/listing?path=<dir>
GET /api/v1/shell/io/file?path=<file>
GET /api/v1/shell/io/file/details?path=<file>
GET /api/v1/shell/io/file/public-address?path=<file>
POST /api/v1/shell/io/file?path=<file>          (raw request body is the file content)
POST /api/v1/shell/io/directory?path=<dir>
DELETE /api/v1/shell/io/file?path=<file>
DELETE /api/v1/shell/io/directory?path=<dir>    (recursive; root paths rejected with 400)
```

## When To Use

Use this sub-skill when the agent must list a sandbox directory, download or inspect a sandboxed file, generate a temporary public download URL, upload content by writing the raw request body to a file, create a directory, or delete a sandboxed file or directory.

Do not use this sub-skill for:

- Long-term user memories. Use `references/memories/` instead.
- Account skills or gateway configuration. Use `references/skill-development/` or `references/ai-gateways/` instead.

## Endpoints

- **List Directory** (`GET listing`): returns entries with name, directory flag, size in bytes (null for directories), and UTC creation and modification timestamps. Returns 404 when the directory does not exist.
- **Download File** (`GET file`): returns the raw file contents. Returns 404 when the file does not exist.
- **Get File Details** (`GET file/details`): returns name, size, UTC timestamps, MIME type, and plain-text detection. Returns 404 when the file does not exist.
- **Get File Public Address** (`GET file/public-address`): creates a temporary public download URL for one file. Returns the file name, size, and public URL. Returns 404 when the file does not exist.
- **Upload File** (`POST file`): writes the raw request body to the destination path, creating or replacing the file.
- **Create Directory** (`POST directory`): creates the directory path inside the sandbox.
- **Delete File** (`DELETE file`): deletes one file. Returns 404 when the file does not exist.
- **Delete Directory** (`DELETE directory`): recursively deletes a non-root directory and its contents. Root paths are rejected with 400. Returns 404 when the directory does not exist.

## Safety

- The sandbox is isolated per account plus shell user ID. Never assume a path visible under one `X-Shell-User-Id` exists under another.
- Deleting a directory is recursive and cannot be undone. Deleting any file or directory requires explicit user approval; state the exact `path` and the shell user ID before calling.
- A public address exposes the file to anyone with the URL. Generate one only when the user explicitly asked to share the file, and say so in the response.

## Validation

- List before reading, publishing, or deleting to confirm the `path` exists under the current shell user ID.
- After upload, confirm with file details (size and timestamps) or a fresh listing.
- After delete, confirm the path is gone (404 on details, or absent from listing).
- The trace ID is preserved.

## Limits

- The upload endpoint declares a request size cap in its description, but the handler has no local size validation; global enforcement may still apply. Check Plans and Limits for the current value.
- The sandbox root cannot be deleted; root delete attempts return 400.
- Nonexistent files and directories return 404 on read, details, public-address, and delete calls.

## Escalation

- The wrong file set is visible: the `X-Shell-User-Id` is probably wrong. Confirm which shell user ID owns the data before retrying.
- The task needs long-term user memory rather than files: load `references/memories/` (memories live in the `@memories` RAG collection, not in the sandbox).
- The upload exceeds the size cap: ask the user to split the content or provide another delivery path instead of retrying.
- A destructive call was not approved: stop and ask for explicit approval with the exact path.
