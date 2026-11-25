# Local-first AI Cybersecurity Agency
## Engineering skeleton and build blueprint

Build a four-department agency that turns an operator's recorded lab assessment into a signed-off, evidence-backed report. The operator conducts the assessment and supplies raw command transcripts, request/response captures or configuration exports. The Tester department prepares candidate findings from those files. Automated target probing, scan execution and agent-directed retesting are outside this implementation. A retest means a human can submit additional recorded evidence for the same claim.

**No Proof, No Pass.** A finding becomes eligible for export only when its evidence belongs to the approved engagement, its citations resolve, the Verifier confirms the exact claim, and the operator approves the resulting report version. A hash detects changes since import. It cannot establish that the original recording was authentic. A human sign-off records a decision about the evidence available at that time.

### Build map

| Level | Working increment | Observable outcome |
|---|---|---|
| 0 | Contracts and local inference | A typed response on the RTX 2080 and a recorded memory baseline |
| 1 | Scope approval and evidence intake | One approved capture can be retrieved by ID and cited by line |
| 2 | Tester and Verifier | Supported, contradicted and unresolved claims take distinct paths |
| 3 | Persistent orchestration | A paused review survives restart and stops after two additional evidence rounds |
| 4 | Reporting and evaluation | An approved report can be traced to original evidence and compared with a single-pass baseline |
| 5 | Local dashboard and portfolio release | A reviewer can follow one case from scope approval to export |

**Repository layout at completion.** Paths below are relative to the repository root. The commands in each level are acceptance targets for the components you build. Run them from the root in the project environment. Fixture tests use a scripted model adapter. Live gates explicitly use Ollama.

```text
config/                      scope.toml, runtime.toml
src/agency/                  cli.py, contracts.py, model.py, db.py
  evidence/                  intake.py, store.py, reader.py, mcp_server.py
  roles/                     tester.py, verifier.py, writer.py
  prompts/                   tester.md, observer.md, verifier.md, writer.md
  workflow/                  state.py, graph.py, approvals.py, events.py
  reporting/                 select.py, render.py
  api/                       app.py, routes.py
fixtures/                    demo/, invalid/, evaluation/
tests/                       level0/ through level5/
scripts/                     smoke_model.py, benchmark.py, demo.py
frontend/                    package.json, src/, e2e/
docs/                        architecture.md, evaluation.md, demo.md
var/                         local database, evidence objects, exports and gate results
```

Use one Python package and one application process initially. Keep original evidence, checkpoint databases and real engagement exports in gitignored local storage. Commit only synthetic fixtures, configuration examples and sanitised demo outputs.

## Level 0: Environment & Schemas

### Level Goal & Expectations

Prove that a single local model can return a bounded, validated response on the RTX 2080 with 8 GB VRAM and 16 GB system RAM. Treat JSON validity, evidence validity and model accuracy as separate measurements.

### Step-by-Step Build Instructions

1. Create a Python 3.11 virtual environment with `python -m venv .venv`. Activate it, create a `pyproject.toml` using the `src/` package layout, and install with `python -m pip install -e ".[dev]"`. Declare Pydantic v2, LangGraph, its SQLite checkpoint adapter, Ollama's Python client and pytest. Add MCP at Level 1 and FastAPI/Uvicorn at Level 5. Commit a resolved dependency lock and record the Python and Ollama versions.
2. In the Ollama service environment set `OLLAMA_MAX_LOADED_MODELS=1` and `OLLAMA_NUM_PARALLEL=1`. Restart the service. Install a local Qwen3-4B Q4_K_M model first. Use `ollama list` and `ollama show <installed-tag>` to record its installed ID, tag and quantization. Point every role at that same tag and loopback endpoint. Test Qwen3-8B Q4_K_M in a separate benchmark run. Record CPU offload if observed.
3. In `config/runtime.toml`, start with a 2,048-token context, 384-token generation limit, one in-flight generation, temperature zero and a 120-second timeout. These are initial test settings. Budget prompt, schema, evidence and output together. On overflow, split the evidence into cited spans or pause for narrower input. Keep complete originals on disk.
4. In `contracts.py` derive boundary models from a common strict base. Set limits on text, list lengths and byte counts. Use application-generated identifiers. Store the claim separately from its severity and business impact. Define the following contracts before writing prompts.

