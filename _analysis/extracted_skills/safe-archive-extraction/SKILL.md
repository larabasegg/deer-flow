---
name: safe-archive-extraction
description: "Prevents path traversal, ZIP bomb, and symlink attacks when extracting user-supplied ZIP archives. TRIGGER when: implementing archive extraction for user-uploaded files, building a plugin or extension installer that accepts ZIP packages, designing any feature that extracts archives from untrusted sources."
---

# Safe Archive Extraction
*Prevents path traversal, ZIP bomb, and symlink attacks that `zipfile.extractall()` does not guard against.*

## Key decisions

1. Validate every archive member using both an upfront string check for `..` and absolute paths AND a post-normalization `is_relative_to()` check against the destination directory. Without the post-normalization check, path normalization can convert a seemingly safe encoded name into an escaping path that passes the string filter — the two checks protect against different attack encodings.

2. Enforce an uncompressed size limit by accumulating written bytes during streaming extraction, not by reading the archive's declared size fields. Without this, a ZIP bomb with a small compressed payload but a large declared size can be ignored if you trust the declared field — or a bomb with falsified metadata bypasses a pre-read check. Only bytes actually written to disk are authoritative.

3. Detect and skip symlink entries by inspecting Unix mode bits in the archive metadata before extraction. Without this, a symlink entry in the archive pointing to an arbitrary host path is extracted to the destination and followed on subsequent reads — an attacker can read or write to any file the process can access.

## Anti-patterns

- **What**: Use `zipfile.extractall(dest)` for user-supplied archives
- **Why**: `extractall()` does not check for path traversal, does not enforce size limits, and extracts symlinks as-is — all three attack vectors are left open
- **Symptom**: A crafted archive with `../../../etc/cron.d/evil` as a member writes outside the destination directory; a ZIP bomb exhausts disk or memory; discovered only under adversarial input — functional tests with normal archives always pass

- **What**: Trust the declared uncompressed size from the ZIP central directory to pre-check archive safety
- **Why**: The ZIP format allows attacker-controlled values in size fields; actual decompressed output can differ by arbitrary amounts
- **Symptom**: A 1KB ZIP with falsified metadata claiming 512MB uncompressed passes a pre-check but expands to exhaust disk during extraction; or a real 10GB bomb with accurate metadata is rejected by the check but still requires reading the header

## Structural template

```python
import zipfile, posixpath
from pathlib import Path

MAX_TOTAL_UNCOMPRESSED = 512 * 1024 * 1024  # 512MB


def is_symlink_member(info: zipfile.ZipInfo) -> bool:
    # Detect symlinks via Unix mode bits in external_attr
    return (info.external_attr >> 16) & 0o170000 == 0o120000


def safe_extract_archive(archive_path: Path, dest: Path) -> None:
    dest = dest.resolve()
    total_written = 0

    with zipfile.ZipFile(archive_path) as zf:
        for info in zf.infolist():

            # Layer 1: string check — obvious traversal and absolute paths
            if ".." in info.filename or info.filename.startswith("/"):
                raise ValueError(f"Unsafe archive member: {info.filename}")

            # Layer 2: skip symlinks — check mode bits before touching disk
            if is_symlink_member(info):
                logger.warning("Skipping symlink entry: %s", info.filename)
                continue

            # Layer 3: post-normalization destination check
            # Guards against encoding tricks that survive the string filter
            member_path = dest / posixpath.normpath(info.filename)
            if not member_path.resolve().is_relative_to(dest):
                raise ValueError(f"Path traversal detected: {info.filename}")

            member_path.parent.mkdir(parents=True, exist_ok=True)

            # Layer 4: stream write with accumulated size limit
            # Only actual bytes written count — declared size is untrusted
            with zf.open(info) as src, open(member_path, "wb") as dst:
                while chunk := src.read(65536):
                    total_written += len(chunk)
                    if total_written > MAX_TOTAL_UNCOMPRESSED:
                        raise ValueError(
                            "Archive too large — possible ZIP bomb"
                        )
                    dst.write(chunk)
```
