# fixtures/demo

Contains the stable synthetic originals used throughout the build.

Required demo bytes:
- `response.txt`
  - line 1: `HTTP/1.1 403 Forbidden`
  - line 2: `X-Request-ID: demo-017`
- `config.txt`
  - line 1: `application=demo`
  - line 2: `debug_mode=true`

All files must be UTF-8 and LF-terminated. Preserve exact bytes. Seed claims C1, C2, and C3 against these captures and add a separate instruction-to-confirm fixture for prompt-injection testing.