| Contract | Required fields and constraints |
|---|---|
| ScopeConfig | engagement ID, approved asset IDs, operator, scope version |
| EvidenceRef | engagement ID, evidence ID, digest, first line, last line, exact quote |
| RawFinding | finding ID, asset ID, one checkable claim, one or more EvidenceRefs |
| ReviewDecision | finding ID, one of the three verifier verdicts, rationale, cited spans, missing fact when requesting more evidence |
| FindingState | finding ID, revision, review status, additional evidence rounds used in 0..2 |
| Approval | engagement ID, purpose, object version or digest, operator, decision, UTC timestamp |
| ReportDraft | engagement ID, confirmed finding IDs and revisions, evidence-backed sections, scope, limitations |

```python
from typing import Literal
from pydantic import BaseModel, ConfigDict, Field, model_validator

class Contract(BaseModel):
    model_config = ConfigDict(strict=True, extra="forbid")

class EvidenceRef(Contract):
    engagement_id: str
    evidence_id: str
    sha256: str = Field(pattern=r"^[0-9a-f]{64}$")
    first_line: int = Field(ge=1)
    last_line: int = Field(ge=1)
    quote: str = Field(min_length=1, max_length=2000)

    @model_validator(mode="after")
    def check_range(self):
        if self.last_line < self.first_line:
            raise ValueError("reversed line range")
        return self

Verdict = Literal["confirmed", "false_positive", "needs_retest"]
```

5. Define one adapter in `model.py` with a live implementation and a scripted implementation. Pass the expected Pydantic JSON schema to Ollama's structured output format, disable streaming for structured replies, and validate the returned content with `model_validate_json`. Reject malformed output before a state update. Allow one format retry, then surface a model error. Format retries consume no evidence rounds.
6. Implement `scripts/smoke_model.py` to run ten short synthetic review cases and write model ID, settings, cold/warm latency, first-pass validity, failures and peak observed GPU/system memory to a JSON result. Sample GPU memory during inference and label the sampling interval. Include idle usage so desktop GPU load is visible.

### Concrete Deliverables

- [ ] `pyproject.toml`, dependency lock, `.gitignore`, `config/runtime.toml`.
- [ ] `src/agency/contracts.py`, `src/agency/model.py`, `scripts/smoke_model.py`.
- [ ] `tests/level0/test_contracts.py`, `tests/level0/test_model_failures.py`, local `var/gates/level0.json`.

### Mandatory Demo & Verification Gate ("Pass Criteria")

Run `python -m pytest -q tests/level0` and `python scripts/smoke_model.py --cases 10 --out var/gates/level0.json`. Inject an extra field, a string line number, a reversed range, malformed JSON twice and a timeout. Each invalid response must leave the accepted-state table unchanged. The live run must complete ten validated cases without OOM under the recorded settings. Record any format retries and CPU offload. If the run fails, reduce context or model size and repeat with the new configuration recorded.

## Level 1: Scope Approval & Evidence Intake

### Level Goal & Expectations

Bind each original capture to one approved engagement and one asset. The evidence service reads bounded local objects. Imported URLs, transcripts and instructions are data. The only inference connection is to the configured local Ollama service.

### Step-by-Step Build Instructions

1. Create `config/scope.toml` with asset ID `juice-local` and a human-readable lab description. Implement a CLI scope approval that writes a versioned approval before opening intake. Check engagement, asset and scope version at import and retrieval. Scope labels record the operator's attribution.
2. Create `fixtures/demo/` with UTF-8, LF-terminated synthetic files. Use these two separate originals throughout the build. Keep the exact bytes stable.

| File | Line 1 | Line 2 |
|---|---|---|
| response.txt | HTTP/1.1 403 Forbidden | X-Request-ID: demo-017 |
| config.txt | application=demo | debug_mode=true |

3. Implement `evidence/intake.py` to accept a selected local file and metadata: asset, capture time, source description, operator and capture type. Set a prototype limit of 1 MiB per text file. Reject oversize files while reading, binary input, symlinks and files outside the designated import folder. Ignore content-supplied filenames when choosing storage paths.
4. Implement `store.py` to hash the original bytes, write a temporary object, atomically publish it, then commit its metadata in SQLite. A unique key on engagement, asset and digest makes an identical re-import idempotent. A failed database commit can leave an unreferenced object. Clean those objects only after checking references. Keep filesystem permissions local to the operator.
5. Implement `reader.py` with one-based inclusive line references over decoded text. Specify that line terminators are excluded from individual lines and multi-line quotes join with LF. Verify engagement, asset, digest, range and quote before returning a citation. Preserve sensitive originals locally and generate separate redacted display copies when needed.
6. Expose `read_excerpt(evidence_id, first_line, last_line)` through a local stdio MCP server. Bind the engagement in the parent process. Run the reader under an OS account or container with read-only access to the evidence directory, no target network access and a bounded response size. The parent handles imports. MCP supplies the interface; OS permissions enforce isolation.

