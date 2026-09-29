# AI-Native Kubernetes Platform — Secure Build Plan

Sep 28, 2026 · @Sudhanshu Singh

## Summary and constraints

Build a **secure Kubernetes tool layer** (RBAC-aware reads, redaction, deterministic triage, change timeline) and expose it through a CLI, an MCP server and a lean terminal UI, for teams that today have only raw `kubectl`. Nothing is built until a short validation phase confirms the need, a sponsor, and the data-handling rulings.

**What changed after the staff review**

- Security is split into **platform-enforced controls** (RBAC, admission policy, cloud IAM, API audit logs) and **tool guardrails** (UX safety that a user could bypass with `kubectl`). Only the first counts as enforcement.
- New Phase 0: validate the problem, get a sponsor, file the K9s intake request, and get data-classification rulings before writing code.
- Desktop UI cut. Surfaces are CLI, MCP and a minimal TUI; the tool layer is the product.
- Prod writes go through the firm's existing prod access process, never a parallel approval system.
- AI is measured on replayed real incidents and on its value over rules alone, not on synthetic scenarios.
- Added ownership, support, distribution (jump hosts, VDI) and API-server load sections.

**Goals**

- Faster day-to-day cluster work than raw `kubectl` for namespace-scoped users.
- Faster root-cause analysis through triage rules, a change timeline, and an AI layer that shows its evidence.
- Works with any conformant cluster: EKS, AKS, GKE, OpenShift, Rancher/RKE2, on-prem.
- Passes an enterprise security review and has an owning team before any write feature ships.

**Non-goals**

- Replacing server-side security controls. The tool never claims to enforce what only the platform can enforce.
- A desktop UI (revisit only if users ask after the TUI is in use).
- Its own prod-approval or two-person-approval system.
- Replacing CI/CD, GitOps or the observability stack.
- Autonomous remediation.

**Constraints in your environment**

- No internet downloads: every dependency must come through the internal mirror.
- LLM use needs an approved endpoint and a data-classification ruling for cluster logs and manifests, which may contain customer data.
- Copilot's MCP policy must be enabled by its admins for the MCP route.
- Many private clusters are reachable only from jump hosts or VDI.
- Code written here belongs to the firm; open-sourcing it later needs the open-source office's approval, decided up front.

## Security model and threat model

The user holds the same credentials in `kubectl`, so anything the tool does client-side is a guardrail, not enforcement. Real enforcement lives in the platform; the tool's job is to never exceed it, never leak data, and make mistakes hard.

**Enforcement boundary**

| Control | Enforced by | Bypassable by the user? |
| --- | --- | --- |
| Who can read or change what | Kubernetes RBAC, cloud IAM (EKS access entries, Azure RBAC, GKE IAM) | No |
| What changes are allowed (no privileged pods, no `latest` tags, protected namespaces) | Admission policy: ValidatingAdmissionPolicy, Kyverno or Gatekeeper | No |
| Prod write access and approvals | The firm's existing just-in-time / privileged access process | No |
| Authoritative audit record | Kubernetes API audit logs + cloud audit (CloudTrail, Azure Activity Log, GCP Audit Logs) | No |
| Where cluster data may go (LLM, Copilot) | Data-classification ruling + network and tenant controls on the model endpoint | No |
| Prod read-only mode, blocked verbs, confirm prompts | Tool guardrail | Yes, via `kubectl` |
| Dry-run diff before apply | Tool guardrail | Yes |
| Redaction before sending to a model | Tool guardrail (defence in depth) | Yes |
| Local activity trail | Tool guardrail (convenience, incident reports) | Yes |

**Principles**

1. **Never exceed the user.** All calls run as the signed-in user; the tool never holds its own credentials.
2. **Rely on the platform for enforcement.** If a rule must hold, it belongs in RBAC, admission policy or IAM, and the tool surfaces it rather than reimplementing it.
3. **Read-only by default.** Writes only exist once an owning team and the firm's prod access process are in place.
4. **Structural AI safety.** The model gets read-only tools and no outbound channels; approvals happen outside the model.
5. **Data classification first, redaction second.** Only data classes approved for a destination are sent; redaction catches mistakes.
6. **Honest claims.** The doc and the tool never describe a guardrail as a security control.

**Threats and mitigations**

