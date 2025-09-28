# Codex-rs Architecture Documentation

## Overview

Codex-rs is the core Rust implementation of OpenAI's Codex CLI, a sophisticated command-line interface for AI-powered coding assistance. This document provides comprehensive technical documentation of the codebase architecture, design patterns, and implementation details.

## Project Structure

```
codex-rs/
├── cli/                    # Main CLI entry point and command routing
├── core/                   # Core business logic and protocol handling
├── tui/                    # Terminal user interface (ratatui-based)
├── protocol/               # Protocol definitions and serialization
├── mcp-client/            # Model Context Protocol client
├── mcp-server/            # MCP server implementation
├── mcp-types/             # MCP type definitions
├── exec/                  # Command execution framework
├── execpolicy/            # Execution policy management
├── linux-sandbox/         # Linux security sandboxing
├── chatgpt/               # ChatGPT API integration
├── responses-api-proxy/   # API proxy layer
├── file-search/           # File search functionality
├── git-tooling/           # Git integration utilities
├── apply-patch/           # Patch application logic
├── ansi-escape/           # ANSI terminal handling
├── login/                 # Authentication management
└── common/                # Shared utilities

```

## Core Architecture

### 1. Layered Architecture

The system follows a clean layered architecture:

```
┌─────────────────────────────────────┐
│         CLI / TUI Layer             │  User Interface
├─────────────────────────────────────┤
│         Core Logic Layer            │  Business Logic
├─────────────────────────────────────┤
│        Protocol Layer                │  Communication
├─────────────────────────────────────┤
│      Tool Execution Layer           │  Tool Management
├─────────────────────────────────────┤
│       Platform Layer                 │  OS Integration
└─────────────────────────────────────┘
```

### 2. Message Flow Architecture

```rust
// Submission Flow: User → Core → Model → Response
pub struct Submission {
    pub id: String,
    pub op: Op,  // UserInput, UserTurn, ExecApproval, etc.
}

// Event Flow: Core → UI
pub enum Event {
    SessionConfigured,
    TurnBegin,
    Message(MessageEvent),
    ToolUse(ToolUseEvent),
    TurnComplete,
}
```

### 3. Async Architecture

The entire system is built on Tokio's async runtime with sophisticated concurrency patterns:

```rust
// Channel-based communication
let (tx_sub, rx_sub) = async_channel::unbounded::<Submission>();
let (tx_event, rx_event) = async_channel::unbounded::<Event>();

// Event loop pattern
tokio::select! {
    Some(submission) = rx_sub.recv() => handle_submission(submission).await,
    Some(event) = internal_events.recv() => process_event(event).await,
    _ = shutdown_signal() => break,
}
```

## Design Patterns

### 1. Command Pattern

All CLI commands follow a consistent pattern:

```rust
#[derive(Debug, clap::Parser)]
struct Cli {
    #[command(subcommand)]
    command: Subcommand,
}

#[derive(Debug, clap::Subcommand)]
enum Subcommand {
    Exec(ExecCli),      // Execute commands
    Login(LoginCommand), // Authentication
    Proto(ProtoCli),    // Protocol operations
    Mcp(McpCli),        // MCP operations
    Apply(ApplyCommand), // Apply patches
}
```

### 2. Factory Pattern for Tools

Tools are dynamically created based on model capabilities:

```rust
pub struct ToolsConfig {
    pub shell_type: ConfigShellToolType,
    pub plan_tool: bool,
    pub apply_patch_tool_type: Option<ApplyPatchToolType>,
    pub web_search_request: bool,
    pub include_view_image_tool: bool,
    pub experimental_unified_exec_tool: bool,
}

impl ToolsConfig {
    pub fn new(params: &ToolsConfigParams) -> Self {
        match params.model_family {
            ModelFamily::Sonnet => /* config */,
            ModelFamily::Opus => /* config */,
            ModelFamily::Grok => /* config */,
            _ => /* default */,
        }
    }
}
```

### 3. Observer Pattern in TUI

Event-driven updates with channel-based notifications:

```rust
pub struct App {
    app_event_tx: AppEventSender,  // Send events
    // Components subscribe to events
}

impl App {
    async fn handle_event(&mut self, event: AppEvent) {
        match event {
            AppEvent::CodexEvent(e) => self.process_codex_event(e),
            AppEvent::FileSearchResult { .. } => self.update_search(),
            // ... handle all events
        }
    }
}
```

### 4. Strategy Pattern for Execution

Different execution strategies based on configuration:

