# Alexandria

A modular digital library platform built in Rust. Alexandria combines ebook management, format conversion, and text-to-speech into a unified system, with each module maintained as an independent git submodule.

## Project Information

- **Author:** Louis Casinelli Jr
- **Language:** Rust
- **License:** MIT

## Modules

Alexandria contains three independent submodules, each with its own repository and workspace:

### Server (`server/server/`)

The Alexandria Library Server manages a digital library of ebooks and audiobooks. It provides a REST API for uploading, searching, and downloading books, a web-based catalog interface, and optional integration with the file-processor and TTS hub services.

- **Interfaces:** External API, Internal/Admin API, Web UI (minijinja + HTMX)
- **Storage:** SQLite (rusqlite) with Tantivy full-text search
- **Security:** Tiered API key authentication (External, Internal, Admin), ZIP bomb detection, path traversal protection
- **Default port:** `127.0.0.1:3030`

```bash
cd server/server
cargo build --release
cargo run --bin alexandria-server
```

Configuration via environment variables (`ALEXANDRIA_BIND`, `ALEXANDRIA_LIBRARY_PATH`, etc.) or `alexandria.toml`. See [server/server/](server/server/) for full details.

### File Processor (`file-processor/file-processor/`)

A comprehensive ebook format conversion, validation, and repair library. Converts between 15 formats (EPUB, PDF, MOBI, AZW3, FB2, DOCX, HTML, Markdown, and more) through a unified Document intermediate representation.

- **Interfaces:** CLI, HTTP server, MCP server, Rust API, C FFI, WebAssembly
- **Features:** Convert, validate (with WCAG accessibility checks), repair, metadata management, merge/split, dedup, Open Library lookup

```bash
cd file-processor/file-processor
cargo build --release

# CLI usage
./target/release/file-processor convert input.epub -f txt -o output.txt
./target/release/file-processor validate book.epub --accessibility

# HTTP server
./target/release/file-processor-server
```

See [file-processor/file-processor/](file-processor/file-processor/) for full documentation.

### TTS Hub (`tts/tts/`)

A plugin-based text-to-speech orchestration engine. Processes text through a configurable multi-stage pipeline: preprocessing, TTS synthesis, audio conversion, and post-processing. Plugins are subprocess-based (shell scripts or binaries) discovered via `plugin.toml` manifests.

- **Interfaces:** CLI, REST API daemon (Axum, default port `7420`)
- **Pipeline stages:** Pre-processor, TTS, Converter, Post-processor
- **Included plugins:** text-prep, say-tts (macOS `say`), loudnorm (EBU R128), mp3-encoder

```bash
cd tts/tts
cargo build --release

# Run a pipeline
./target/release/tts-hub-cli \
  --orchestration examples/audiobook-pipeline.toml \
  --plugins plugins/

# Start the daemon
TTS_HUB_PORT=7420 ./target/release/tts-hub-daemon
```

See [tts/tts/](tts/tts/) for plugin development and orchestration configuration.

## Building the Full Platform

Each module builds independently. From the Alexandria root:

```bash
# Build all modules
(cd server/server && cargo build --release)
(cd file-processor/file-processor && cargo build --release)
(cd tts/tts && cargo build --release)
```

The server can optionally delegate format conversion to the file-processor and audiobook generation to the TTS hub when their services are running. Configure with:

- `ALEXANDRIA_FILE_PROCESSOR_URL` - URL of the file-processor HTTP server
- `ALEXANDRIA_TTS_URL` - URL of the TTS hub daemon

## Requirements

- Rust 1.75+
- macOS or Linux
