# evidence

Evidence intake and isolated reading belong here.

Create:
- `intake.py` - metadata capture, 1 MiB text limit, binary/symlink/outside-folder rejection
- `store.py` - SHA-256 hashing, temporary-file write, atomic publish, and SQLite metadata
- `reader.py` - one-based inclusive line citation and engagement/asset/digest/range/quote validation
- `mcp_server.py` - stdio MCP `read_excerpt(evidence_id, first_line, last_line)` service

Bind the engagement in the parent process. The reader must have read-only evidence access and no target network access.
