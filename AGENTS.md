# AGENTS.md

This file contains guidelines for agentic coding assistants working in this repository.

## Build, Test, and Lint Commands

### Build Commands
```bash
# Build and install development binaries (dgranted, dassumego)
make cli

# Build Go binary only (outputs to ./bin/dgranted)
make go-binary

# Build for all platforms (Linux, macOS, Windows)
make ci-cli-all-platforms

# Remove installed dev binaries
make clean
```

### Test Commands
```bash
# Run all tests
go test ./...

# Run a single test
go test -run TestName ./pkg/path/to/package

# Run tests with verbose output
go test -v ./...
```

### Lint Commands
```bash
# Basic Go vetting
go vet ./...

# Run golangci-lint
golangci-lint run --timeout 5m
```

### Debug Commands
```bash
# Run with debug logging
GRANTED_LOG=debug dgranted <command>

# Debug assume in IDE with FORCE_ASSUME_CLI env var
# See CONTRIBUTING.md for full launch configuration
```

## Code Style Guidelines

### Imports
- Group imports with blank lines between: standard library, then external packages
- Import third-party packages using full path (e.g., `github.com/aws/aws-sdk-go-v2/aws`)
- Alias imports for clarity when needed (e.g., `awshttp "github.com/aws/aws-sdk-go-v2/aws/transport/http"`)

### Formatting
- Use standard Go formatting (`go fmt`)
- Use tabs for indentation
- Maximum line length not strictly enforced, but keep readable (~100-120 chars)

### Types and Naming
- **Structs/Interfaces**: CamelCase with descriptive names (e.g., `AwsSsoAssumer`, `Launcher`)
- **Interfaces**: Often end with `-er` or `-or` suffix (e.g., `Assumer`, `Exporter`)
- **Functions**: CamelCase for exported, lowercase for unexported
- **Constants**: CamelCase (e.g., `DefaultRegion`)
- **Variables**: camelCase
- **Packages**: lowercase, single word preferred (e.g., `cfaws`, `config`)

### Error Handling
- Always check and return errors, never ignore them
- Wrap errors with context using `errors.Wrap()` from `github.com/pkg/errors`
- Define custom error types with `Error()` string method and optional `Unwrap()` method
  ```go
  type NoAccessError struct {
      Err error
  }
  func (e NoAccessError) Error() string { ... }
  func (e NoAccessError) Unwrap() error { return e.Err }
  ```
- Use `errors.As()` for type assertions on errors
- Return early on errors to avoid deep nesting

### Logging and Output
- Use `clio` library for all user-facing output (not `fmt.Printf`)
  ```go
  clio.Info("message")
  clio.Warn("message")
  clio.Success("message")
  clio.Error("message")
  clio.Debugf("debug message", "key", value)
  ```
- Important: Output to stderr, not stdout (stdout reserved for credential export)
- For console/browser commands, prefer `clio.Infof`/`clio.Successf` for formatted messages

### Comments
- Package comments at top of file explaining purpose
- Exported functions and types must have comments
- Use `// TODO:` for future work
- Keep comments concise and focused on "why" not "what"

### Architecture Patterns
- **Single binary, dual behavior**: One binary switches based on `argv[0]` (granted vs assume)
- **Interface-driven design**: Use interfaces for extensibility (e.g., `Assumer` interface)
- **Context usage**: Pass `context.Context` to AWS operations for cancellation
- **Configuration**: Load using `config.Load()` which returns `*Config` struct
- **AWS profiles**: Use `cfaws` package to load and initialize profiles
- **Credential storage**: Use `securestorage` package, never store plaintext tokens

