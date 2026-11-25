# workflow

Persistent LangGraph orchestration and durable human decisions belong here.

Create:
- `state.py` - the lightweight `AgencyState` graph boundary
- `graph.py` - persistent SQLite checkpointer and stable engagement thread IDs
- `approvals.py` - scope, evidence, and report sign-off handlers
- `events.py` - ordered per-engagement audit events

Enforce two additional evidence rounds per finding. Use request IDs, expected revisions, idempotent transitions, and stale-revision conflicts.
