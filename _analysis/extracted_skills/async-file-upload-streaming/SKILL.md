---
name: async-file-upload-streaming
description: "Prevents gateway memory exhaustion and event loop blocking when handling large file uploads in async web frameworks. TRIGGER when: implementing a file upload endpoint in an async web framework (FastAPI, aiohttp, Django Channels, or similar), choosing between reading the full body vs chunked streaming for file ingestion, adding file size limits to an async upload handler."
---

# Async Web Framework File Upload Streaming
*Prevents memory exhaustion and event loop blocking caused by reading entire uploaded files into memory before writing.*

## Key decisions

1. Stream uploaded files to disk using chunked reads instead of reading the full body at once. Without this, a 100MB document upload copies the entire file into the gateway process heap before writing — under concurrent uploads this exhausts memory and blocks the async event loop for the duration of the write.

2. Use async I/O for the disk write. Without this, even a streaming read is followed by a synchronous write that holds the event loop, making the gateway unresponsive to other requests during large file writes.

3. Enforce the size limit during streaming, before the full file has been read. Without this, a limit check after reading the full body still reads the entire file into memory before rejecting it — the memory spike occurs regardless of the limit, and adversarial clients can force OOM on any unprotected upload endpoint.

## Anti-patterns

- **What**: Read full body into memory, then write to disk (`content = await file.read(); path.write_bytes(content)`)
  **Why**: Full-body read loads the entire multipart payload into the heap; synchronous `write_bytes()` holds the event loop during the write
  **Symptom**: Gateway becomes unresponsive during large file uploads; memory spikes visible in production metrics during document ingestion; single-file unit tests pass because they use small fixtures

- **What**: Check file size after reading the full body to enforce upload limits
  **Why**: The entire file is already in process memory when the check runs; the limit rejects persistence but not the memory spike
  **Symptom**: Memory exhaustion still occurs before the reject path fires; adversarial clients can force OOM by sending oversized files to any upload endpoint

## Structural Template

```python
# FastAPI example — pattern applies to any async Python web framework
import aiofiles
from fastapi import UploadFile, HTTPException

MAX_UPLOAD_BYTES = 100 * 1024 * 1024  # 100MB
CHUNK_SIZE = 64 * 1024                # 64KB per read


@router.post("/upload")
async def upload_file(file: UploadFile):
    dest_path = resolve_upload_path(file.filename)
    total_written = 0

    async with aiofiles.open(dest_path, "wb") as f:
        while True:
            chunk = await file.read(CHUNK_SIZE)
            if not chunk:
                break

            total_written += len(chunk)

            # Enforce limit before writing — reject before disk use grows
            if total_written > MAX_UPLOAD_BYTES:
                await f.close()
                dest_path.unlink(missing_ok=True)
                raise HTTPException(413, "Upload exceeds size limit")

            await f.write(chunk)

    return {"path": str(dest_path), "bytes": total_written}


# Anti-pattern — avoid:
# content = await file.read()      # entire file in heap
# if len(content) > MAX: raise ... # too late — memory already used
# path.write_bytes(content)        # synchronous — blocks event loop
```

*Evidence from: FastAPI (Python async web framework), open-source AI agent project — March 2026*
