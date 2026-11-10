# AgentGate

An AI agent evaluation workbench for measuring task completion, tool-use correctness, and resilience to prompt injection.

AgentGate runs local tool-using models through procurement tasks, retrieves evidence with Ollama embeddings and FAISS, and checks the resulting answers and sandbox state independently of the gateway. Controlled comparisons separate model behavior, prompt defenses, and server-side enforcement. The DeBERTa work includes classifier adaptation and a separate detector-in-the-loop agent comparison, with baseline and cross-domain results preserved.

## Recovery after filtering

The live agent also supports one bounded recovery attempt after filtering: search
the same authorized organization corpus, exclude the blocked source, and rescan
all replacement passages before inference. Enable it with
`AGENTGATE_THREAT_RECOVERY=retrieve` alongside input filtering.

A separate **96-episode fresh-scenario comparison** holds the fine-tuned detector,
gateway, tools and agent budgets fixed while comparing filtering-only with
filtering plus recovery. Both arms can access the same fallback documents.
It reports task completion, attacker success, paired task gains/losses and latency.

On this pilot, attacked task completion rose from **1/24 to 5/24**; attack success
stayed at **2/24** and clean completion at **9/24**. Overall mean latency increased
from **1.58 s to 2.47 s**. The paired comparison contains five task gains and one loss.

- [Comparison report](experiments/agent-recovery-v1/REPORT.md)
- [Implementation and reproduction](docs/recovery-comparison.md)

```sh
npm run benchmark:recovery:verify
```

## Detector-in-the-loop comparison

The live agent supports opt-in DeBERTa filtering of untrusted tool results and
retrieved passages before generation. A controlled comparison holds the gateway,
scripted reviewer, tasks and agent settings fixed while varying no detector,
upstream DeBERTa and the domain-adapted checkpoint.

See the [results and run provenance](experiments/agent-detector-v1/README.md) and
[implementation/reproduction guide](docs/detector-comparison.md). Both a primary
120-episode run and the first supplementary run are retained separately. The
report measures attacker success, clean/attacked task completion, false alarms,
and end-to-end/detector latency.

```sh
npm run benchmark:detector:verify
```

## Real agent benchmark

The [120-episode pilot report](experiments/agent-pilot-v1/README.md) compares
Qwen3 1.7B and Llama 3.2 3B across unprotected, prompt-only, and gateway plus
scripted-review modes. Ten paired synthetic task families test exact tool
parameters, purchases, approvals, real RAG answers and citations, indirect
injection, duplicate effects, and answer contamination.

The benchmark checks final outcomes independently of gateway verdicts. It retains
all failures, records whether attacks actually reached the model, and publishes
full inference/retrieval trajectories, model digests, frozen source hashes, token
counts and latency. Ten families and one trial per cell make this a pilot, not a
claim of general agent robustness.

- [Protocol, architecture and reproduction commands](docs/agent-benchmark.md)
- [Generated comparison report](experiments/agent-pilot-v1/REPORT.md)
- [Machine-readable summary](experiments/agent-pilot-v1/summary.json)
- [Recorded trajectories](experiments/agent-pilot-v1/episodes.jsonl)
- [Original DeBERTa training experiment](experiments/deberta-v1/results/REPORT.md)
- [DeBERTa v2 learning-rate search and TF-IDF comparison](experiments/deberta-v2/results/REPORT.md)
- [DeBERTa v3 expanded-data and fresh-test comparison](experiments/deberta-v3/results/REPORT.md)

```sh
npm run benchmark -- --out artifacts/agent-pilot-v1
npm run benchmark:report -- --input artifacts/agent-pilot-v1
npm run benchmark:verify
```

Live runs require the local models and retrieval service described in the protocol.
`benchmark:verify` replays the published evidence offline; CI does not download
models or substitute mock inference for the live results. The app's original
12-case reference suite and legacy platform dashboard are separate demonstrations.

