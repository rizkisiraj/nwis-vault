---
tags: [concept]
---

# File Upload and Storage

Up: [[Concepts MOC]]

## The idea

Accepting files is risky. The server must check size, type and content, store the bytes somewhere durable, and avoid trusting the filename. Uploads are normally sent as multipart HTTP requests.

Archives need extra care: a small zip can expand to gigabytes (a zip bomb), so limits on entries and compression ratio are applied.

Key points:

- **Validation**: size limit, allowed types, checksum.
- **Storage**: disk, database or object storage, with a stable path or id.
- **Integrity**: a hash proves a file was not corrupted or tampered with.
- **Atomic install**: unpack to a temporary place and swap in only after checks pass.

## In this codebase

- **Global ceiling.** `tgmdp_backend/src/main/resources/application.properties:150-151` sets `spring.servlet.multipart.max-file-size=50MB` and `max-request-size=50MB` for every multipart endpoint.
- **Ordinary uploads.** Many controllers take `@RequestParam("file") MultipartFile`: Excel imports such as `PersonalMaterialsImportController.java:98` (`POST /import`, `consumes = MULTIPART_FORM_DATA_VALUE`), `NonEngineerImportController`, `QAImportController`, and photo/attachment DTOs holding `List<MultipartFile>` (`dto/amsads/PhotoInsertRequestDto.java`). Permission is checked first with `@PreAuthorize`. `LogFilter` refuses to log multipart bodies, printing `[multipart upload, path=...]` instead (`middleware/LogFilter.java:175-229`).
- **The careful path: OTA bundle upload.** The registry (`tgmdp_backend/.../registry/service/`) is where OTA web bundles are uploaded, and it implements nearly every item in the list above:
  - *Zip-bomb limits* in config (`application.properties:200-204`): 10000 entries, 200 MB per entry, 500 MB total, compression ratio 200. `ZipProcessingService` enforces them (`~225-236`: `size / compressedSize > maxCompressionRatio` throws `ZIP_COMPRESSION_RATIO_TOO_HIGH`).
  - *Path traversal check* on entry names: `isUnsafeEntryName` rejects leading `/`, backslashes, `//`, NUL bytes and `..` (line ~238-260).
  - *Integrity*: SHA-256 is computed during the single-pass copy (`FileStorageService.SaveResult(path, sha256)`) and per entry for the manifest.
  - *Atomic write*: `LocalFileStorageService.java:79-85` writes to `UUID + ".tmp"` then `Files.move(tempPath, finalPath, REPLACE_EXISTING)`.
  - *Auth*: `app.ota.auth-enabled` (default true, off in the local profile) gates the endpoints.
  Why so strict: devices download and run these bundles, so one bad zip would break every phone. See [[Over-the-Air Updates]].
- **Client side of the same chain.** The shell re-verifies size, zip header and `x-checksum-sha256` before extracting: [[WebView Pool and Bundle Loading]].
- **Server-side file receipt in the ESB flow.** `MdpSoapEndpoint` streams the SOAP attachment to a `SYNC_DATA` folder using a 4 KB buffer, named by `transactionId`, then parses with DTD/entity processing off. The name comes from our own id, not from the sender.
- **Legacy.** The JSP-era upload used the container's multipart handling with files on the app-server disk; I did not trace it further.

## Why it matters

A weak upload path is a common security hole, and a corrupt bundle can break every device.

## Check yourself

Why compute the SHA-256 *while copying* the file instead of reading it again afterwards?

## Related

- [[Validation]] · [[Defensive Programming]] · [[Secrets Management]] · [[Database Transactions]] · [[Caching]]
- [[tgmdp_backend Logging and Configuration]] · [[tgmdp_backend]] · [[Concepts MOC]]
