# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Granted is a CLI tool for simplifying access to AWS cloud roles. It enables fast role assumption, opening multiple AWS accounts in the browser simultaneously, and encrypted credential caching. The project is maintained at `github.com/fwdcloudsec/granted`.

## Build & Development Commands

```bash
# Build and install development binaries (dgranted, dassume)
make cli

# Build Go binary only (outputs to ./bin/dgranted)
make go-binary

# Build for all platforms (Linux, macOS, Windows)
make ci-cli-all-platforms

# Remove installed dev binaries
make clean

# Run all tests
go test ./...

# Run a single test
go test -run TestName ./pkg/path/to/package

# Lint
go vet ./...
golangci-lint run --timeout 5m

# Debug output for AWS credentials (useful for testing)
make aws-credentials
```

## Architecture

### Single Binary, Dual Personality

The project uses **one Go binary** (`cmd/granted/main.go`) that switches behavior based on `argv[0]`:

- **granted/dgranted**: Manages Granted configuration (browser settings, SSO config, credentials)
- **assumego/dassumego**: Assumes AWS roles (wrapped by shell scripts to export env vars)

The routing logic is in `cmd/granted/main.go:34`:
```go
switch filepath.Base(os.Args[0]) {
case "assumego", "assumego.exe", "dassumego", "dassumego.exe":
    app = assume.GetCliApp()
default:
    app = granted.GetCliApp()
}
```

### Why Shell Script Wrappers?

The `assume` command needs to export AWS credentials as environment variables to the calling shell. Since a subprocess cannot modify its parent's environment, `scripts/assume` (and variants for fish/tcsh/powershell) wraps the Go binary and evaluates its output to set env vars.

### Key Packages

| Package | Purpose |
|---------|---------|
| `pkg/granted/` | Main CLI commands for `granted` binary |
| `pkg/assume/` | Core role assumption logic for `assume` binary |
| `pkg/cfaws/` | AWS profile parsing, various assume methods (SSO, IAM, credential process, Azure, Google) |
| `pkg/config/` | State folder, browser templates, keyring configuration |
| `pkg/securestorage/` | Keyring-based encrypted credential storage |
| `pkg/idclogin/` | AWS IAM Identity Center (SSO) login flows |
| `pkg/browser/` | Browser launching and URL handling |
| `pkg/launcher/` | Platform-specific browser launcher abstractions |

### AWS Assumer Implementations

Different credential sources are handled by assumer implementations in `pkg/cfaws/`:
- `assumer_aws_sso.go` - AWS SSO/IAM Identity Center
- `assumer_aws_iam.go` - IAM credentials
- `assumer_aws_credential_process.go` - External credential process
- Plus integrations for Azure, Google, GimmeAWSCreds

## Development Notes

### Debug Logging

```bash
GRANTED_LOG=debug dgranted <command>
```

### Debugging assume in IDE

Use `FORCE_ASSUME_CLI=true` to force the binary into assume mode regardless of argv[0]:

```json
{
  "env": {
    "GRANTED_LOG": "debug",
    "FORCE_ASSUME_CLI": "true",
    "GRANTED_ALIAS_CONFIGURED": "true"
  }
}
```

### IO Pattern

Informational output goes to **stderr** (allows piping credentials via stdout). Use the `clio` logging library:

```go
clio.Info("message")
clio.Warn("message")
clio.Success("message")
clio.Error("message")
```

### Development vs Production Binaries

| Dev | Prod |
|-----|------|
| `dgranted` | `granted` |
| `dassumego` | `assumego` |
| `dassume` | `assume` |
