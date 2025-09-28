# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Development
```bash
# Run the CLI (from codex-rs directory)
just codex [args]  # or just c [args]

# Run TUI interface
just tui [args]

# Run exec subcommand
just exec [args]

# Run MCP server
just mcp-server-run [args]
```

### Testing
```bash
# Run all tests with nextest
just test

# Run specific test
just test <test_name>

# Update snapshot tests
cargo insta review
```

### Code Quality
```bash
# Format Rust code
just fmt

# Run clippy linter
just clippy

# Fix clippy issues automatically
just fix

# Format JavaScript/TypeScript
pnpm format:fix
```

## Architecture

This is a polyglot monorepo combining Rust for core functionality and Node.js for distribution:

### Distribution Strategy
- **Node.js wrapper** (`codex-cli/bin/codex.js`): Platform-aware launcher that selects the correct Rust binary
- **Rust binaries**: Platform-specific executables (x86_64/aarch64, Linux/macOS/Windows) in `codex-cli/dist/`
- **npm package**: Distributed via npm, installs appropriate binary for user's platform

### Core Rust Workspace (`codex-rs/`)
The project uses a Cargo workspace with 25+ crates organized by functionality:

- **cli**: Main CLI entry point, argument parsing, and command routing
- **core**: Core business logic, agent implementation, and tool orchestration
- **tui**: Terminal UI using ratatui (custom patched version for better performance)
- **protocol**: Core protocol definitions and message types
- **mcp-client/server/types**: Model Context Protocol implementation for tool integration
- **exec/execpolicy**: Command execution with policy enforcement
- **linux-sandbox**: Platform-specific sandboxing (Landlock/seccomp on Linux)
- **chatgpt**: ChatGPT API integration and authentication
- **responses-api-proxy**: API proxy for handling responses

### Key Architectural Patterns
1. **Tool System**: Extensible tool framework where each tool (Read, Write, Bash, etc.) is implemented as a separate module with standardized interfaces
2. **Async Runtime**: Tokio-based async runtime for concurrent operations
3. **Event-Driven TUI**: Crossterm for terminal events, ratatui for rendering with custom state management
4. **Sandbox Execution**: Environment-aware execution with platform-specific sandboxing capabilities

## Development Guidelines

### Code Style Requirements
- **Rust 2024 edition** required
- **No unwrap/expect**: `unwrap_used` and `expect_used` are denied by clippy
- **Import granularity**: Use `imports_granularity=Item` in rustfmt
- **Error handling**: Use `anyhow` for application errors, `thiserror` for library errors

### TUI Development
- Use ratatui's `Stylize` trait for concise styling (e.g., `text.bold().fg(Color::Blue)`)
- Prefer `.into()` conversions over manual construction
- Follow styling guidelines in `codex-rs/tui/styles.md`
- Test TUI components with snapshot testing using `insta`

### Testing Conventions
- Use `cargo-nextest` for faster test execution
- Snapshot tests for TUI rendering with `cargo-insta`
- Environment-aware tests that adapt to sandbox limitations
- Mock external services in tests

### Authentication & Security
- Supports ChatGPT plans (Plus, Pro, Team, Enterprise) and API keys
- Zero Data Retention (ZDR) compliance for enterprise users
- Platform-specific sandboxing with graceful degradation
- Environment variables: `CHATGPT_API_KEY`, `CODEX_SANDBOX_NETWORK_DISABLED`