```rust
trait ExecutionStrategy {
    async fn execute(&self, cmd: &str) -> Result<Output>;
}

struct SandboxedExecution { /* Landlock + seccomp */ }
struct DirectExecution { /* No sandboxing */ }
struct DockerExecution { /* Container isolation */ }
```

## Tool System Architecture

### Tool Definition

Tools are defined with JSON schemas for validation:

```rust
pub struct ResponsesApiTool {
    pub name: String,
    pub description: String,
    pub strict: bool,
    pub parameters: JsonSchema,
}

// Example: Bash tool
ResponsesApiTool {
    name: "bash".to_string(),
    description: "Execute bash commands".to_string(),
    parameters: JsonSchema::Object {
        properties: btreemap! {
            "command" => JsonSchema::String {
                description: Some("Command to execute".to_string())
            },
        },
        required: Some(vec!["command".to_string()]),
    },
}
```

### Tool Execution Lifecycle

1. **Registration**: Tools register with the core at startup
2. **Validation**: Input validated against JSON schema
3. **Authorization**: Check approval policies
4. **Execution**: Run in appropriate context (sandbox/direct)
5. **Result Streaming**: Stream output back to model
6. **Cleanup**: Resource cleanup and state update

### Built-in Tools

| Tool | Purpose | Security |
|------|---------|----------|
| `bash` | Execute shell commands | Sandboxed by default |
| `read` | Read file contents | Path validation |
| `write` | Write files | Path restrictions |
| `edit` | Modify existing files | Atomic operations |
| `search` | Search codebase | Indexed search |
| `web_search` | Search web | Rate limited |
| `plan` | Task planning | State tracking |
| `apply_patch` | Apply code patches | Rollback support |

## TUI Architecture

### Component Hierarchy

```
App
├── ChatWidget          # Main chat interface
│   ├── InputArea      # User input handling
│   ├── MessageList    # Message display
│   └── StatusBar      # Status information
├── FileSearchOverlay  # File search UI
├── DiffViewer        # Git diff display
└── HistoryCells      # Conversation history
```

### Event System

```rust
pub enum AppEvent {
    // Core events
    CodexEvent(Event),
    NewSession,
    ExitRequest,

    // UI events
    StartFileSearch(String),
    FileSearchResult { query: String, matches: Vec<FileMatch> },

    // Animation events
    StartCommitAnimation,
    CommitTick,

    // State updates
    UpdateModel(String),
    UpdateReasoningEffort(Option<ReasoningEffort>),
}
```

### Rendering Pipeline

1. **Event Processing**: Handle keyboard/mouse events
2. **State Update**: Update application state
3. **Widget Calculation**: Calculate widget layouts
4. **Rendering**: Render to terminal buffer
5. **Swap Buffers**: Display to user

## Security Architecture

### Linux Sandboxing

The system implements defense-in-depth with multiple security layers:

#### Landlock (Filesystem)

```rust
fn apply_landlock_rules(writable_roots: Vec<PathBuf>) {
    let ruleset = Ruleset::default()
        .handle_access(AccessFs::from_all(ABI::V5))
        .create()
        // Read-only access to entire filesystem
        .add_rules(path_beneath_rules(&["/"], AccessFs::from_read()))
        // Write access only to specified directories
        .add_rules(path_beneath_rules(&writable_roots, AccessFs::from_all()))
        .restrict_self();
}
```

#### Seccomp (System Calls)

```rust
fn install_network_filter() {
    let filter = seccomp_filter! {
        // Allow essential syscalls
        A32/A64: read, write, close, exit, exit_group,

        // Block network syscalls
        A32/A64: socket => ERRNO(EPERM),
        A32/A64: connect => ERRNO(EPERM),
        A32/A64: bind => ERRNO(EPERM),
    };
    filter.install();
}
```

### Execution Policies

```rust
pub enum SandboxPolicy {
    None,                    // No restrictions
    ReadOnlyFilesystem,     // Can't write files
    WritableCurrentDirectory, // Write only in CWD
    WritableRoots(Vec<PathBuf>), // Specific directories
}

pub enum NetworkPolicy {
    Full,      // Unrestricted network
    Disabled,  // No network access
}
```

## State Management

### Three-Tier State Architecture

#### 1. Session State (Persistent)
```rust
pub struct SessionState {
    pub approved_commands: HashSet<Vec<String>>,
    pub history: ConversationHistory,
    pub token_info: Option<TokenUsageInfo>,
    pub latest_rate_limits: Option<RateLimitSnapshot>,
}
```

