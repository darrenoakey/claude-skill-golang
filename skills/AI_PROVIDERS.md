# AI Provider SDKs (Native Go)

All major AI providers now have official Go SDKs. Use these for any Go app that needs AI — never shell out to CLI tools.

| Provider | Package | Structured Output | Streaming |
|----------|---------|-------------------|-----------|
| **Claude** | `github.com/anthropics/anthropic-sdk-go` | `JSONOutputFormatParam` with JSON schema | `client.Messages.NewStreaming()` |
| **OpenAI** | `github.com/openai/openai-go` | `ResponseFormatJSONSchemaParam` with `Strict: true` | `client.Chat.Completions.NewStreaming()` |
| **Gemini** | `google.golang.org/genai` | `ResponseJsonSchema` + `ResponseMIMEType: "application/json"` | `GenerateContentStream()` returns `iter.Seq2` |
| **Ollama** | Raw HTTP to `localhost:11434` | `"format": <json_schema>` in chat request | Line-delimited JSON chunks |

## Key Patterns

- All use native structured output (JSON schema enforcement) — never use prompt injection for structured output
- Anthropic/OpenAI streaming use `ssestream.Stream` with `Next()`/`Current()`/`Err()` pattern
- Gemini streaming uses Go 1.22+ range-over-func (`iter.Seq2`)
- Ollama raw HTTP is preferred over `github.com/ollama/ollama/api` which drags in the entire Ollama binary dependency tree
- Auth: SDKs auto-read `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`/`GOOGLE_API_KEY`
- System messages: Anthropic uses separate `System` param (not in messages array); Gemini uses `SystemInstruction`; OpenAI/Ollama include in messages array
- OpenAI SDK `apierror.Error.Error()` panics when `.Request`/`.Response` are nil — use `errors.As` before calling `.Error()` in error classification
- Reference implementation: `daz-agent-sdk/go/provider/` has working providers for all four
