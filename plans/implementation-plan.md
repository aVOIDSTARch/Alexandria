# Alexandria Superproject - Comprehensive Implementation Plan

## Context

Alexandria is a superproject combining three independent git submodules (server, file-processor, tts) with shared data directories (books/, audiobooks/) into a unified ebook library, conversion, and text-to-speech system. Substantial existing code lives in `/parts/` (ebook-converter, crusty-tts, crusty-project/meta-crusty) that must be copied and adapted -- never modified in place. The goal is to build a fully functional, self-contained system with powerful search, comprehensive format support, and a plugin-based TTS pipeline.

**Implementation order**: Server first (with stubs for conversion) -> File-processor -> TTS hub + plugins.

---

## Phase 1: Server (`/Alexandria/server/`)

The library server is the foundation. It stores, indexes, searches, and serves ebooks and audiobooks through three interfaces. It NEVER implements conversion itself -- it delegates to the file-processor via HTTP.

### 1.1 Cargo Workspace Structure

```
server/
  Cargo.toml                    # workspace root
  alexandria.toml               # example config
  crates/
    core/                       # alexandria-server-core
      Cargo.toml
      src/
        lib.rs
        config.rs               # Config cascade: env -> TOML -> defaults
        db/
          mod.rs                # DatabaseBackend trait (modular, swappable)
          sqlite.rs             # SQLite implementation via rusqlite
        search/
          mod.rs                # SearchBackend trait (modular, swappable)
          tantivy_backend.rs    # tantivy full-text search implementation
        models.rs               # Book, Audiobook, ApiKey, Permission, UploadJob
        storage.rs              # File storage (adapt /parts DirStore + safe_path)
        auth.rs                 # API key management, permission tiers, middleware
        error.rs                # thiserror types
        security.rs             # Reuse /parts security.rs checks
        conversion_client.rs    # HTTP client stub for file-processor
    external-api/               # alexandria-external-api
      Cargo.toml
      src/
        lib.rs
        routes.rs               # Upload, download, list, search (read-only + upload)
        middleware.rs            # Rate limiting, input validation, API key enforcement
    internal-api/               # alexandria-internal-api
      Cargo.toml
      src/
        lib.rs
        routes.rs               # Admin ops, batch ops, metadata editing, conversion triggers
        middleware.rs            # Elevated API key checks
    web-interface/              # alexandria-web
      Cargo.toml
      src/
        lib.rs
        routes.rs               # HTML pages served with HTMX interactivity
        templates/              # minijinja or askama templates
        static/                 # CSS, JS (HTMX), images
        auth_pages.rs           # Login page backend (gated by `login-page` feature flag)
    server-bin/                 # alexandria-server (main binary)
      Cargo.toml
      src/
        main.rs                 # Load config, build all routers, serve
```

### 1.2 Database Layer (Modular)

Trait-based abstraction so any database can be swapped in later:

```rust
pub trait DatabaseBackend: Send + Sync {
    fn get_book(&self, id: &str) -> Result<Book>;
    fn list_books(&self, opts: &ListOptions) -> Result<ListResult>;
    fn insert_book(&self, book: &NewBook) -> Result<String>;
    fn update_book(&self, id: &str, updates: &BookUpdate) -> Result<()>;
    fn delete_book(&self, id: &str) -> Result<()>;
    fn get_api_key(&self, key_hash: &str) -> Result<Option<ApiKey>>;
    fn create_api_key(&self, key: &NewApiKey) -> Result<String>;
    // ... audiobooks, upload queue, etc.
}
```

Initial implementation: SQLite via `rusqlite`

Tables:
- `books`: id (UUID), title, authors (JSON), language, isbn_10, isbn_13, description, subjects (JSON), series_name, series_position, format, file_path, file_size, sha256, cover_image_path, created_at, updated_at
- `audiobooks`: id, book_id (FK), conversion_id, chapter_count, format, total_duration_ms, created_at
- `api_keys`: key_hash (SHA-256), name, tier (external/internal/admin), permissions (JSON), created_at, expires_at, is_active
- `upload_queue`: id, api_key_id, original_filename, status, created_at, processed_at