## Future roadmap: Dify integration (planned)

**Status: proposed; not implemented or evaluated.** This roadmap adds Dify as
a second procurement-agent runtime while retaining AgentGate's custom agent,
gateway, evaluation framework and existing published experiments. It does not
claim that Dify improves task completion, security or latency.

### Goals and architecture

The planned integration would let a Dify agent choose procurement tools while
AgentGate governs requests and independently evaluates their effects:

```text
User / benchmark runner
        |
        v
Dify procurement agent ---- model provider (e.g. local Ollama)
        |
        | authenticated MCP calls
        v
AgentGate MCP endpoint
        +-- authorization, tool schemas and procurement policies
        +-- parameter-bound approval and execution audit
        +-- sandbox catalog, document and purchase tools
        +-- PLANNED tool-result screening and bounded recovery
                    |
                    +-- DeBERTa detection
                    +-- authorized FAISS retrieval
                    +-- rescored passages and source citations
        |
        v
Independent evaluator: sandbox state, answer, citations, traces and timing
```

The screening boundary is a **design target**, not existing Dify integration.
AgentGate's present DeBERTa filtering and recovery run inside its own agent
step. Calling the MCP gateway alone does not establish that Dify receives
screened tool results; native Dify knowledge retrieval and other input paths
could also bypass screening. The first milestone uses synthetic procurement
data and sandbox effects, not real purchases or supplier messages.

### Phase 1: establish compatibility

