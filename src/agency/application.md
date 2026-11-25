# agency

Implement the application package here.

Required top-level modules:
- `cli.py` - scope approval and paused-run operator commands
- `contracts.py` - strict Pydantic v2 boundary models
- `model.py` - live Ollama and scripted model adapters
- `db.py` - SQLite domain transactions and expected-revision updates

One model adapter must validate structured output and allow one format retry. State changes must happen only after successful validation.
