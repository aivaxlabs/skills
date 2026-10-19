# Publish and Share a File

Use when the agent must upload content to the shell sandbox and optionally generate a temporary public download URL.

## Objective

Write the raw content to the right sandbox path under the right shell user ID, confirm the upload, and expose a public URL only when the user explicitly asked to share the file.

## Preconditions

- The owning `X-Shell-User-Id` is known; paths under one shell user ID are invisible to another.
- The destination `path` inside the sandbox is decided.
- Explicit user approval exists when the upload replaces an existing file; deletes always need their own approval.

## Decision Tree

1. Does the destination directory exist? List it first; create it with `POST directory` when missing.
2. Upload with `POST file`: the raw request body is the file content written to `?path=<file>`, creating or replacing the file.
3. Confirm with file details (size, timestamps) or a fresh listing.
4. Was sharing requested? Only then call `GET file/public-address` and report the file name, size, and public URL. A public address exposes the file to anyone with the URL.

## Construction

```text
GET /api/v1/shell/io/listing?path=<dir>
POST /api/v1/shell/io/directory?path=<dir>
POST /api/v1/shell/io/file?path=<file>          (raw request body is the file content)
GET /api/v1/shell/io/file/details?path=<file>
GET /api/v1/shell/io/file/public-address?path=<file>
```

## Validation

- The listing shows the destination directory before upload.
- File details after upload match the expected size and timestamps.
- The public URL is generated only on explicit share requests.
- The trace ID is preserved.

## Failure Modes

- 404 on listing or details: wrong `path` or wrong `X-Shell-User-Id`. Confirm both before retrying.
- Upload exceeds the size cap: split the content or choose another delivery path instead of retrying.
- Unrequested public URL: never generate one speculatively; sharing is an explicit user decision.

## Limits

- The upload endpoint declares a request size cap in its description; global enforcement may still apply (see Plans and Limits).
- Nonexistent files return 404 on details and public-address calls.