### Concrete Deliverables

- [ ] Scope configuration, approval CLI, SQLite schema and evidence modules shown above.
- [ ] The two demo originals, a second engagement fixture and invalid-import fixtures.
- [ ] `tests/level1/test_intake.py`, `test_citations.py`, `test_reader_boundary.py` under the same test directory.

### Mandatory Demo & Verification Gate ("Pass Criteria")

Run `python -m pytest -q tests/level1`. Import before approval must fail. After approval, import both demo files twice and assert exactly two evidence records. Read line 2 of the configuration and match its exact text. Reject an unknown asset, cross-engagement ID, symlink, oversized input, invalid line range and mismatched hash. Through the actual isolated reader, verify original bytes cannot be changed and an attempted connection to a test listener fails. No rejected import may create a visible evidence record.

## Level 2: Tester & Adversarial Verification

### Level Goal & Expectations

Produce one narrow candidate claim at a time. Give the Verifier an evidence-first view. The same model serves both roles, so agreement alone provides no independent corroboration.

### Step-by-Step Build Instructions

1. Write `prompts/tester.md` and `roles/tester.py`. Supply an approved asset and registered excerpts. Ask for a claim about a recorded observation plus exact citations. A recorded command or request is descriptive metadata. Cite its captured output for claims about results. Reject uncited candidates before verification.
2. Write `prompts/observer.md`. Run a fresh model call with the evidence alone and ask which facts it establishes. Keep the tester's conclusion and conversation history out of this call. Save the observation with the finding revision.
3. In `prompts/verifier.md` and `roles/verifier.py`, compare the exact claim, blind observation and raw spans. Use `confirmed` for direct support, `false_positive` for explicit contradiction and `needs_retest` where a fact is missing. Require a specific missing fact for the third verdict. Persist a short user-visible rationale. Revalidate every returned reference in Python.
4. Seed three claims against Level 1 evidence. Fix the claim wording in these tests so the expected answers are unambiguous.

| ID | Claim | Expected verdict | Decisive evidence or missing fact |
|---|---|---|---|
| C1 | The supplied configuration records debug mode as enabled. | confirmed | config.txt line 2 |
| C2 | The captured response has HTTP status 200. | false_positive | response.txt line 1 explicitly says 403 |
| C3 | Request demo-017 was denied because the user's account was disabled. | needs_retest | The response gives no reason for denial |

5. Add a fixture containing an instruction to mark every case confirmed. Treat it as quoted evidence. Generate finding IDs, allowed tools and routing decisions in application code. Keep the verifier read-only.

### Concrete Deliverables

- [ ] Tester, observer and verifier prompts, two role modules and a seeded claim manifest.
- [ ] `tests/level2/test_verdicts.py`, `tests/level2/test_invalid_citations.py`.
- [ ] `scripts/demo.py` with an initial `verify` subcommand and machine-readable verdict output.

### Mandatory Demo & Verification Gate ("Pass Criteria")

Run `python -m pytest -q tests/level2` and `python scripts/demo.py verify --fixtures fixtures/demo --model live`. Require C1, C2 and C3 to produce the three verdicts above with valid citations. A fabricated quote or a quote from a different asset must fail validation even when the model says confirmed. Record live failures separately from scripted routing tests. Keep this demo set separate from the eventual benchmark.

## Level 3: Persistent Orchestration & Human Gates

### Level Goal & Expectations

Make every transition reproducible from persisted state. A restarted service must resume the correct engagement, preserve approvals and enforce the two-round limit per finding.

### Step-by-Step Build Instructions

1. Implement `workflow/state.py`. Store lightweight identifiers in graph state. Keep evidence bytes in the store and finding revisions in SQLite. Use these fields as the graph boundary.

```python
from typing import Literal, TypedDict

class AgencyState(TypedDict):
    engagement_id: str
    scope_version: int
    stage: Literal["scope", "intake", "review", "await_evidence",
                   "draft", "signoff", "export", "closed"]
    pending_finding_ids: list[str]
    active_finding_id: str | None
    report_digest: str | None
```

