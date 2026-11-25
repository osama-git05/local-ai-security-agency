# config

Local configuration for the agency.

Create:
- `runtime.toml` - Ollama endpoint/model, 2048-token context, 384-token generation limit, temperature zero, one in-flight generation, and 120-second timeout.
- `scope.toml` - the approved lab scope, including asset ID `juice-local` and a human-readable lab description.

Configuration values must be explicit and reproducible. Do not store secrets or real engagement data here.
