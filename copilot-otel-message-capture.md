# GitHub Copilot Chat OTel — settings required for message capture

Verified 2026-09-23 with VS Code 1.139.0 and the bundled Copilot Chat 0.67.0, exporting to a local
OpenTelemetry Collector (contrib 0.159.0) with a file sink.

Symptom this addresses: telemetry arrives (spans, metrics, log events with model and token counts)
but prompts and responses never appear.

## 1. VS Code `settings.json` (user scope)

```json
{
  "github.copilot.chat.otel.enabled": true,
  "github.copilot.chat.otel.exporterType": "otlp-http",
  "github.copilot.chat.otel.protocol": "http/protobuf",
  "github.copilot.chat.otel.otlpEndpoint": "http://localhost:4318",
  "github.copilot.chat.otel.captureContent": true,
  "github.copilot.chat.otel.maxAttributeSizeChars": 0,
  "github.copilot.chat.otel.serviceName": "copilot-vscode",
  "github.copilot.chat.otel.headers": { "Authorization": "Bearer <collector token>" }
}
```

- `captureContent: true` is the setting that puts messages on the spans. Default is `false`. Without it you
  get spans, metrics, and log events with token counts but no `gen_ai.input.messages` or
  `gen_ai.output.messages`.
- `maxAttributeSizeChars: 0` means no truncation. Set a positive number only if your backend caps attribute size.
- `exporterType` must be `otlp-http` or `otlp-grpc`. Do not use `file`; it writes spans as `{}`.
- `headers` is only needed if the collector requires auth. Drop it for an open collector.
- Reload the window after changing any of these. They are read once at extension activation.

## 2. Environment of the shell VS Code resolves at startup

These must NOT be set, or they override the settings above:

- `COPILOT_OTEL_CAPTURE_CONTENT` (if present and `false`, it wins over the setting)
- `OTEL_EXPORTER_OTLP_ENDPOINT` / `COPILOT_OTEL_ENDPOINT` (redirects export away from your endpoint)
- `OTEL_EXPORTER_OTLP_PROTOCOL` (if `grpc`, breaks the HTTP exporter)
- `COPILOT_OTEL_FILE_EXPORTER_PATH` (forces the broken file exporter)

VS Code injects the login-shell environment into the extension host regardless of how it was launched
(Dock, `open -a`, or terminal). If your `.zshrc` or `.bashrc` exports any of these for another tool,
guard them. VS Code sets `VSCODE_RESOLVING_ENVIRONMENT=1` in the shell it uses for resolution:

```zsh
if [[ -z "$VSCODE_RESOLVING_ENVIRONMENT" ]]; then
  export OTEL_EXPORTER_OTLP_ENDPOINT=http://<other-collector>:4317
  export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
fi
```

## 3. No managed settings touching OTel

Resolution order inside the extension:

```
captureContent = managed/policy value
              ?? env COPILOT_OTEL_CAPTURE_CONTENT
              ?? user setting
              ?? false
```

If any `github.copilot.chat.otel.*` key appears in a managed-settings source, the extension ignores every
personal OTel setting and uses the managed values only. If the managed block sets an endpoint but not
`captureContent`, capture is off and your `true` is silently ignored. Either remove the OTel keys from
managed settings, or add `"github.copilot.chat.otel.captureContent": true` to the managed block itself.

Managed-settings sources VS Code reads:

- `/etc/github-copilot/managed-settings.json` (Linux)
- `/Library/Application Support/GitHubCopilot/managed-settings.json` (macOS)
- The GitHub organization / enterprise Copilot policy tied to the signed-in account

Check with Command Palette → **Show Policy Diagnostics**.

## 4. Collector: traces pipeline must exist and reach a trace store

Messages travel only as span attributes. Log records and metrics never carry them, even with
`captureContent` on.

| Signal | Carries message content |
|---|---|
| Traces: `chat *`, `invoke_agent`, `execute_tool` span attributes | Yes: `gen_ai.input.messages`, `gen_ai.output.messages`, `gen_ai.system_instructions`, `gen_ai.tool.definitions`, `gen_ai.tool.call.arguments`, `gen_ai.tool.call.result` |
| Logs: `gen_ai.client.inference.operation.details`, `copilot_chat.*` events | No. Model, response id, finish reasons, token counts only |
| Metrics: `copilot_chat.*`, `gen_ai.client.*` | No |

The collector needs a `traces` pipeline with an exporter that stores spans:

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 2s

exporters:
  file:
    path: /otel-live/otel-live.jsonl
  # or otlp/tempo, jaeger, etc.

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [file]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [file]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [file]
```

A metrics-only or logs-only pipeline is the most common way to end up with telemetry but no messages.
Also make sure there is no `attributes`, `transform`, or `redaction` processor in the traces pipeline that
strips `gen_ai.*` keys.

## 5. Backend attribute size limit

Prompts are 5 KB to 200 KB per attribute. If the trace store caps attribute size, the message attributes
get dropped or truncated while everything else looks fine. For Grafana Tempo, raise `max_span_attr_byte`
in overrides (default 2048), or set `maxAttributeSizeChars` in VS Code to something under the cap.

## Verify

After a window reload, send one chat message, then confirm the resolved config in the
**GitHub Copilot Chat** output log:

```
[OTel] Instrumentation enabled — exporter=otlp-http endpoint=http://localhost:4318/ captureContent=true
```

If that line says `captureContent=false`, the problem is in sections 1 to 3. If it says `true` and the
backend still has no messages, the problem is in sections 4 or 5.

Quick check against a file sink:

```bash
jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?
       | select(.name | startswith("chat"))
       | .attributes[] | select(.key=="gen_ai.input.messages")
       | .value.stringValue | length' otel-live.jsonl
```

Any positive number means content is being captured.