2. In `graph.py` build a LangGraph StateGraph with scope approval, intake, candidate preparation, verification, additional-evidence intake and draft preparation nodes. Compile with a persistent SQLite checkpointer. Assign a stable thread ID per engagement. The Lead owns routing and budgets in Python. Implement the following transitions.

| Event | Durable action | Next stage |
|---|---|---|
| Scope approved / declined | Bind approved scope version / close engagement | Intake / closed |
| Confirmed or false positive | Store verdict and exact finding revision | Next queued candidate, then draft |
| Needs retest with rounds remaining | Pause with the missing fact and current revision | Operator adds evidence or closes as inconclusive |
| Additional evidence accepted | Atomically increment rounds and create a revision | Verify that revision |
| Needs retest after two additional rounds | Store terminal inconclusive status | Next candidate |
| Report approved / declined | Bind signature to draft digest / retain edit reason | Export / draft revision |

3. Use LangGraph interrupts for human decisions. Resume with the same thread ID. An interrupted node starts from its beginning on resume, so put durable mutations after the interrupt or in an idempotent downstream node. Persist request IDs and expected revisions. Reject stale submissions. Duplicate submissions return the original outcome.
4. In `db.py` use one SQLite transaction for the domain transition, retry counter, event and request ID. Protect updates with an expected revision. A checkpoint and domain transaction can commit separately. On replay, look up the durable transition key before applying it again. Never derive retry counts from chat messages.
5. In `events.py` emit monotonically ordered per-engagement events. Include timestamp, actor, finding revision, decision and evidence IDs. Keep model formatting failures distinct from evidence rounds. Stub the draft boundary until Level 4.

### Concrete Deliverables

- [ ] StateGraph, persistent checkpointer, approval handlers and event repository.
- [ ] `tests/level3/test_resume.py`, `test_retry_budget.py`, `test_idempotency.py`.
- [ ] CLI commands to inspect a paused run, add an evidence revision and resume it.

### Mandatory Demo & Verification Gate ("Pass Criteria")

Run `python -m pytest -q tests/level3`. Start C3 at zero rounds, pause, stop the process and resume with the original thread ID. Supply two distinct synthetic records for demo-017 that still omit the denial reason. The finding must become inconclusive after three verifier decisions in total. Submit the second event twice and assert the counter remains two. Start two simultaneous submissions at the same revision and assert only one transition commits. Confirm a different finding still has its own full budget.

## Level 4: Reporting, Sign-Off & Evaluation

### Level Goal & Expectations

Publish only claims supported at their approved revisions. Measure whether adding verification changes the quality of accepted findings and whether that improvement justifies its latency on the target hardware.

### Step-by-Step Build Instructions

1. In `reporting/select.py`, select confirmed finding revisions for one engagement. Recheck every evidence reference before creating the writer input. Give the writer a list of allowed finding IDs and facts. Keep false positives, workflow errors and inconclusive cases in the operator audit view.
2. In `prompts/writer.md` and `roles/writer.py`, request a structured executive summary, technical finding, evidence citation, limits of interpretation and suggested remediation for each accepted claim. Use `impact_unknown` where business context is absent. Provide the human reviewer with a side-by-side view of original claims and proposed report text. A valid citation can still accompany an overstated conclusion.
3. Render Markdown deterministically in `reporting/render.py`. Enforce one technical entry per selected ID and require citations in executive assertions as well. Produce a valid zero-findings report when the set is empty. Describe the evidence reviewed and its limits. Hash the final rendered bytes. An edit or new evidence revision invalidates sign-off. Export exactly the approved bytes via atomic rename.
4. In `fixtures/evaluation/`, prepare a labelled held-out set. A practical pilot is 30 cases, ten per evidence class. Keep related captures and variants in the same split. Have a human record the expected decision, acceptable spans and justification before the live run. Exclude labels and expected answers from prompts.
5. Implement `scripts/benchmark.py`. Give both modes identical claims and evidence, model digest, generation settings and output schema. Baseline uses one decision call. The agency uses the separate observation and verification passes. Freeze the evidence for this comparison. Measure the cost of human evidence requests in a separate workflow demo. Record ten warm-run latency samples or more per configuration and label cold-start latency separately.
6. Write `docs/evaluation.md`. Report a three-class confusion matrix and raw counts. For accepted findings, precision is correctly confirmed divided by all confirmed. Recall is correctly confirmed divided by all labelled supported. Report zero denominators as unavailable. Count schema failures, unresolved cases, peak VRAM/RAM and median end-to-end latency. Compare 4B and 8B only after measuring each on the same hardware. State the sample size and any offload or memory failure.