### 1.3 Search Layer (Modular)

```rust
pub trait SearchBackend: Send + Sync {
    fn index_book(&self, book: &Book) -> Result<()>;
    fn remove_book(&self, id: &str) -> Result<()>;
    fn search(&self, query: &str, filters: &SearchFilters) -> Result<SearchResults>;
    fn rebuild_index(&self) -> Result<()>;
}
```

Initial implementation: tantivy (pure Rust, embedded)
- Index fields: title (Text, boosted), authors (Text), description (Text), subjects (Text), isbn (Text), format (Facet), id (stored)
- Rebuild from SQLite on startup if index is stale
- Incremental updates on add/update/delete
- Faceted filtering by format, language, subject
- Snippet highlights in search results

### 1.4 API Design

All APIs under `/api/v1/` with Axum router composition.

**External API** (`/api/v1/external/`) -- public, restricted:

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/upload` | Multipart file upload to secure staging area |
| GET | `/books` | List books (paginated, searchable) |
| GET | `/books/{id}` | Book metadata |
| GET | `/books/{id}/download` | Download book file (streaming) |
| GET | `/books/{id}/cover` | Download cover image |
| GET | `/audiobooks` | List audiobooks |
| GET | `/audiobooks/{id}/download` | Download audiobook |
| GET | `/search?q=...` | Full-text search via tantivy |
| GET | `/health` | Health check (unauthenticated) |

**Internal API** (`/api/v1/internal/`) -- elevated permissions (all external endpoints PLUS):

| Method | Endpoint | Description |
|--------|----------|-------------|
| PUT | `/books/{id}/metadata` | Update metadata |
| DELETE | `/books/{id}` | Delete book |
| POST | `/convert` | Trigger conversion via file-processor |
| GET | `/convert/{job-id}/status` | Conversion job status |
| POST | `/reindex` | Rebuild search index |
| GET | `/stats` | Library statistics |
| GET | `/api-keys` | List API keys (admin only) |
| POST | `/api-keys` | Create API key (admin only) |

**Web Interface** (`/`) -- HTMX-powered server-rendered UI:

| Route | Description |
|-------|-------------|
| `GET /` | Book catalog (paginated grid/list, search bar, HTMX partial updates) |
| `GET /book/{id}` | Book detail page with download links |
| `GET /search?q=...` | Search results (HTMX live search) |
| `GET /login` | Login page (feature-gated) |
| `POST /login` | Authenticate (feature-gated) |
| `GET /assets/*` | Static assets (CSS, HTMX JS, images) |

### 1.5 Authentication Model

- Three tiers: `external` (read + upload), `internal` (read-write + conversion), `admin` (full access)
- Header check: `X-API-Key` or `Authorization: Bearer`
- Keys stored as SHA-256 hashes in SQLite
- Web interface: session cookies when login feature is enabled
- CSRF protection on all web form submissions

### 1.6 File Storage

Adapted from `/parts/ebook-converter/crates/library-server/src/storage.rs`:
- Books in `books/processed/{format}/` directories
- Uploads go to `books/unprocessed/unsorted/` (isolated, not web-accessible)
- Audiobooks in `audiobooks/{book-slug}/{conversion-id}/`
- `safe_path()` path traversal guard
- Security checks: ZIP bomb, DRM, oversized files

### 1.7 Conversion Client (Stub)

Initially returns "service not configured" until file-processor exists. When configured, POSTs to file-processor HTTP API and polls for job status.

### 1.8 Key Files to Reuse (Copy + Adapt from /parts)

| Source | Target | Adaptation |
|--------|--------|------------|
| `ebook-converter/crates/library-server/src/storage.rs` | `core/src/storage.rs` | Add format subdirs, SQLite sync, tantivy indexing |
| `ebook-converter/crates/library-server/src/api.rs` | `external-api/src/routes.rs` | Split into external/internal, add auth |
| `ebook-converter/crates/core/src/security.rs` | `core/src/security.rs` | Reuse all checks as-is |
| `ebook-converter/crates/core/src/document.rs` | `core/src/models.rs` | Reuse Metadata struct |
| `ebook-converter/crates/core/src/library.rs` | `core/src/models.rs` | Reuse LibraryEntry, ListOptions |
| `ebook-converter/crates/core/src/detect.rs` | `core/src/` | Reuse Format enum and detect() |
| `crusty-project/meta-crusty/src/config.rs` | `core/src/config.rs` | Reuse env -> TOML cascade |
| `crusty-project/meta-crusty/src/lib.rs` | `server-bin/src/main.rs` | Reuse router composition |

### 1.9 Dependencies

- Web: axum 0.7, tower-http 0.5, tokio
- Database: rusqlite (with bundled feature)
- Search: tantivy
- Templates: minijinja (or askama)
- Auth: sha2, uuid
- Serialization: serde, serde_json, toml
- Logging: tracing, tracing-subscriber
- HTMX: served as static JS asset

---

## Phase 2: File-Processor (`/Alexandria/file-processor/`)

Independent ebook conversion library with CLI, HTTP API, manifest processing, library client, and MCP server. Supports ALL detected formats.

### 2.1 Cargo Workspace Structure

```
file-processor/
  Cargo.toml                    # workspace root
  crates/
    core/                       # file-processor-core (copy entire /parts ebook-converter-core, extend)
    cli/                        # file-processor-cli (adapt /parts cli, add batch command)
    api/                        # file-processor-api (embeddable library for other Rust crates)
    server/                     # file-processor-server (HTTP API with job tracking)
    mcp/                        # file-processor-mcp (MCP server, separate entity)
    ffi/                        # file-processor-ffi (C-ABI bindings, extend /parts)
    wasm/                       # file-processor-wasm (WebAssembly bindings, extend /parts)
```

### 2.2 Format Support Matrix (All Formats)

| Format | Reader | Writer | Notes |
|--------|--------|--------|-------|
| EPUB (2/3) | Copy /parts | Copy /parts | Exists |
| Plain Text | Copy /parts | Copy /parts | Exists |
| HTML | NEW (html5ever) | NEW | Deps already in workspace |
| Markdown | NEW (pulldown-cmark) | NEW | Deps already in workspace |
| PDF | NEW (lopdf) | NEW (printpdf) | Deps already in workspace |
| MOBI | NEW | NEW | Magic bytes detection exists |
| AZW3/KF8 | NEW | NEW | Extension of MOBI |
| JSON | NEW | NEW | Direct Document IR serde |
| XML | NEW (quick-xml) | NEW | Deps already in workspace |
| RTF | NEW | NEW | Custom tokenizer |
| FB2 | NEW (quick-xml) | NEW | XML-based, detection exists |
| DOCX | NEW (zip + quick-xml) | NEW | ZIP detection exists |
| CBZ | NEW (zip + image) | NEW | ZIP + image detection exists |
| CBR | NEW (unrar) | NEW | RAR detection exists |
| SSML | NEW (quick-xml) | NEW | XML-based, detection exists |

### 2.3 New Features Beyond /parts

- **Manifest processing**: Batch file with one conversion per line (`/path/to/book.epub -> pdf --repair`)
- **Library client**: HTTP client to fetch/upload books from/to the server API
- **MCP server**: Exposes convert, validate, repair, metadata, lookup as MCP tools
- **All format readers/writers**: Complete the matrix above

### 2.4 HTTP API

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/convert` | Submit conversion job |
| GET | `/api/v1/jobs/{id}/status` | Job status |
| GET | `/api/v1/jobs/{id}/result` | Download converted file |
| POST | `/api/v1/validate` | Validate a file |
| POST | `/api/v1/repair` | Repair a file |
| POST | `/api/v1/info` | Extract metadata |
| GET | `/api/v1/formats` | List supported formats |
| GET | `/api/v1/health` | Health check |

---

## Phase 3: TTS Hub + Plugins (`/Alexandria/tts/`)

Centralized TTS orchestration engine with plugin pipeline, menu API, and batch queue.

### 3.1 Cargo Workspace Structure

```
tts/
  Cargo.toml                    # workspace root
  crates/
    hub-core/                   # tts-hub-core (copy /parts crusty-core, extend)
    hub-daemon/                 # tts-hub-daemon (adapt /parts crusty-daemon)
    hub-cli/                    # tts-hub-cli (adapt /parts crusty-cli)
  plugins/                      # Plugin directories
    input-streams/
    tts-convertors/
    audio-format-convertors/
    pre-post-processors/
    convertors/
```

### 3.2 New Features Beyond /parts

- **Menu API** (`menu_api.rs`): Generate JSON/XML descriptions of all plugin capabilities and options for dynamic UI rendering
- **Batch queue** (`queue.rs`): Process multiple files with same orchestration config, preserve naming and metadata
- **Metadata pass-through** (`metadata.rs`): Maintain file naming and metadata throughout the pipeline

### 3.3 Daemon API

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/plugins` | List available plugins |
| GET | `/api/v1/plugins/{id}` | Plugin details |
| GET | `/api/v1/menu` | Get menu API (JSON or XML) |
| POST | `/api/v1/pipeline/validate` | Validate orchestration file |
| POST | `/api/v1/pipeline/run` | Execute pipeline |
| POST | `/api/v1/queue` | Submit batch queue |
| GET | `/api/v1/jobs/{id}/status` | Job status |
| GET | `/api/v1/jobs/{id}/stream` | Stream output audio |
| GET | `/api/v1/health` | Health check |

---

## Phase 4: Integration & Orchestration

### 4.1 Superproject Configuration

```toml
# Alexandria/alexandria.toml
[server]
bind = "0.0.0.0:3030"
library_path = "./books"
audiobooks_path = "./audiobooks"
db_path = "./data/alexandria.db"
search_index_path = "./data/search-index"
file_processor_url = "http://localhost:8080"
tts_url = "http://localhost:7420"

[file_processor]
bind = "0.0.0.0:8080"
library_url = "http://localhost:3030/api/v1/internal"
api_key_file = "./secrets/fp-api-key.txt"

[tts]
bind = "0.0.0.0:7420"
plugins_path = "./tts/plugins"
library_url = "http://localhost:3030/api/v1/internal"
```

### 4.2 Deployment

- Docker: Multi-stage Dockerfile per submodule, Docker Compose for full stack
- Systemd: Unit files per service
- Install script: Interactive setup

### 4.3 Data Flow

```
Upload -> server (books/unprocessed/unsorted/)
       -> classify format -> store metadata in SQLite -> index in tantivy
       -> move to books/unprocessed/{format}/

Convert -> server calls file-processor HTTP API
        -> file-processor converts format
        -> uploads result back to server
        -> server stores in books/processed/{format}/

TTS     -> TTS hub fetches text from library
        -> splits by chapter, queues
        -> Pipeline: pre-process -> TTS -> post-process -> audio format
        -> Output stored in audiobooks/{book}/{conversion}/
```

---

## Testing Strategy

### Phase 1 (Server)
- Unit tests: DB/search trait implementations, auth middleware, config, storage, security
- Integration tests: Full API tests for all three interfaces
- Security tests: Path traversal, malformed uploads, DRM, ZIP bombs

### Phase 2 (File-Processor)
- Unit tests per format: Round-trip read -> write -> read
- CLI tests with assert_cmd, manifest batch tests
- Property tests with proptest for encoding edge cases

### Phase 3 (TTS)
- Unit tests: Orchestration, registry, pipeline, menu API, queue
- Integration tests: End-to-end with mock plugins
- Reuse 27+ tests from /parts

### Phase 4 (Integration)
- End-to-end: upload -> convert -> TTS -> verify audiobook in library

---

## Verification Plan

1. Phase 1: `cargo test --workspace` in server/. Upload EPUB, search, download, verify web UI.
2. Phase 2: `cargo test --workspace` in file-processor/. Convert EPUB to PDF via CLI and HTTP API.
3. Phase 3: `cargo test --workspace` in tts/. Run pipeline with sample plugins, verify menu API.
4. Phase 4: Docker Compose all services. Full end-to-end workflow.
