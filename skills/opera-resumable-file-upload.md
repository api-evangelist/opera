---
name: Upload a large file to Opera Ads with resume
description: >-
  Push a large data file to the Opera Ads File Upload API over the TUS 1.0.0
  resumable protocol, chunked, with authenticated retry and resume from the
  server's authoritative offset.
api: openapi/opera-file-upload-api-openapi.yml
operations:
  - getUploadCapabilities
  - createUploadSession
  - uploadChunk
  - getUploadOffset
  - cancelUpload
  - listUploads
---

# Upload a large file to Opera Ads with resume

The Opera Ads File Upload API is a TUS 1.0.0 server. Uploads survive network
failure and continue from the last byte the server durably received.

## Before you start

- The upload **host is issued per customer** by the Opera team; the docs write
  it as `https://<service-host>/upload`. You cannot discover it — you must be
  given it.
- Get credentials from Opera: either an `adx_`-prefixed API key or an HMAC key
  pair. **Prefer HMAC** — the secret never travels with the request.
- HMAC: sign `{METHOD}\n{PATH}\n{TIMESTAMP}` with HMAC-SHA256, lowercase hex,
  and send `X-HMAC-Key-Id`, `X-HMAC-Timestamp` (Unix seconds) and
  `X-HMAC-Signature`. The server rejects timestamps outside ±300 seconds, so
  your clock must be right.

## Steps

1. **(Optional) `getUploadCapabilities`** — `OPTIONS /upload/files`. This is the
   one unauthenticated call. Read `Tus-Max-Size` (4 GB) and `Tus-Extension`
   before committing to an upload plan.

2. **`createUploadSession`** — `POST /upload/files` with `Content-Length: 0`,
   `Tus-Resumable: 1.0.0`, `Upload-Length: <file size in bytes>`, and
   `Upload-Metadata: key <base64>,filename <base64>`. `key` is the destination
   path relative to your allocated prefix, e.g. `20260401/file.csv.gz`. A
   leading `/` is stripped; `..` is rejected with 400.

   **Save the `Location` header.** It is the session URL and the only way to
   resume if your process dies.

3. **`uploadChunk`** — `PATCH {Location}` with
   `Content-Type: application/offset+octet-stream`, `Upload-Offset` set to the
   byte where this chunk starts, and the chunk as the body. The 204 response's
   `Upload-Offset` is your next offset. Use 50 MB chunks by default; 100–200 MB
   on stable high-bandwidth links; 10–25 MB on mobile. Chunks must be ≥ 5 MB
   (except the final one) and ≤ 256 MB — the gateway rejects larger with 413.

4. **On failure, `getUploadOffset`** — `HEAD {Location}` returns the
   authoritative `Upload-Offset`. Re-read the file from that offset and
   continue. **This call can block for 1–3 minutes** after an interrupted
   chunk while the server finalizes; that is documented behaviour, not an
   error. Use a generous timeout (240s) and retry until it answers.

5. **Repeat until `Upload-Offset == Upload-Length`.** That is completion —
   there is no separate finalize call.

6. **`cancelUpload`** — `DELETE {Location}` aborts. It is idempotent: already
   complete or already cancelled returns 204 with no further action. This is
   the ONLY idempotent operation Opera Ads documents, and it is idempotent by
   TUS, not by an idempotency key.

7. **Batches:** upload each file, then signal completion by uploading a
   zero-length `_SUCCESS` marker under the same date prefix, e.g.
   `20260401/_SUCCESS`.

## Failure handling

- `400` bad `Upload-Metadata` (missing/empty `key`, path traversal) or a
  non-final chunk under 5 MB
- `401` missing/invalid credentials — check the HMAC timestamp window first
- `403` the upload ID exists but belongs to another customer
- `404` unknown upload ID
- `409` already complete, or your `Upload-Offset` disagrees with the server's —
  do NOT guess, call `getUploadOffset` and restart from what it says
- `413` chunk over 256 MB, or `Upload-Length` over 4 GB at creation
- Errors are `{"error": "..."}`, not RFC 9457 problem details.

## Watch the quotas

500 GB total storage per customer. Sessions expire after **7 days** — an
incomplete upload older than that is cancelled and its data deleted, and resume
is no longer possible. Use `listUploads` (`GET /upload/files?status=0`) to find
sessions still in progress before they lapse. The `offset` shown by
`listUploads` / `getUpload` is NOT a safe resume point for an in-progress
upload; only `HEAD` is.