| Threat | Example | Mitigation |
| --- | --- | --- |
| Customer data sent to a model | Account IDs in application logs | Classification ruling per destination; per-cluster AI switch; redaction as backup; no prod data until approved |
| Exfiltration through the assistant | An injected log line makes Copilot post data via its web or terminal tools | MCP mode limited to non-prod; no outbound tools in the built-in agent; assistant tool set reviewed |
| Prompt injection driving changes | Log says "delete the namespace" | Model has no write tools; proposals executed only after approval in the tool, which re-derives the diff |
| Malicious kubeconfig | Exec plugin points to an attacker binary | Allowlist of known plugin commands; warn on unknown ones; never auto-load kubeconfigs from repos |
| Terminal escape injection | Pod log contains ANSI sequences that rewrite the screen | Strip control characters from all rendered cluster data |
| Wrong-cluster mistake | User thinks they're on dev | Tier banner, confirm prompts (guardrail); RBAC keeps prod read-only for most users (enforcement) |
| API server overload | Hundreds of users with cluster-wide watches on prod | Namespace-scoped, lazy watches; client QPS limits; load test; coordinate with platform team on API Priority and Fairness |
| Credential leakage | Tokens in the tool's logs | Tokens never logged; the tool doesn't write tokens itself (note: cloud CLIs cache their own tokens on disk, outside the tool's control) |
| Supply chain | Compromised dependency | Internal mirror only, pinned `go.sum`, vulnerability scans, SBOM, signed releases |

## Architecture and components

A single Go binary holds the tool layer; the CLI, MCP server and TUI are thin surfaces over it. There is no server to host and no central credential store.

&#91;embedded content: architecture · 3 front ends, 6 core modules, 4 external systems\]

Every surface goes through the same tool layer, so RBAC checks, redaction, control-character stripping and the activity trail apply equally to the TUI, the CLI and Copilot.

**Core modules (Go)**

| Module | Responsibility | Key libraries |
| --- | --- | --- |
| Cluster connector | Loads kubeconfig, runs allowlisted exec auth plugins, context switching, tier labels | `client-go`, `clientcmd` |
| Resource access | Namespace-scoped, lazy watches; paginated lists; client QPS limits; CRDs via discovery + dynamic client | `client-go/informers`, `dynamic`, `discovery` |
| Triage rules | Deterministic checks for the top 15 failure patterns; runs with or without AI | stdlib |
| Change timeline | Kubernetes-native history first (ReplicaSet revisions, ConfigMap hashes, Helm release records); Git and GitOps later | `client-go`, Helm SDK (read) |
| Output sanitiser | Strips terminal control sequences; redacts secrets and configured patterns before any model call | Custom rules + entropy detection |
| Guardrails + dry-run | RBAC pre-checks via SelfSubjectAccessReview, tier-based prompts, server-side dry-run diffs | `authorization/v1` |
| AI orchestrator | Built-in agent when an internal endpoint is approved; typed read tools, token budget, structured output | Model client interface (OpenAI-compatible, Azure OpenAI, Bedrock, Vertex, local) |
| MCP server | Exposes the same read tools over stdio to approved assistants | MCP Go SDK (via mirror) |
| Activity trail | Local JSON lines for incident reports; optional SIEM forwarding. Not the audit record of truth | stdlib |
| Metrics adapters | Prometheus and Loki queries from per-team templates; optional | Prometheus API, Loki HTTP API |

**Proposed repository layout**

```
cmd/myk8s/            main, Cobra commands (incl. `mcp`)
internal/connect/     kubeconfig, plugin allowlist, tiers
internal/access/      scoped watches, lists, QPS limits
internal/triage/      deterministic rules
internal/history/     change timeline
internal/sanitise/    control chars, redaction
internal/guard/       RBAC pre-checks, dry-run, prompts
internal/trail/       activity trail + SIEM forwarder
internal/ai/          orchestrator, tools, model clients
internal/mcp/         MCP server
internal/tui/         Bubble Tea views
test/e2e/             kind clusters + fault scenarios
eval/                 incident replays + scorer
```

## Multi-cloud connectivity

Support every provider through the standard kubeconfig and client-go's exec credential plugin mechanism, not through custom per-cloud auth code. This is how `kubectl` itself works, so anything `kubectl` can reach, the tool can reach, with the same identity and no new secrets.

| Platform | How auth works | What the tool does | Notes |
| --- | --- | --- | --- |
| Amazon EKS | IAM identity → token from `aws eks get-token` (exec plugin); mapped via EKS access entries | Uses kubeconfig as-is; optional cluster listing via AWS SDK with the user's profile/SSO | Private endpoints need VPN/Direct Connect; respect `AWS_PROFILE` and SSO sessions |
| Azure AKS | Entra ID via `kubelogin` exec plugin; Azure RBAC or Kubernetes RBAC | Uses kubeconfig; warn if a context uses local admin credentials | Private clusters need network path; recommend local accounts disabled |
| Google GKE | Google identity via `gke-gcloud-auth-plugin` | Uses kubeconfig; optional listing via GKE API | Private clusters via VPN or Connect Gateway |
| OpenShift | OAuth token from `oc login`, or OIDC | Uses kubeconfig; understands Routes, DeploymentConfigs, Projects as CRDs | Token expiry handled by re-login prompt |
| Rancher / RKE2 | Rancher-issued token or OIDC | Uses kubeconfig; treats Rancher proxy URL like any API server | Prefer short-lived tokens |
| On-prem / vanilla | OIDC via kubelogin, or client certificates | Uses kubeconfig; flags long-lived client certs and static tokens as a risk | Custom CA bundles supported |
| Local (kind, minikube) | Client certs | Used for development and the test suite | Never treated as prod |

**Connectivity rules**

- No cloud credentials are stored by the tool. It calls the exec plugins the user already has installed.
- Exec plugin commands are checked against an allowlist (`aws`, `kubelogin`, `gke-gcloud-auth-plugin`, `oc`, and firm-approved others); unknown commands need explicit user confirmation, since a kubeconfig can otherwise run any binary.
- The tool never writes tokens itself. Cloud CLIs keep their own token caches on disk; the doc and the tool say so rather than claiming tokens exist only in memory.
- TLS verification is always on; `insecure-skip-tls-verify` contexts are blocked by default.
- Corporate proxies (`HTTPS_PROXY`, `NO_PROXY`) and custom CA bundles are honoured.
- Each context gets a tier label (dev / test / prod) from a central config or a cluster label; it drives guardrails and the banner colour.
- Multi-cluster views use per-cluster timeouts so one unreachable private cluster doesn't freeze the tool.
- Cloud SDK features (listing clusters) are optional add-ons, keeping the core binary's dependency list small.

**Where it runs**

| Location | Typical access | What works |
| --- | --- | --- |
| Managed laptop | Public or VPN-reachable API endpoints | CLI, TUI, MCP in VS Code |
| VDI | Private endpoints via corporate network | CLI, TUI; MCP only if VS Code and Copilot are available there |
| Jump host / bastion | Private clusters, often prod | CLI and TUI only; no MCP, no GUI |

Because the most sensitive clusters are usually reached from jump hosts, the CLI and TUI must be fully useful on their own. Distribution is through the internal artifact repository, with a package for each managed platform and a simple version check against that repository.

## Security controls in detail

Each control is a release requirement for the phase that introduces it, and each is labelled as platform-enforced or tool guardrail.

### Identity and access

- **\[Platform\]** All calls run as the user; RBAC and cloud IAM decide what's allowed. Impersonation only where admins configure it, and logged by the API server.
- **\[Guardrail\]** `SelfSubjectAccessReview` before showing an action; `SelfSubjectRulesReview` to scope views. Namespace-scoped users must work fully without cluster-wide list permissions.
- **\[Guardrail\]** Idle timeout locks the TUI and clears the tool's in-memory state.

### Writes

- **\[Platform\]** Prod writes happen only through the firm's existing just-in-time or privileged access process. The tool does not build its own two-person approval.
- **\[Platform\]** Dangerous changes (namespace, CRD, PV deletion; RBAC edits; privileged pods) are blocked by admission policy owned by the platform team, not by the tool.
- **\[Guardrail\]** Writes disabled by default; enabled per tier by central config.
- **\[Guardrail\]** Every write: RBAC pre-check → server-side dry-run → diff → confirm → apply → verify. Prod tier asks for the cluster name and a change-ticket ID, recorded in the activity trail.
- **\[Guardrail\]** Pre-flight warnings: PodDisruptionBudgets, StatefulSets, HPA conflicts, GitOps ownership (Argo CD/Flux will revert manual changes), known migrations.

### Data handling

- **\[Platform\]** Written data-classification ruling for each destination (internal LLM, Copilot) before any cluster data is sent. Until then, AI features only run against dev clusters with synthetic workloads.
- **\[Guardrail\]** Secret values never fetched by default; revealing one is explicit, masked and logged.
- **\[Guardrail\]** Plain env values and ConfigMap data treated as potentially sensitive: shown locally, redacted before any model call.
- **\[Guardrail\]** Redaction for known token formats, private keys, connection strings, card numbers, emails, high-entropy strings and firm-specific patterns, with its own test corpus and fuzz tests. Redaction is defence in depth, not the primary control.
- **\[Guardrail\]** Control characters and ANSI sequences stripped from all cluster data before rendering.

### AI safety (structural)

- Built-in agent: read-only typed tools, no shell, no HTTP, no outbound channel of any kind.
- MCP mode: read-only tools only; limited to non-prod clusters until the assistant's other tools (terminal, web, GitHub) can be controlled for these sessions.
- Proposals are executed by the guardrail path, which re-derives the diff; the model's description of a change is never trusted.
- Per-cluster switch to disable AI entirely; hard caps on tool calls and tokens per run.
- Saved runs (redacted prompts, tool calls, results) for review, with a set retention period.

### Audit

- **\[Platform\]** The audit record of truth is the Kubernetes API audit log plus cloud audit logs; they record the real user regardless of which tool was used.
- **\[Guardrail\]** Local activity trail in JSON lines for incident reports and AI run replay, optionally forwarded to the SIEM as it happens. Not presented as compliance evidence.

### Application and supply chain

- Single static Go binary; no runtime-loaded plugins.
- Dependencies only via the internal mirror; pinned `go.sum`; dependency count reviewed each release.
- CI: `govulncheck`, `gosec`, `staticcheck`, secret scanning, licence check, SBOM (CycloneDX or SPDX).
- Signed binaries and checksums (cosign or the firm's signing service); provenance attestation.
- FIPS-capable build if policy requires it.
- No telemetry; usage counts, if any, are opt-in and stay internal.
- Independent security review before the first pilot, and a penetration test before any write feature.

## AI agent design

The agent is only as good as the evidence it gathers and the test suite that measures it, so deterministic pre-checks and an evaluation harness come before clever prompting.

### Investigation flow

1. **Scope:** resolve the target ("checkout" → Deployment `checkout` in namespace `shop`, cluster `prod-eu`). Ask if ambiguous.
2. **Deterministic triage (no LLM):** run built-in rules for the common failures: ImagePullBackOff, CrashLoopBackOff exit codes, OOMKilled, pending pods with scheduling reasons, failing probes, missing Secret/ConfigMap, empty Endpoints, PVC not bound, quota exceeded. Many cases end here, fast and free.
3. **Change timeline:** collect what changed recently: rollout revisions, image tags, ConfigMap/Secret versions, Helm release records, HPA and node events. Git and GitOps history are added later, once repo access per team is solved.
4. **Targeted evidence:** the LLM picks further read tools (logs from failing vs healthy pods, metrics around the change time, dependent Services).
5. **Diagnosis:** likely cause, confidence level, evidence list with links to each resource, and what was ruled out.
6. **Recommendation:** suggested action with blast radius and rollback plan; handed to the approval gate if the user wants to act.

### Tool set

| Tool | Kind | Guardrail |
| --- | --- | --- |
| `get_resource`, `list_resources` | Read | RBAC-checked, Secrets return metadata only |
| `get_events` | Read | Time-windowed |
| `get_logs` | Read | Line and byte caps, redacted, current + previous container |
| `get_metrics` | Read | PromQL templates configured per team, since label conventions differ; no free-form queries in v1 |
| `get_rollout_history` | Read | ReplicaSets, Helm, GitOps status |
| `diff_revisions` | Read | Spec diff between two revisions, Secret values masked |
| `get_endpoints`, `get_network_policies` | Read | Namespace-scoped |
| `propose_action` | Proposal | Produces a structured proposal only; the approval gate executes it |

### Context and model handling

- Summarise before sending: logs are clustered (repeated lines collapsed with counts), events deduplicated, manifests trimmed to relevant fields.
- Per-run token budget with a clear message when evidence was truncated.
- Model client interface supporting any OpenAI-compatible endpoint, Azure OpenAI, AWS Bedrock, Vertex AI, and local models (vLLM, Ollama) so the firm's approved endpoint plugs in by config.
- Structured output (JSON schema) for diagnoses so the UI can render evidence and actions reliably.
- Deterministic replay: every run saves its tool calls and results so a diagnosis can be reproduced and reviewed.

### Evaluation

The AI must prove value over the deterministic rules on real incidents; synthetic single-fault scenarios only guard against regressions.

**Three test sets**

| Set | Source | Purpose |
| --- | --- | --- |
| Regression scenarios | 13+ injected faults on kind (bad image, OOMKill, probe, missing Secret, selector mismatch, NetworkPolicy, unschedulable, PVC, bad config in a new revision, quota, DNS) | Catch regressions on every build; most should be solved by rules alone |
| Incident replays | Real past incidents from the pilot teams' postmortems, rebuilt as cluster snapshots plus the logs and events of the time | Measure accuracy on multi-factor causes, which is what matters in practice |
| Adversarial | Prompt injection in logs and annotations, secrets planted in env vars and logs | Prove zero unsafe proposals and zero leaks |

**Metrics**

- **Marginal value over rules:** diagnosis accuracy with rules + AI minus rules alone, on incident replays. If the gain is small, the AI is not worth its approval and running cost.
- **Calibration:** when the agent says "high confidence", how often it is right; wrong answers at high confidence count double.
- **Abstain rate:** how often it correctly says "not enough evidence" instead of guessing.
- **Time to cause:** minutes from question to correct diagnosis, compared with the pilot team's usual process.
- **Safety:** zero unsafe proposals in the adversarial set; zero secret values found when scanning saved prompts.

In MCP mode through Copilot the suite can't run automatically in CI; incident replays are then reviewed by hand each release, or run against an approved local model.

## AI backends and the MCP fallback

If no internal LLM endpoint is approved at day 0, the tool runs as an MCP server and an approved assistant such as GitHub Copilot does the reasoning; the security controls stay in the tool either way.

**Two backends, one tool layer**

| Backend | Who drives the investigation | When to use |
| --- | --- | --- |
| Built-in orchestrator | The tool calls the internal LLM endpoint and renders diagnoses in the TUI | Once an internal endpoint with tool calling is approved |
| MCP server mode (`myk8s mcp`) | Copilot (VS Code agent mode) or another approved MCP client calls the tool's read tools | Day 0 fallback, and a permanent option for developers who live in their editor |
| No LLM | Deterministic triage rules only | AI disabled for a cluster, or no assistant approved |

**How MCP server mode works**

- Runs locally over stdio as the signed-in user; no network listener.
- Exposes the same typed read tools as the built-in agent, plus `run_triage` for the deterministic checks.
- Every tool result passes the redaction filter; Secret values are never returned.
- Every tool call is written to the audit log with the client name.
- MCP mode is limited to dev and test clusters by default. In the assistant, these tools sit next to its terminal, web and GitHub tools, so an injected log line could steer it to send data out; prod access over MCP needs that tool set locked down and a data-classification ruling first.

**Writes over MCP**

- Early development: no write tools exposed over MCP at all.
- Later: write tools only return a proposal ID. Approval happens out of band in the tool's own TUI or CLI (`myk8s approve <id>`), which shows the server-side dry-run diff. The assistant can never approve its own change, and the editor's own tool-confirmation prompt is not treated as the approval gate.

**Checks with the organisation**

- Copilot Business/Enterprise admin policy must allow MCP servers; some organisations also require servers to be registered on an allowlist.
- Get a written ruling on which cluster data (logs, manifests, events) may be sent to Copilot's models; logs can contain customer data.

**Trade-offs versus the built-in orchestrator**

- The automated evaluation suite can't run through Copilot in CI; use manual review runs, or an approved local model for automated runs.
- Less control over prompts, token budgets and output format, so diagnoses vary more.
- No model integration work, and developers get it inside their editor.
- The model client interface stays in the design, so switching to the internal endpoint later needs no rework of the tools.

## Phased roadmap

Roughly 10 months to a production-ready v1 was the original engineering estimate. With security reviews, approvals and a pilot, plan for 12 to 18 months elapsed; the gates matter more than the dates.

&#91;embedded content: revised roadmap · 6 phases, 5 gates\]

Phases 0 to 3 can start as a side project with a sponsor's blessing; Phase 5 cannot, because prod write tooling needs an owning team, support and on-call. Phase 2 is the core to protect if time runs short.

### Deliverables per phase

**Phase 0 — Validate and sponsor**

- [ ] Interview 5 teams: current workflow, pain points, time to diagnose recent incidents
- [ ] Find a sponsor (manager or platform/SRE lead) and check what the platform team already runs or plans
- [ ] File the open-source intake request for K9s (and Headlamp); decide build vs adopt when it's answered
- [ ] Request data-classification rulings for the internal LLM and for Copilot
- [ ] Check Copilot's MCP policy with its admins
- [ ] Agree IP and open-source position with the open-source office

**Phase 1 — Foundations**

- [ ] Confirm the mirror serves `client-go`, Cobra, Bubble Tea, Helm SDK, the MCP Go SDK
- [ ] Threat model and design doc; security design review
- [ ] CI: build, tests, `govulncheck`, `gosec`, SBOM, signing
- [ ] kind clusters in CI; one dev cluster per cloud for nightly auth tests
- [ ] API load budget agreed with the platform team

**Phase 2 — Secure tool layer, read-only**

- [ ] Connector with exec-plugin allowlist and tier labels
- [ ] Namespace-scoped, lazy watches with QPS limits; load test
- [ ] Output sanitiser: control characters, redaction with test corpus
- [ ] Triage rules for the top 15 failure patterns
- [ ] Kubernetes-native change timeline
- [ ] CLI (`myk8s get`, `myk8s triage`, `myk8s timeline`) and `myk8s mcp`, non-prod only
- [ ] Regression scenario suite in CI

**Phase 3 — Lean terminal UI (or K9s plugin)**

- [ ] Context switcher with tier banner; resource lists with filter and live updates
- [ ] Describe, YAML, events, logs; triage and timeline views
- [ ] Tested on laptops, VDI and jump hosts

**Phase 4 — Built-in AI and incident replays**

- [ ] AI orchestrator on the approved endpoint, read-only tools, structured output
- [ ] Incident replay set from pilot postmortems; calibration and abstain metrics
- [ ] Adversarial set (injection, planted secrets)
- [ ] Decision: keep, narrow or drop the built-in AI based on its gain over rules

**Phase 5 — Writes via the firm's access process**

- [ ] Owning team, support model and on-call agreed
- [ ] Restart, scale, rollback with dry-run diffs and pre-flight warnings, non-prod first
- [ ] Prod writes only through JIT access; admission policies agreed with the platform team
- [ ] Penetration test

## Features added beyond the original idea

These close the gaps that decide whether an enterprise team adopts it; items that only made sense with a desktop UI are deferred.

| Feature | Why it matters | Phase |
| --- | --- | --- |
| Tier banner and confirm prompts | Prevents acting on the wrong cluster (guardrail) | 2–3 |
| RBAC-aware views | Namespace-scoped users can use it without cluster-admin | 2 |
| Deterministic triage rules | Most failures are known patterns; instant, free, works with AI off | 2 |
| Kubernetes-native change timeline | The best root-cause signal; no Git access needed | 2 |
| MCP server mode | Day-0 AI through Copilot on non-prod, with no model integration work | 2 |
| Exec-plugin allowlist | Stops a malicious kubeconfig from running arbitrary binaries | 2 |
| Control-character stripping | Stops logs from manipulating the terminal | 2 |
| Scoped, lazy watches with QPS limits | Keeps API server load acceptable when many people use it | 2 |
| Works on jump hosts and VDI | Reaches the private clusters that matter most | 3 |
| Evidence links and confidence in diagnoses | Users can verify claims; confidence is measured, not guessed | 4 |
| Incident replay evaluation | Proves the AI's value over rules on real incidents | 4 |
| Runbook links | Points diagnoses to the team's existing runbooks | 4 |
| Dry-run diffs and pre-flight warnings | Shows exact changes; flags PDBs, StatefulSets, GitOps ownership | 5 |
| Incident report export | Timeline, evidence and actions for postmortems | 4–5 |
| Security posture checks | Privileged pods, hostPath, missing limits, `latest` tags | later |
| Right-sizing hints | Requests vs actual usage from Prometheus | later |
| Resource graph, fleet view, desktop UI | Useful, but only if users ask once the TUI is in use | on demand |

## Testing, release and adoption

Every pull request runs unit, integration, security and regression-scenario checks; nothing merges red.

**Testing layers**

| Layer | What it covers | Tooling |
| --- | --- | --- |
| Unit | Guardrail decisions, triage rules, redaction, sanitiser, parsers | Go `testing`, table tests, fuzzing for redaction and sanitiser |
| Integration | Real API server behaviour, watches, RBAC denials, namespace-only users | `envtest`, kind |
| End-to-end | CLI, MCP and TUI flows on seeded clusters | kind, scripted terminal tests (teatest) |
| Multi-cloud | Exec plugins and private endpoints on real EKS, AKS, GKE dev clusters | Nightly job with least-privilege identities |
| Load | API server requests and watch counts for N simultaneous users | kind or a shared dev cluster; budget agreed with the platform team |
| AI | Regression scenarios in CI; incident replays and adversarial set each release | `eval/` scorer |
| Security | Vulnerabilities, insecure code, secrets in repo | `govulncheck`, `gosec`, secret scanner |

**Release pipeline**

1. Reproducible build with a pinned Go toolchain and `-trimpath`.
2. All test layers; AI scoreboard published as a build artifact.
3. SBOM and provenance attestation.
4. Sign binaries and checksums.
5. Publish to the internal artifact repository, with packages for laptops, VDI and jump hosts.
6. Release notes call out security-relevant changes.

**Adoption**

- One pilot team that lives in raw `kubectl`; weekly feedback.
- One-page security summary for reviewers: data flows, what leaves the machine, enforced controls versus guardrails.
- Short docs: install, connecting to each cloud, tiers, AI data handling.
- Measure before and after on the pilot team's real incidents.

## Ownership and lifecycle

Internal tools without an owning team fade out, so ownership is a gate for Phase 5 and a goal from Phase 2.

- **Owner:** you plus a second contributor from the pilot team by Phase 2; a named owning team (platform or SRE) before any write feature.
- **Kubernetes version support:** the cluster versions the firm runs, tracked against `client-go`'s supported skew; tested in CI on the oldest and newest.
- **Patching:** dependency and CVE updates within the firm's patch windows; `govulncheck` failures block release.
- **Support:** an internal channel, documented troubleshooting, and a clear "not for use during incidents until pilot is complete" note early on.
- **Exit plan:** if adoption stays low after Phase 3, keep the MCP server and triage rules, and stop the TUI.

## Risks, metrics and open decisions

The biggest risks are approvals and ownership, not engineering, which is why Phase 0 exists.

**Risks**

| Risk | Impact | Mitigation |
| --- | --- | --- |
| No sponsor or no validated need | Tool built nobody adopts | Phase 0 gate; stop if interviews don't show the pain |
| K9s gets approved | TUI work duplicated | Build-vs-adopt decision in Phase 0; ship the tool layer as a K9s plugin and MCP server |
| No data-classification ruling for logs | AI limited to synthetic dev data | Triage and timeline still deliver value; request rulings in Phase 0 |
| Dependencies missing from the mirror | Stack blocked | Checked in Phase 1; small dependency list |
| Guardrails mistaken for security controls | Failed security review, false confidence | Enforcement boundary table; platform-enforced controls only |
| AI adds little over rules | Cost without benefit | Marginal-value metric; decision point in Phase 4 |
| API server load at scale | Platform team blocks rollout | Scoped watches, QPS limits, load test, agreed budget |
| No owning team | Tool fades; prod writes unsafe | Ownership gate before Phase 5 |

**Success metrics**

- Time from alert to identified cause for the pilot team, before versus after.
- Share of incidents where triage rules alone found the cause, and the AI's gain on top.
- Calibration: accuracy of high-confidence diagnoses.
- Zero security findings rated high, zero secret values in saved prompts.
- Weekly active users and teams after each phase.

**Open decisions**

- [ ] Build vs adopt: outcome of the K9s intake request
- [ ] Which LLM endpoint is approved, and for which data classes?
- [ ] Is Copilot's MCP policy enabled, and which cluster data may it see?
- [ ] Who is the sponsor, and which team would own the tool long term?
- [ ] Where writes' approvals come from: which JIT or privileged access process applies
- [ ] Open-source position agreed with the open-source office

**Pre-flight checklist (before any code)**

- [ ] Sponsor agreed and platform team consulted
- [ ] 5 team interviews done, pain confirmed
- [ ] K9s intake request filed
- [ ] Data-classification rulings requested
- [ ] Threat model reviewed by a security colleague
