---
name: golang
description: Go development standards and practices for zero-fabrication, test-driven development with strict quality gates. Use when working on Go projects that require rigorous testing, real integrations only, and idiomatic Go patterns.
---

# Go Development Skill

**Zero-Fabrication | Test-Driven | Zero-Tolerance**

This skill defines the mandatory standards for Go development. These rules are **non-negotiable** and enforced to ensure consistency, maintainability, and correctness.

## The Golden Rules

1.  **Real Tests Only**: No mocks, no fakes. Tests must hit actual systems (DB, File, API).
2.  **Co-located Tests**: Every `.go` file has a `_test.go` beside it.
3.  **Immutability Preferred**: Unexported fields, constructor functions, value semantics where practical.
4.  **One Responsibility Per File**: Files are small and focused. File name describes the single concept.
5.  **Zero Warnings**: `go vet`, `golangci-lint`, and `gofmt` must be clean. No exceptions.
6.  **"HOW" vs "WHAT"**: 95% of code in `pkg/` (reusable utilities), 5% in `cmd/` (entry points).
7.  **Document Everything**: Go doc comments on ALL exported types, functions, methods, and constants.

## Documentation & Standards

-   **[CODE_RULES.md](skills/CODE_RULES.md)**: Coding standards, naming, style, error handling, and prohibited patterns.
-   **[ARCH_RULES.md](skills/ARCH_RULES.md)**: Project structure, file organization, and package layout.
-   **[TEST_RULES.md](skills/TEST_RULES.md)**: Testing philosophy, patterns, and mandatory practices.
-   **[SETUP.md](skills/SETUP.md)**: Project initialization, `run` script template, and configuration.
-   **[EXAMPLES.md](skills/EXAMPLES.md)**: Reference guide to the included example files.
-   **[AI_PROVIDERS.md](skills/AI_PROVIDERS.md)**: Native Go SDKs for Claude, OpenAI, Gemini, Ollama.
-   **[GIO_UI.md](skills/GIO_UI.md)**: Gio GUI framework — core concepts, then sub-files: [GIO_GOTCHAS.md](skills/GIO_GOTCHAS.md), [GIO_PATTERNS.md](skills/GIO_PATTERNS.md), [GIO_PLATFORM.md](skills/GIO_PLATFORM.md).
-   **[GOTCHAS.md](skills/GOTCHAS.md)**: Runtime pitfalls, library quirks, and hard-won lessons (Gio, chromedp, SQL, CGO, etc.).

## Repository & Project Layout

**Required structure**
```
README.md
.gitignore
run                  # Shell script facade (executable)
src/                 # Go module root (go.mod lives here)
  go.mod
  go.sum
  cmd/               # Entry points (WHAT) ~5%
    myapp/
      main.go
  pkg/               # Reusable packages (HOW) ~95%
    text/
      truncate.go
      truncate_test.go
    net/
      client.go
      client_test.go
  internal/          # Private packages (not importable externally)
output/              # Gitignored runtime outputs
  bin/               # Compiled binaries
  testing/           # Test output/logs/artifacts
local/               # Gitignored large downloads/artifacts
```

**Directory constraints**
- Shallow structure preferred. Once you use subdirectories, place peer packages at the same nesting level.
- Max 20 files per directory. Split into sub-packages if larger.
- No experimental scripts or alt versions at root or anywhere else; use branches for iteration.
- **output/ and local/ must be accessed relative to the project root, not the current working directory.**
- `go.mod` lives inside `src/` so import paths stay clean (no `src/` prefix in imports).
- The `run` script handles all directory navigation internally.

---

## Run Facade (Shell Script)

The `run` tool is a **shell script** named `run` (no extension, executable). It orchestrates by invoking Go toolchain commands. It must not contain business logic.

**Callers never cd into src/** - they just call `./run` or `~/bin/projectname`. The `run` script navigates to `src/` internally.

**~/bin wrapper script** - when creating a `run` for a project, also create a wrapper script in `~/bin/` so the tool is globally accessible:

```bash
# ~/bin/myproject (2 lines, executable)
#!/bin/bash
exec ~/src/myproject/run "$@"
```

This means:
- `~/src/myproject/run` is the real implementation
- `~/bin/myproject` is the global entry point that delegates to it
- Users call `myproject check` from anywhere, never needing to cd

**See [SETUP.md](skills/SETUP.md) for the complete `run` script template.**

---

## Secrets & Configuration

**Secrets**
- Secrets come only from **keyring** (via `zalando/go-keyring` or similar) or **environment variables** for deployment.
- The literal word `keyring` must **never appear in tests**.
- If secret retrieval fails, return the error; do not stub or override.

**Configuration**
- Internal apps: encode base configuration in code as constants or package-level vars.
- Environment-specific values via environment variables only.
- Use a single `config` package to centralize env var reading.

---

## Development Workflow

**For each file change:**
1. Write or modify code following the standards in [CODE_RULES.md](skills/CODE_RULES.md)
2. Run formatter: `gofmt -w .` (from `src/`)
3. Run vet: `go vet ./pkg/text/`
4. Run tests for that specific package: `go test ./pkg/text/ -run TestTruncate -count=1 -v`
5. Fix any issues and repeat steps 2-4

**At task completion:**
1. Run `./run check` (full quality gate suite: lint + all tests)
2. All tests, vet, lint, and `gofmt` must be clean
3. **No commits** without full `./run check` pass

**If debugging a runtime issue or library quirk**, read [GOTCHAS.md](skills/GOTCHAS.md).

**IF YOU VIOLATE THESE RULES, YOU WILL FAIL.**
