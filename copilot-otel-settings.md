# Copilot OTel — settings

Copilot CLI 1.0.80 · VS Code 1.134.0 · Copilot Chat 0.62.0

## VS Code — `~/Library/Application Support/Code/User/settings.json`

```json
{
    "github.copilot.chat.otel.enabled": true,
    "github.copilot.chat.otel.exporterType": "otlp-http",
    "github.copilot.chat.otel.protocol": "http/protobuf",
    "github.copilot.chat.otel.otlpEndpoint": "http://localhost:4318",
    "github.copilot.chat.otel.captureContent": true,
    "github.copilot.chat.otel.maxAttributeSizeChars": 0,
    "github.copilot.chat.otel.serviceName": "copilot-vscode"
}
```

Requires a window reload. Other keys: `resourceAttributes` (object), `headers` (object),
`outfile`, `dbSpanExporter.enabled`. `exporterType`: `otlp-grpc|otlp-http|console|file`.
`protocol`: `''|http/json|http/protobuf|grpc`.

## Copilot CLI — environment

```bash
export COPILOT_OTEL_ENABLED=true
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
export OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true
export OTEL_SERVICE_NAME=copilot-cli
export OTEL_RESOURCE_ATTRIBUTES="deployment.environment=local"
```

Others: `COPILOT_OTEL_EXPORTER_TYPE` (`otlp-http|file`), `COPILOT_OTEL_FILE_EXPORTER_PATH`,
`COPILOT_OTEL_SOURCE_NAME`, `OTEL_EXPORTER_OTLP_HEADERS`, `OTEL_LOG_LEVEL`,
`OTEL_EXPORTER_OTLP_CERTIFICATE`, `OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE`,
`OTEL_EXPORTER_OTLP_CLIENT_KEY`.


