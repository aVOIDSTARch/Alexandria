# Alexandria - Questions & Suggestions

## Resolved Questions

### Q1: Build Order - Server needs file-processor for conversion?
**Resolution**: Build server first with a stub conversion client that returns "not configured". This follows the start-here.md order exactly. The stub gets replaced with a real HTTP client once the file-processor exists.

### Q2: Web Interface Approach?
**Resolution**: HTMX + server-side rendering. Templates rendered in Rust (minijinja or askama) with HTMX for dynamic updates (live search, pagination). No JavaScript framework required. Best balance of speed, reliability, and interactivity.

### Q3: Format Coverage Scope?
**Resolution**: All formats (maximum coverage). Support everything in both the Format enum AND the books/ directory structure: EPUB, PDF, MOBI, AZW3, HTML, Markdown, TXT, FB2, DOCX, CBZ, CBR, SSML, JSON, XML, RTF. This delivers "all the bells and whistles."

### Q4: Database Technology?
**Resolution**: SQLite + tantivy with a modular trait-based abstraction so any database backend can be swapped in later. The `DatabaseBackend` and `SearchBackend` traits define the contract; SQLite and tantivy are the initial implementations.

---

## Suggestions

### S1: EPUB Version Handling
The books/ directory has separate `epub-3/` and `epub-4/` folders. The existing code has an `EpubVersion` enum with V2 and V3. Suggest adding V4 support to the enum and using the EPUB version to route files to the correct subdirectory. EPUB 4 is still emerging, so the writer may initially produce EPUB 3 with forward-compatible features.

### S2: Search Index Durability
Suggest storing a hash of the SQLite book count + last-modified timestamp alongside the tantivy index. On startup, compare hashes -- if they match, skip rebuild. If not, do a full reindex. This avoids slow startups on large libraries while ensuring consistency.

### S3: Upload Security Sandbox
The server info.md says the upload folder "cannot be used to access the server outside." Suggest implementing this as:
- Uploads go to `books/unprocessed/unsorted/` which is NOT served by any web route
- The directory has no symlinks out
- A background task (or on-demand trigger) moves validated files to the appropriate `unprocessed/{format}/` directory
- Only files in `books/processed/` are served for download

### S4: API Versioning
All APIs are under `/api/v1/`. Suggest keeping this prefix from day one so future breaking changes can coexist under `/api/v2/` without disrupting existing clients.

### S5: MCP Server as Separate Binary
The file-processor info.md says "A MCP server will also be connected as a separate entity." Suggest making `file-processor-mcp` its own binary crate that depends on `file-processor-core` but runs independently. This way it can be deployed/updated separately and doesn't bloat the main server.

### S6: Plugin SDK Crate for TTS
As the plugin ecosystem grows, suggest creating a `tts-plugin-sdk` crate that plugin authors can depend on. It would provide the protocol implementation, manifest parsing, and helper traits so plugins don't have to implement the framed protocol from scratch.

### S7: Audiobook Naming Convention
Suggest a consistent naming scheme for audiobook output:
```
audiobooks/{book-title-slug}/{conversion-id}/
  metadata.json         # conversion config, source book ID, voice, etc.
  chapter-01.mp3
  chapter-02.mp3
  ...
```
The `metadata.json` preserves provenance and makes it easy to re-generate or compare conversions.

### S8: Configuration File Locations
Suggest supporting multiple config file locations (checked in order):
1. `./alexandria.toml` (project root)
2. `~/.config/alexandria/config.toml` (user config)
3. `/etc/alexandria/config.toml` (system config)
4. Environment variables (highest priority, override all files)

### S9: Rate Limiting for External API
The external API is public-facing. Suggest implementing per-API-key rate limiting using a token bucket algorithm (in-memory, no external dependency). Default: 100 requests/minute for external keys, 1000/minute for internal, unlimited for admin.

### S10: Graceful Shutdown
All three services should handle SIGTERM/SIGINT gracefully:
- Finish in-flight requests (with a timeout)
- Flush tantivy index to disk
- Close SQLite connection cleanly
- Log shutdown reason
This is especially important for the TTS hub which may have long-running jobs.

---

## Open Items for Future Discussion

1. **Login page scope**: The web interface login is feature-gated. What authentication method should it use? Session cookies with bcrypt-hashed passwords? OAuth? Start simple with username/password?

2. **Audiobook streaming**: Should the web interface support in-browser audio playback, or just downloads? HTMX can handle a simple audio player.

3. **Book cover extraction**: The EPUB reader can extract cover images. Should covers be stored separately and served via the `/books/{id}/cover` endpoint from day one, or deferred?

4. **Multi-user support**: Currently single API key tiers. Is multi-user support (user accounts, personal libraries, sharing) on the roadmap, or is single-user/single-admin sufficient?

5. **Backup strategy**: Should the server support database and index backup/restore commands?