### Concrete Deliverables

- [ ] Writer, report selector, renderer, report digest and approval/export service.
- [ ] Held-out cases, private-to-evaluator answer key, benchmark script and evaluation table.
- [ ] `tests/level4/test_report_gate.py`, `test_benchmark.py` and synthetic example report.

### Mandatory Demo & Verification Gate ("Pass Criteria")

Run `python -m pytest -q tests/level4` and `python scripts/benchmark.py --cases fixtures/evaluation --modes single,verified --out var/gates/level4.json`. Confirm that the demo report includes C1 only, cites its original line and makes no unsupported claim of user impact. Export before sign-off must fail. Change one byte of an approved draft and verify the old approval cannot export it. Both modes must account for all 30 cases, including model failures. Passing this gate requires an honest comparison even if the verifier scores worse.

## Level 5: Full-Stack Dashboard & Portfolio Polish

### Level Goal & Expectations

Let an operator inspect the original evidence, understand a decision and approve the exact report being exported. The backend remains authoritative for scope, retries, transitions and exports.

### Step-by-Step Build Instructions

1. Add FastAPI in `api/app.py` and `api/routes.py`. Bind the local service to loopback. Use a local operator session for mutations, allow only the frontend origin, validate request schemas and enforce engagement ownership at every read and write. Return a conflict for stale revisions.

| Endpoint | Contract |
|---|---|
| POST /engagements | Create a draft scope and return the run ID |
| POST /engagements/{id}/decisions | Submit a scope or report decision with expected revision and request ID |
| POST /engagements/{id}/evidence | Import a bounded file for an approved asset, optionally linked to a pending evidence request |
| GET /engagements/{id}/findings/{fid} | Return claim, rationale, evidence references and revision history |
| GET /engagements/{id}/evidence/{eid} | Return authorised, bounded display excerpts |
| GET /engagements/{id}/events | Stream persisted events using server-sent events and support reconnection by event ID |
| GET /engagements/{id}/report | Return the current draft and digest |
| POST /engagements/{id}/export | Return the exact approved report or a refusal with the reason |

2. Scaffold React/TypeScript in `frontend/`. Implement an engagement view with scope and approval state, a findings queue with accessible text verdicts, a split claim/evidence review panel, and a report preview. Make line citations open the cited range. Render evidence as plain text. Indicate a paused decision, a stale revision, and an unavailable model explicitly.
3. Reuse the CLI's application services. For live updates, fetch a snapshot with its last event ID, then subscribe from that ID. On reconnect, deduplicate by ID. Refresh stale state before allowing a second decision. Keep full evidence out of streamed event payloads.
4. Implement `scripts/demo.py dashboard --seed fixtures/demo` to start a fresh synthetic engagement in a separate demo data directory. Include an explicitly labelled scripted mode for repeatable UI tests and a live option for the recorded walkthrough.
5. Add `docs/architecture.md` with trust boundaries and storage ownership, `docs/demo.md` with setup and reset steps, and a README linking to the demo, sample report and benchmark. Record a short walkthrough showing a rejected claim and an unresolved one alongside the confirmed case. Label measured results with model ID and hardware.

### Concrete Deliverables

- [ ] FastAPI app, API tests, typed frontend client, review pages and event stream.
- [ ] `frontend/e2e/review.spec.ts`, package lock, `tests/level5/test_api.py`.
- [ ] Synthetic walkthrough, architecture diagram, sample report, benchmark results and reproducible README.

### Mandatory Demo & Verification Gate ("Pass Criteria")

Run `python -m pytest -q`, then `npm ci`, `npm run build` and `npm run test:e2e` in `frontend/`. Configure the browser tests to launch the local API and seed a fresh demo directory. Approve scope, import the two captures, inspect C2's rejection, close C3 as inconclusive and follow C1's citation. Refresh during a pause, reconnect the event stream and submit a duplicate approval. Expect one durable decision, no duplicate timeline rows and an export matching the displayed digest. Request an evidence ID from a second engagement and verify access is denied. Finish with one live Ollama walkthrough on the RTX 2080 and record the actual outcome.

### Engineering references

- [LangGraph interrupts and resuming a persisted thread](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [Ollama structured outputs and schema validation](https://docs.ollama.com/capabilities/structured-outputs)
- [Ollama environment configuration, memory and concurrency](https://docs.ollama.com/faq)
- [Pydantic v2 strict validation](https://docs.pydantic.dev/latest/concepts/strict_mode/)