### Testing
- Use standard Go testing (`testing` package)
- Table-driven tests for multiple test cases
- Name test functions `Test<FunctionName>` (e.g., `TestExpandRegion`)
- Use subtests with `t.Run()` for table-driven tests
- Use `github.com/stretchr/testify/assert` for assertions
  ```go
  for _, tt := range tests {
      t.Run(tt.name, func(t *testing.T) {
          got, err := FunctionUnderTest(tt.args)
          assert.Equal(t, tt.want, got)
          if (err != nil) != tt.wantErr {
              t.Errorf("error = %v, wantErr %v", err, tt.wantErr)
          }
      })
  }
  ```

### AWS SDK v2 Patterns
- Use `aws.NewConfig()` to create AWS service clients
- Pass context to all AWS operations: `client.Operation(ctx, input)`
- Use pointer helpers: `aws.String("value")`, `aws.Int(42)`
- Use service-specific options pattern: `sts.New(sts.Options{Region: region})`
- Handle errors with smithy types: `serr, ok := err.(*smithy.OperationError)`
- Check HTTP status codes via `awshttp.ResponseError`
- Credential providers use caching: `aws.NewCredentialsCache(provider)`

### Launcher/Browser Pattern
- Implement `Launcher` interface for browser integration
- Return `[]string` for command and args (e.g., `[]string{"firefox", "--new-tab", url}`)
- Implement `UseForkProcess()` - return false for `open` commands, true for direct executables
- Use `forkprocess` library for browser launching when UseForkProcess returns true
- Custom launchers use Go templates: `text/template` with `{{.Profile}}`, `{{.URL}}` variables
- Parse template commands: `template.New("").Parse(commandTemplate)`

### Configuration Patterns
- TOML config files parsed via `github.com/BurntSushi/toml`
- Use struct tags for TOML mapping: `toml:"name,omitempty"`
- Default config with `config.NewDefaultConfig()` - OS-specific defaults
- Load config with `config.Load()` returns `*Config` struct
- Save config with `(*Config).Save()` writes to Granted config path
- Support XDG directories: `XDG_CONFIG_HOME`, `XDG_CACHE_HOME`, `XDG_STATE_HOME`

### Assumer Pattern
- Implement `Assumer` interface for credential sources (AWS SSO, IAM, GCP, etc.)
- Methods: `AssumeTerminal`, `AssumeConsole`, `Type()`, `ProfileMatchesType()`
- Register assumers in priority order via `cfaws.RegisterAssumer()`
- Specific types first (SSO, Google Auth), generic types last (IAM)
- Use `errors.As()` for type assertions on errors
- Define custom error types: `NoAccessError` with `Error()` and `Unwrap()` methods

### CI/CD
- GitHub Actions runs on `push`
- Go version: 1.22.1
- Steps: checkout, setup go, build all platforms, lint (`go vet`), test (`go test ./...`)
- Separate `golangci-lint` job with 5m timeout
- Shellcheck job validates shell scripts
- Binaries cross-compiled: linux, macos, windows

### Key Libraries
- `github.com/urfave/cli/v2` - CLI framework
- `github.com/aws/aws-sdk-go-v2/*` - AWS SDK v2
- `github.com/common-fate/clio` - Structured logging/output
- `gopkg.in/ini.v1` - INI file parsing (AWS config files)
- `github.com/AlecAivazis/survey/v2` - Interactive prompts
- `github.com/pkg/errors` - Error wrapping (`errors.Wrap()`)
- `github.com/BurntSushi/toml` - TOML parsing
- `github.com/aws/smithy-go` - AWS SDK error types
- `github.com/fatih/color` - Terminal colors
- `github.com/stretchr/testify` - Testing assertions

### Development Notes
- Development binaries use `d` prefix: `dgranted`, `dassumego`, `dassume`
- Production binaries: `granted`, `assumego`, `assume`
- Use `FORCE_ASSUME_CLI=true` env var to debug assume in IDE
- Shell scripts (`scripts/assume`, etc.) wrap `dassumego` to export environment variables
- Tests live alongside source files (e.g., `region_test.go` with `region.go`)
- Output goes to stderr (via clio), stdout reserved for credential export
