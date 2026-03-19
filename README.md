# Logging Package

This package provides a comprehensive, thread-safe logging system for the MariaDB Backup S3 application. It supports multiple output formats, structured logging with context, and configurable log levels.

## Features

- **Thread-safe operations** with mutex protection
- **Multiple formatters**: Human-readable console output with colors, JSON for machine processing
- **Structured logging** with fields, component names, and operation IDs
- **Context management** for tracking operations across function calls
- **Configurable log levels**: DEBUG, INFO, WARN, ERROR, FATAL
- **Multiple output destinations**: stdout, stderr, or files
- **Go standard library only** (no external logging dependencies)

## Quick Start

```go
import "github.com/go-core-fx/cli-logger"

// Create a default logger
defaultLogger := logger.NewDefault()

// Create a context with the logger and component
ctx := logger.WithLogger(context.Background(), defaultLogger)
ctx = logger.WithComponent(ctx, "my-component")

// Log messages
defaultLogger.Info(ctx, "Application started")
defaultLogger.Error(ctx, "Something went wrong", err, logger.Fields{
    "user_id": 123,
    "action": "login",
})
```

## Configuration

The logging system can be configured using the `Config` struct:

```go
config := logger.Config{
    Level:        logger.LogLevelInfo,
    Format:       "human",        // "human" or "json"
    Output:       os.Stdout,      // Use os.Stdout, os.Stderr, or any io.Writer
    EnableColors: true,
    TimeFormat:   "2006-01-02 15:04:05.000",
}

configuredLogger, err := logger.New(config)
if err != nil {
    // handle error
}
ctx := logger.WithLogger(context.Background(), configuredLogger)
```

## Log Levels

- `LogLevelDebug`: Detailed debugging information
- `LogLevelInfo`: General information messages
- `LogLevelWarn`: Warning messages
- `LogLevelError`: Error messages
- `LogLevelFatal`: Fatal errors that terminate the application

## Formatters

### Human Formatter

Console-friendly output with colors and structured fields:

```
2025-01-23 10:30:45.123 INFO  [backup] op=backup-123456789 Starting backup duration=2.5s
2025-01-23 10:30:47.456 ERROR [upload] Upload failed error=connection timeout filename=backup.tar.gz
```

### JSON Formatter

Machine-readable JSON output:

```json
{
  "timestamp": "2025-01-23T10:30:45.123Z",
  "level": "INFO",
  "message": "Starting backup",
  "component": "backup",
  "operation_id": "backup-123456789",
  "fields": {
    "duration": "2.5s"
  }
}
```

## Context Management

The logging system provides utilities for managing context across operations:

```go
// Add component name
ctx = logger.WithComponent(ctx, "database")

// Add operation ID
ctx = logger.WithOperationID(ctx, "backup-123")

// Add custom fields
ctx = logger.WithFields(ctx, logger.Fields{
    "table": "users",
    "rows": 1000,
})

// Generate operation ID
operationID := logger.GenerateOperationID("backup")
```

## Contextual Logging

To retrieve the logger from a context:

```go
retrievedLogger := logger.GetLogger(ctx)
if retrievedLogger == nil {
    // Handle missing logger (e.g., use a default)
    retrievedLogger = logger.NewDefault()
    ctx = logger.WithLogger(ctx, retrievedLogger)
}
```

You can add component and operation ID to the context:

```go
ctx = logger.WithComponent(ctx, "my-component")
ctx = logger.WithOperationID(ctx, "op-12345")
// or generate a new operation ID
opID := logger.GenerateOperationID("my-component")
ctx = logger.WithOperationID(ctx, opID)
```

Then log as usual:

```go
l := logger.GetLogger(ctx)
l.Info(ctx, "Message with component and operation ID")
```

## Integration Examples

### In Main Application

```go
func main() {
    // Initialize logging
    appLogger := logger.NewDefault()
    ctx := logger.WithLogger(context.Background(), appLogger)
    ctx = logger.WithComponent(ctx, "main")

    appLogger.Info(ctx, "Application starting")

    // Your application logic here...

    appLogger.Info(ctx, "Application completed")
}
```

### In Backup Operations

```go
func Execute(ctx context.Context, cfg Config) error {
    backupLogger := logger.GetLogger(ctx)
    operationID := logger.GenerateOperationID("backup")

    ctx = logger.WithOperationID(ctx, operationID)
    ctx = logger.WithComponent(ctx, "backup")

    backupLogger.Info(ctx, "Starting backup process")

    // Backup stages with individual components
    if err := backupStage(ctx, cfg); err != nil {
        backupLogger.Error(ctx, "Backup stage failed", err)
        return err
    }

    backupLogger.Info(ctx, "Backup completed successfully")
    return nil
}
```

## Environment Variables

The logging system can be configured via environment variables:

- `LOG_LEVEL`: Set log level (debug, info, warn, error, fatal). Default: info
- `LOG_FORMAT`: Set format (human, json). Default: human
- `LOG_OUTPUT`: Set output destination:
  - `stdout`: Standard output
  - `stderr` (default): Standard error
  - Any file path: Write logs to the specified file
- `NO_COLOR`: When set (any non-empty value), disables colored output for human format

Example usage:
```bash
# Set log level to debug and output to JSON format
export LOG_LEVEL=debug
export LOG_FORMAT=json
export LOG_OUTPUT=/var/log/mariadb-backup.log

# Disable colors in human format
export NO_COLOR=1
```

## Thread Safety

All logging operations are thread-safe and can be called concurrently from multiple goroutines. The implementation uses `sync.RWMutex` to protect internal state while allowing concurrent reads.

## Performance

The logging system is designed for high performance:

- Minimal allocations in hot paths
- Efficient field merging
- Lazy formatter initialization
- Context-aware logging to avoid unnecessary work

## Error Handling

The logging system handles errors gracefully:

- Formatter errors are logged to stderr and don't crash the application
- Invalid configurations return errors during initialization
- Missing context information falls back to sensible defaults

## Testing

The logging system is designed to be testable:

```go
func TestMyFunction(t *testing.T) {
    // Create an in-memory buffer for testing
    var buf bytes.Buffer

    config := logger.Config{
        Level:  logger.LogLevelDebug,
        Format: "json",
        Output: &buf, // Use an io.Writer like &bytes.Buffer{}
    }

    logger, err := logger.New(config)
    if err != nil {
        t.Fatal(err)
    }

    // Set the logger in the context
    ctx := logger.WithLogger(context.Background(), logger)

    // Use the logger in your tests
    logger.Info(ctx, "Test message")

    // Verify log output
    logOutput := buf.String()
    if !strings.Contains(logOutput, `"message":"Test message"`) {
        t.Error("Expected log message not found")
    }
}