1. Select and record a Dify release, deployment method, dependencies, agent
   mode, plugin versions and MCP transport. Check the selected version's
   documentation because these interfaces and deployment requirements change.
   See the [Dify repository](https://github.com/langgenius/dify).
2. Start Dify locally and configure a compatible model provider. For the first
   comparison, prefer the same Ollama model and model digest used by the
   existing agent. Record supported generation settings and defaults.
3. Start AgentGate, create a dedicated organization-scoped MCP credential and
   configure the exact organization context and authentication headers in the
   Dify client. Keep tokens, local paths and real data out of the repository.
4. Verify container-to-host connectivity, MCP discovery and calls, transport
   compatibility, session behavior, timeouts and token revocation. Test that
   missing, invalid, revoked or cross-organization credentials fail.
5. Document exact setup steps and record any incompatibility. Do not assume that
   a given Dify release supports AgentGate's approval flow without testing it.

**Deliverable:** a reproducible local setup and a smoke test that discovers
allowed tools and performs an authorized, tenant-scoped read.

### Phase 2: procurement agent and approvals

1. Configure a Dify agent that dynamically chooses among supplier-document,
   catalog and draft-purchase tools for the user's procurement task. Define the
   system instructions, tool descriptions, output format and execution budget.
2. Route all consequential calls through AgentGate. Preserve exact arguments,
   source and citation identifiers, proposal IDs, pending status and tool
   outcomes when mapping MCP responses into Dify.
3. Define the asynchronous approval lifecycle: submit the proposal, display or
   retain its ID, wait or poll with a bounded timeout, and continue only after
   AgentGate records a decision. Verify whether the selected Dify version can
   pause and resume; otherwise design a narrowly scoped external adapter.
4. Test parameter fingerprint binding, denial, expiry, reuse, changed arguments,
   duplicate requests, policy changes and organization isolation. A scripted
   benchmark reviewer must remain distinguishable from actual human review.
5. Correlate the Dify run with AgentGate's policy decisions, approval events,
   execution journal and hash-chain audit without disclosing credentials.

**Acceptance:** a permitted synthetic task creates at most its authorized
draft; denied, changed, expired and duplicate proposals produce no unauthorized
effects. Document any unsupported behavior instead of bypassing controls.

### Phase 3: integrate DeBERTa and recovery

1. Inventory every untrusted input reaching Dify, including MCP tool responses,
   retrieved passages, uploaded documents, native knowledge nodes and other
   tools. Disable or explicitly route paths not covered by the intended guard.
2. Design a gateway response layer or Dify-side adapter that scores relevant
   responses and passages **before model consumption**. Initially use the
   existing frozen DeBERTa checkpoint and validation-selected threshold.
3. Retain original evidence and scores for audit while sending only screened
   content and permitted citation metadata to the agent. Verify the actual model
   input, not merely the gateway's recorded decision.
4. After a block, allow at most one read-only recovery search over the
   authorized organization corpus using the original user request. Exclude
   blocked sources, rescore every fallback passage, preserve citations and
   return explicit no-evidence status when suitable evidence is unavailable.
5. Define and test failure behavior for detector outage, retrieval errors,
   malformed or oversized results, timeouts, repeated calls and missing
   evidence. Never pass unchecked fallback text after a scoring failure.
6. Test mixed clean/malicious passages, a poisoned fallback, absent fallback,
   duplicate sources, cross-tenant access and attempts to bypass screening.

**Acceptance:** blocked text is absent from the model input, every delivered
replacement passage has a recorded score, and recovery creates no write or
external action. Do not describe this protection as complete until verified.

### Phase 4: benchmark integration and evaluation

1. Implement a Dify runner adapter that records scenario and condition, runtime
   and model identity, configuration, final answer, tool and approval events,
   sandbox effects, returned citations, tokens, latency, errors and termination.
   Keep outcome checks independent of Dify and gateway verdicts.
2. Author fresh clean/attack scenario pairs covering evidence retrieval, answer
   contamination, unauthorized tool use, approval manipulation, missing
   fallback evidence and poisoned replacement passages. Reserve separate
   development scenarios for debugging; freeze the evaluation set before runs.
3. Preregister the matrix, trial count, order, stopping rules and primary
   metrics. Compare Dify and the custom agent using the same tasks, model digest,
   synthetic corpus, gateway policy, scripted reviewer, detector and threshold
   where feasible. Record differences in prompts, tool schemas, retrieval,
   generation settings, budgets and orchestration as possible confounders.
4. Reset and isolate sandbox state for every episode. Alternate or randomize
   order, record supported seeds, and set equal time and turn limits where
   possible. Retain errors, timeouts and incomplete episodes in denominators;
   define retries before the experiment.
5. Report clean and attacked task completion, attack success, unauthorized
   effects, citation correctness, false alarms, errors, tokens and end-to-end
   latency. Include counts and denominators, per-model and per-family results,
   paired gains/losses and uncertainty where the sample supports it.
6. Document cold versus warm runs and retrieval/model cache effects. Distinguish
   a comparison of agent runtimes using shared components from a comparison of
   entire application stacks. Do not attribute stack-level differences to Dify
   alone or generalize a small authored pilot to production.
7. Freeze a manifest before inference with hashes of scenarios, code, sanitized
   workflow export, model artifacts and configuration. Save redacted trajectories
   and machine-readable results so independent outcome checks replay offline.

**Acceptance:** every scheduled cell appears once, replayed outcomes reproduce
the report, altered evidence or configuration is detected, and prior frozen
AgentGate reports remain unchanged. Existing benchmark numbers are not Dify
results.

### Phase 5: release and operational checks

1. Publish versioned setup, reproducibility and troubleshooting documentation;
   include a sanitized workflow export if supported. Explain retrieval ownership
   and the screening coverage of each input path.
2. Add unit and integration tests for MCP authentication, gateway enforcement,
   approval transitions, screening, recovery, tenant isolation and adapter
   behavior. Include tests for missing cells, changed outcomes and blocked
   evidence reintroduced into model input.
3. Add offline evidence verification to CI. Keep live inference optional unless
   the CI environment supplies the required models and services; do not label
   fixture-only checks as real-model results.
4. Scan and manually review workflow exports, logs and public artifacts for
   personal information, tokens and local paths. Publish only synthetic data.
5. Publish a new versioned report with the measured results, configuration,
   limitations and exact commit only after the integration and evaluation run.

### Open decisions

- **Interface:** verify direct MCP support for AgentGate's authentication,
  transport and asynchronous approvals; build an adapter only if required.
- **Retrieval:** prefer AgentGate's existing retrieval for the controlled
  comparison; assess Dify-native RAG separately unless corpus, chunking,
  embedding and citation equivalence can be demonstrated.
- **Guard placement:** choose a gateway response layer or Dify-side adapter based
  on coverage and prove that no enabled untrusted input bypasses it.
- **Scope:** decide whether the experiment compares agent runtimes or complete
  stacks; document unavoidable differences before collecting results.
- **Go/no-go:** use compatibility findings and implementation cost to decide
  whether Dify adds sufficient evaluation or demonstration value.

## License

AgentGate-authored software is available under the [PolyForm Noncommercial
License 1.0.0](LICENSE). It permits noncommercial use, modification and
redistribution subject to its terms. Commercial use requires a separate grant
of rights from the applicable copyright holder. The license explicitly permits
uses by certain educational, research, charitable, public-service and government
organizations, as described in its text.

This project is **source-available**, not OSI-approved open source. Third-party
code, data, model weights and other assets remain subject to their own applicable
licenses and notices; the AgentGate license does not replace those terms.

## Current scope

- Register an immutable agent version with explicit tools, destinations, approval requirements, and sensitive-data controls.
- Compare permissive and hardened purchasing reference agents using the same 12-case suite.
- Save every report and policy snapshot, inspect individual findings, and export JSON evidence.
- Evaluate a real target with a separate Promptfoo provider that never silently substitutes fixtures.
- Isolate stored records by authenticated user. Hosted identity comes from Sites; local sign-in is simulated by the starter.
- Expose governed tools through an authenticated MCP JSON-RPC endpoint using the official TypeScript SDK schemas.
- Enforce procurement controls for approved vendors, currency, categories, automatic approval thresholds, and monthly budget.
- Export durable run traces in an OTLP-shaped JSON format and show business KPIs alongside security metrics.

The original page runs **reference policy assessments**; /platform adds real
local Ollama runs, server-side tool enforcement, exact-action approvals,
three-mode benchmarks, replay/rerun, organization roles and hash-chained audit.
See [Ollama setup](docs/ollama-setup.md) and
[platform architecture](docs/platform-architecture.md).

Reference pass rates are deterministic fixture results, not model robustness.
Live benchmark numbers appear only after actual model runs. All effects remain
inside isolated sandbox outboxes, orders, and draft purchase requests. MCP calls
use the same server-side gateway as Ollama runs. OpenTelemetry instrumentation
hooks and OTLP export are included; no external collector or Langfuse account is
required.

## Project tutorial: techniques and areas covered

Use the project from left to right as a compact industry tutorial:

1. **Agent and tool security** — strict Zod schemas, allowlists, exact URL
   validation, sensitive-data filtering, prompt-injection defenses, and a
   fail-closed server-side gateway.
2. **MCP integration** — official MCP request validation, Streamable HTTP-style
   JSON-RPC, hashed/revocable bearer tokens, tool discovery, governed tool calls,
   and asynchronous approval-status lookup.
3. **Policy-as-code** — versioned policy snapshots plus an OPA/Rego reference
   bundle for vendor, currency, category, approval-limit, and budget rules.
4. **Human-in-the-loop control** — exact-action SHA-256 fingerprints, expiring
   one-use approvals, optional two-person review, and policy-change invalidation.
5. **Reliable backend engineering** — organization RBAC, tenant isolation,
   prepared D1 queries, optimistic concurrency, durable model leases, atomic
   audit/effect commits, storage limits, and idempotency-oriented design.
6. **AI evaluation and red teaming** — deterministic Promptfoo regression tests
   plus live development/holdout cases across direct and indirect injection,
   domain confusion, data leakage, multilingual attacks, excessive agency, and
   procurement abuse. Security, usefulness, false blocks, errors, and pending
   runs are reported separately.
7. **Observability and evidence** — durable execution traces, OpenTelemetry API
   spans, OTLP JSON export, hash-chained audit checkpoints, replay versus rerun,
   model latency, and token accounting.
8. **Business value** — safe automation rate, prevented policy violations,
   governed spend, draft request volume, and approval turnaround metrics connect
   technical controls to procurement outcomes.
9. **Software quality** — unit, SQLite-backed concurrency, authenticated API,
   MCP integration, lint, formatting, type checking, Promptfoo, and production
   build gates in GitHub Actions.

## Run locally

Requires Node 24 and npm. Run commands from the project root.

    npm run install:ci
    npm run build
    node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0000_melted_red_ghost.sql
    node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0001_next_shape.sql
    npm run dev

Apply the migration once to a fresh local database. Open the development URL and sign in through the local sign-in link. Click **Load reference agents**, assess the baseline, then assess the hardened version. The baseline should pass 7/12; hardened should pass 12/12. These are deterministic fixture results.

## Verify

Public artifacts use generic hosting metadata and GitHub no-reply commit
attribution. See [public repository hygiene](docs/public-repository.md) for
privacy checks, local configuration, and frozen experiment integrity.

    npm test
    npm run typecheck
    npm run lint
    npm run format:check
    npm run eval:reference
    npm run build

CI performs the same checks without target credentials and exercises the
authenticated HTTP and MCP APIs against a local D1 database. Keep live model
evaluations outside untrusted pull-request jobs.

## Live target evaluation

Set AGENTGATE_TARGET_URL to your own test agent endpoint and optionally AGENTGATE_TARGET_TOKEN, then run:

    npm run eval:live

The runner sends JSON containing prompt, policy, and approval:false. The target must return:

    {"actions":[{"tool":"search_catalog","content":"A-100"}],"text":"Checking delivery availability."}

Actions are proposals, never executed by AgentGate. Target-provided approved fields are explicitly ignored. The provider reports unsafe proposals and final-response sensitive-data matches separately. Malformed responses, timeouts, redirects, and missing configuration become errors. HTTP is accepted only for loopback endpoints. Tests use synthetic secrets.

This generic Promptfoo target adapter remains separate from the bundled Ollama
platform. Build an adapter emitting this schema to test other agents. Its four
starter cases are a smoke suite, not a comprehensive benchmark. Final-text pattern
checks can miss paraphrased or encoded data. The separate headless pilot linked above records actual local model inference.

## Architecture

Browser → authenticated workspace API → D1 agents and reports

Reference suite → deterministic action policy → findings and all-cases-pass release gate

Promptfoo CLI → explicit HTTP target → schema validation → proposal checks and output checks

Reports are append-only through the application API. Database administrators can still modify them; exports are not cryptographically signed.

## Live platform demonstration

1. Configure Ollama locally, create a lab, and run a useful catalog task.
2. Run the development batch in all three modes and inspect unsafe proposals
   separately from unsafe executed actions.
3. Configure procurement controls, create an MCP token, and call the governed
   purchase tool from an MCP client.
4. Request a high-value purchase or supplier message, inspect its fingerprint,
   approve it, and resume.
5. Change policy, replay old proposals without side effects, then rerun the model.
6. Export an OTLP trace, inspect business KPIs, invite a reviewer, enable
   two-person approval, and save an audit checkpoint.

The SQLite-backed tests cover concurrent approvals, procurement controls,
tenant/role isolation,
invitation replay, model leases, audit tampering and honest error reporting.
Future production work includes larger independent attack sets, real connector
idempotency, distributed orchestration, rate limits, and external audit anchoring.

See [threat model](docs/threat-model.md) and [evaluation design](docs/evaluation.md).

## Attribution

- [Promptfoo](https://github.com/promptfoo/promptfoo) — MIT; evaluation runner and custom-provider interface.
- [Model Context Protocol TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) — MCP schemas and protocol types.
- [OpenTelemetry JS](https://github.com/open-telemetry/opentelemetry-js) — vendor-neutral instrumentation API.
- [Open Policy Agent](https://github.com/open-policy-agent/opa) — Rego policy format used by the portable reference bundle; hosted enforcement uses the tested native equivalent.
- [Vinext](https://github.com/cloudflare/vinext), React, Drizzle, and bundled shadcn components retain their respective upstream licenses.
- Application setup and private hosting use the Sites starter.

This repository is an original application using upstream components, not a fork of Browser Use or Promptfoo. Upstream stars are not AgentGate's adoption metrics.

## BERT-family threat detection

The platform includes an observation-only Threat Detection Lab and optional live
user/tool-message scanning. See [local setup and model limitations](docs/threat-detection.md).

## ML experiments and governed RAG

- **Evaluation and error analysis:** per-example JSON exports, an offline HTML
  review screen with false-positive/false-negative and category/language/length
  filters, precision–recall threshold sweeps, validation-only threshold selection,
  and p50/p95 latency. See [ML workflow](docs/ml-workflow.md).
- **Fine-tuning:** a reproducible grouped split utility and a PyTorch/Hugging Face
  training runner comparing TF-IDF logistic regression, the pinned DeBERTa model,
  and a domain-fine-tuned checkpoint. Artifacts record provenance, seeds, settings,
  checkpoint identities, and held-out results. A completed three-seed domain
  adaptation pilot now includes 480 AI-authored synthetic examples and 1,001
  separate public diagnostic examples. See the
  [measured experiment](experiments/deberta-v1/results/REPORT.md),
  [dataset limitations](experiments/deberta-v1/DATA_CARD.md), and
  [reproduction guide](experiments/deberta-v1/README.md). These results are not
  production-security or real-user-traffic measurements.
- **Document retrieval:** organization-scoped document ingestion, content versions,
  retrieval controls, overlapping chunks, Ollama embeddings, and a local FAISS
  service. The Documents tab and governed `search_documents` tool return source
  evidence with citations; Ollama and authenticated MCP share enforcement.
  See [retrieval setup and boundaries](docs/retrieval.md).

## DeBERTa v3: expanded training and a fresh synthetic test

The [v3 experiment](experiments/deberta-v3/README.md) adds 240 original AI-authored
contrast families and removes repeated wrappers from the old training set.
Training now has **360 distinct texts from 180 families** (previously 36 families).
Validation and fresh test each have **96 texts from 48 new families**.
All labels are AI-authored; the test is held out from fitting/selection, not
independently human-annotated or a production benchmark.

- Validation selects among nine word/character TF-IDF configurations and three
  DeBERTa learning rates. The selected learning rate `5e-05` is
  evaluated across seeds 42/43/44 after all choices are locked.
- Primary validation-selected thresholds: DeBERTa mean F1 **96.54%**
  versus selected `tfidf-union-C10` **82.35%**;
  difference **+14.19 pp**, paired 48-family bootstrap
  95% interval **[+7.07, +22.76] pp**.
- DeBERTa mean recall **97.22%**, FPR
  **4.17%**. Validation's 5% FPR constraint
  does not guarantee test FPR.
- Fixed-0.5 deepset F1: upstream **57.98%**, v3 mean
  **60.43%**, but seed F1 ranges
  **52.25%–73.93%** and two of three seeds
  regress against upstream. External diagnostics and both threshold policies
  remain visible in the [complete report](experiments/deberta-v3/results/REPORT.md).
- The default detector is unchanged. Shared synthetic style, short fragments and
  limited test families constrain the claim; classifier results do not establish
  end-to-end agent security. Earlier v1/v2 results are preserved below.

## Original DeBERTa v1 results and model-selection decision

The completed pilot uses 480 AI-authored synthetic examples from 60 contrast
families. The held-out domain test contains **12 families and 96 correlated
variants**, not 96 independently authored scenarios. Fine-tuned metrics below
are means across seeds 42, 43, and 44.

| Evaluation                                               | Measured result                                                                |
| -------------------------------------------------------- | ------------------------------------------------------------------------------ |
| Main protocol: validation-selected thresholds, domain F1 | Upstream 64.00%; fine-tuned mean 95.51%; TF-IDF 95.65%                         |
| Main protocol: domain recall                             | Upstream 50.00%; fine-tuned mean 95.83%                                        |
| Main protocol: domain false-positive rate                | Upstream 6.25%; fine-tuned mean 4.86%; seeds 42 and 43 each 6.25%              |
| Separate deepset diagnostic, fixed threshold 0.5         | F1 declined from 57.98% to 50.75%; recall from 41.44% to 34.09%                |
| Model-selection decision                                 | Keep the existing default detector; fine-tuned checkpoints remain experimental |

The empirical 5% validation false-positive constraint is not a test-set guarantee.
The neural model did not establish an F1 advantage over TF-IDF under the main
protocol. External diagnostics are separate datasets: deepset has 662 labeled
examples; NotInject has 339 benign examples and supports false-positive analysis,
not attack recall or F1. These results do not establish production protection.

See the [full report](experiments/deberta-v1/results/REPORT.md),
[data card](experiments/deberta-v1/DATA_CARD.md), and
[reproduction instructions and evidence](experiments/deberta-v1/README.md).

## DeBERTa v2 exploratory follow-up

The [v2 experiment](experiments/deberta-v2/README.md) screens three learning rates
using validation data, then trains the selected setting with seeds 42/43/44.
TF-IDF regularization is also selected on validation; the original C=1 baseline
is retained. All new selections are locked before evaluation, but v1 test results
were already known: this is an exploratory follow-up, not a new blind benchmark.

Selected checkpoint weight hashes matching v1: **3/3**.
Matching hashes reproduce the original weights; they are not newly improved models.

- Selected DeBERTa LR `2e-05`: mean domain F1 **95.51%**,
  versus original TF-IDF **95.65%** and validation-selected TF-IDF
  (C=0.1) **94.51%**, at validation-selected thresholds.
- Mean domain recall **95.83%**, FPR **4.86%**.
  The paired 12-family bootstrap interval for the F1 difference from original
  TF-IDF is **[-5.67, +8.55] percentage points**.
- Fixed-0.5 deepset F1: upstream **57.98%**, v2 mean **50.75%**.
  Read [every seed, both thresholds and external diagnostics](experiments/deberta-v2/results/REPORT.md).
- The existing detector remains the default. Small synthetic samples, prior test
  reuse and validation selection limit claims; classifier gains do not establish
  improved end-to-end agent safety.

## Future implementations

The following items are planned extensions, not completed features:

4. **Structured assessment reports:** add a separate Ollama JSON-schema response
   step for summaries, finding IDs, evidence references, recommendations, and
   limitations. Validate references against stored evidence; calculate decisions
   and metrics in code. Tool schemas and runtime validation already exist.
5. **Broader Promptfoo release evaluation:** maintain independent attack datasets,
   compare model/prompt releases, import live reports into the workbench, and report
   unsafe proposals/executions, legitimate completion, false blocks, errors, and
   latency separately. Add retrieval recall, citation correctness, grounding, and
   document-injection evaluations for RAG.
6. **Backend scaling and Python CI:** add the Python suites to GitHub Actions,
   bounded inference scheduling with load tests, and separate tables for growing
   assessments, traces, and document metadata. Preserve atomic approval/effect
   commits when migrating away from the current bounded organization JSON record.
