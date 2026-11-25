# roles

Department role implementations belong here.

Create:
- `tester.py` - narrow candidate claims with exact citations
- `verifier.py` - evidence-first adversarial verification
- `writer.py` - structured report writing from an allowlist of confirmed findings

The verifier is read-only. Application code, not the model, generates finding IDs, allowed tools, and routing decisions.
