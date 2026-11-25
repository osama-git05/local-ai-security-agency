# scripts

Operator-facing and verification scripts live here.

Create:
- `smoke_model.py` - ten-case live Ollama smoke test and `var/gates/level0.json` output
- `benchmark.py` - 30-case single-pass versus verified comparison
- `demo.py` - `verify` and `dashboard` subcommands

The scripts must use the same configuration and model settings where a comparison depends on them. Separate live failures from scripted routing tests.