#### 2. Turn State (Per Interaction)
```rust
pub struct TurnState {
    pub pending_approvals: Vec<PendingApproval>,
    pub active_tools: IndexMap<String, ToolExecution>,
    pub reasoning_effort: Option<ReasoningEffort>,
}
```

#### 3. Task State (Per Tool)
```rust
pub struct TaskState {
    pub id: String,
    pub status: TaskStatus,
    pub output_buffer: Vec<u8>,
    pub start_time: Instant,
}
```

## MCP Integration

### Client Architecture

```rust
pub struct McpClient {
    child: tokio::process::Child,
    outgoing_tx: mpsc::Sender<JSONRPCMessage>,
    pending: Arc<Mutex<HashMap<i64, PendingSender>>>,
}

impl McpClient {
    // Initialize stdio-based MCP server
    pub async fn new_stdio_client(
        program: OsString,
        args: Vec<OsString>,
    ) -> Result<Self>;

    // Call MCP tool
    pub async fn call_tool(
        &self,
        tool_name: String,
        arguments: Value,
    ) -> Result<Value>;
}
```

### Protocol Implementation

```rust
pub trait ModelContextProtocolRequest {
    const METHOD: &'static str;
    type Params: DeserializeOwned + Serialize;
    type Result: DeserializeOwned + Serialize;
}

// Example: ListTools request
pub struct ListTools;
impl ModelContextProtocolRequest for ListTools {
    const METHOD: &'static str = "tools/list";
    type Params = ();
    type Result = Vec<Tool>;
}
```

## Performance Optimizations

### 1. Streaming Architecture
- Tool output streamed in real-time
- Incremental rendering for TUI
- Chunked file operations

### 2. Caching Strategies
- File search index caching
- Compiled regex patterns
- Session state persistence

### 3. Resource Management
- Circular buffers for output (128 KiB limit)
- Connection pooling for API requests
- Lazy initialization of expensive resources

### 4. Concurrency
- Parallel tool execution where safe
- Async I/O throughout
- Lock-free data structures where possible

## Error Handling

### Error Types

```rust
#[derive(Error, Debug)]
pub enum CodexErr {
    #[error("stream disconnected: {0}")]
    Stream(String, Option<Duration>),

    #[error("conversation not found: {0}")]
    ConversationNotFound(ConversationId),

    #[error("execution timeout")]
    Timeout,

    #[error("sandbox violation: {0}")]
    SandboxViolation(String),

    #[error("API error: {0}")]
    Api(#[from] ApiError),
}
```

### Error Propagation

- Use `Result<T, CodexErr>` for fallible operations
- `anyhow::Result` for application-level errors
- `thiserror` for structured error types
- Graceful degradation for non-critical failures

## Testing Strategy

### Unit Tests
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_tool_validation() {
        let tool = create_bash_tool();
        let input = json!({"command": "ls"});
        assert!(validate_tool_input(&tool, &input).is_ok());
    }
}
```

### Integration Tests
- End-to-end command execution
- TUI interaction testing
- MCP protocol compliance

### Snapshot Testing
```rust
#[test]
fn test_tui_rendering() {
    let app = create_test_app();
    let output = render_to_string(&app);
    insta::assert_snapshot!(output);
}
```

## Development Guidelines

### Code Style
- Rust 2024 edition
- No `unwrap()` or `expect()` - all errors handled
- Import granularity: `Item` level
- Maximum line length: 120 characters

### Commit Conventions
- Conventional commits format
- Sign-off required for contributions
- Atomic commits with single purpose

### Documentation
- Doc comments for public APIs
- Examples in documentation
- Architecture decision records (ADRs)

## Future Considerations

### Planned Improvements
1. **Plugin System**: Dynamic tool loading
2. **Distributed Execution**: Multi-machine support
3. **Enhanced Caching**: Persistent result cache
4. **WebAssembly Tools**: WASM-based tool execution
5. **GPU Acceleration**: For model inference

### Technical Debt
- Migration from ratatui fork to upstream
- Standardize error handling across crates
- Improve test coverage (target: 80%)
- Performance profiling and optimization

## Conclusion

The Codex-rs codebase represents a sophisticated, production-ready implementation of an AI-powered coding assistant. Its architecture emphasizes:

- **Security**: Multi-layered sandboxing and validation
- **Performance**: Async throughout with streaming
- **Extensibility**: Plugin-ready tool system
- **Reliability**: Comprehensive error handling
- **User Experience**: Responsive TUI with rich features

The codebase serves as an excellent example of enterprise Rust development, demonstrating advanced patterns and best practices throughout.