---
name: golang
description: Go development standards for zero-fabrication, test-driven work with strict quality gates. Use when writing or modifying any Go project — enforces co-located real integration tests, `go vet`/`golangci-lint`/`gofmt` clean, cmd/pkg layout, the `run` shell facade, and idiomatic Go. Load the referenced `skills/*.md` for detailed rules and `GOTCHAS.md` when debugging.
---

# Go Development Skill

**Zero-Fabrication | Test-Driven | Zero-Tolerance**

Mandatory Go standards. Non-negotiable, enforced for consistency, maintainability, and correctness. **IF YOU VIOLATE THESE RULES, YOU WILL FAIL.**

## The Golden Rules

1. **Real Tests Only** — no mocks, no fakes. Tests hit actual systems (DB, file, API).
2. **Co-located Tests** — every `.go` file has a `_test.go` beside it.
3. **Immutability Preferred** — unexported fields, constructor functions, value semantics where practical.
4. **One Responsibility Per File** — small and focused; the file name describes the single concept.
5. **Zero Warnings** — `go vet`, `golangci-lint`, and `gofmt` must be clean. No exceptions.
6. **"HOW" vs "WHAT"** — ~95% of code in `pkg/` (reusable utilities), ~5% in `cmd/` (entry points).
7. **Document Everything** — Go doc comments on ALL exported types, functions, methods, and constants.
8. **25-Line Function Limit** — no function body exceeds 25 lines. Factor into smaller, meaningfully-named functions. No exceptions.

## Documentation & Standards

- **[CODE_RULES.md](skills/CODE_RULES.md)** — coding standards, naming, style, error handling, prohibited patterns.
- **[ARCH_RULES.md](skills/ARCH_RULES.md)** — project structure, file organization, package layout.
- **[TEST_RULES.md](skills/TEST_RULES.md)** — testing philosophy, patterns, mandatory practices.
- **[SETUP.md](skills/SETUP.md)** — project initialization, `run` script template, configuration.
- **[EXAMPLES.md](skills/EXAMPLES.md)** — reference guide to the included example files.
- **[AI_PROVIDERS.md](skills/AI_PROVIDERS.md)** — native Go SDKs for Claude, OpenAI, Gemini, Ollama.
- **[GIO_UI.md](skills/GIO_UI.md)** — Gio GUI framework core concepts; sub-files: [GIO_GOTCHAS.md](skills/GIO_GOTCHAS.md), [GIO_PATTERNS.md](skills/GIO_PATTERNS.md), [GIO_PLATFORM.md](skills/GIO_PLATFORM.md), [GIO_WINDOW_POSITION.md](skills/GIO_WINDOW_POSITION.md).
- **[GOTCHAS.md](skills/GOTCHAS.md)** — runtime pitfalls, library quirks, hard-won lessons (Gio, chromedp, SQL, CGO, etc.). Read when debugging a runtime issue or library quirk.

---

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
- No experimental scripts or alt versions anywhere; use branches for iteration.
- **Access `output/` and `local/` relative to the project root, never the current working directory.**
- `go.mod` lives inside `src/` so import paths stay clean (no `src/` prefix in imports).
- The `run` script handles all directory navigation internally.

---

## Run Facade (Shell Script)

`run` is a **shell script** named `run` (no extension, executable). It orchestrates by invoking Go toolchain commands and must contain no business logic. Callers never `cd` into `src/` — they call `./run` or `~/bin/projectname`; `run` navigates to `src/` internally.

**`~/bin` wrapper** — also create a global-access wrapper:
```bash
# ~/bin/myproject (executable, 2 lines)
#!/bin/bash
exec ~/src/myproject/run "$@"
```

`~/src/myproject/run` is the real implementation; `~/bin/myproject` is the global entry point delegating to it. Users run `myproject check` from anywhere, never needing to `cd`.

**See [SETUP.md](skills/SETUP.md) for the complete `run` script template.**

---

## Secrets & Configuration

**Secrets**
- Only from **keyring** (via `zalando/go-keyring` or similar) or **environment variables** for deployment.
- The literal word `keyring` must **never appear in tests**.
- If secret retrieval fails, return the error; do not stub or override.

**Configuration**
- Internal apps: encode base configuration in code as constants or package-level vars.
- Environment-specific values via environment variables only.
- Centralize env var reading in a single `config` package.

---

## Development Workflow

**For each file change:**
1. Write/modify code following [CODE_RULES.md](skills/CODE_RULES.md).
2. Format: `gofmt -w .` (from `src/`).
3. Vet: `go vet ./pkg/text/`.
4. Run that package's tests: `go test ./pkg/text/ -run TestTruncate -count=1 -v`.
5. Fix and repeat 2–4 until clean.

**At task completion:**
1. `./run check` — full quality gate (lint + all tests).
2. All tests, `go vet`, lint, and `gofmt` must be clean.
3. **No commits** without a full `./run check` pass.
