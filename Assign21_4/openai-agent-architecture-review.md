# OpenAI Agent Architecture Review

**Enterprise internal support platform | Design review | 21 October 2026**

## 1. Recommended architecture

Use the **OpenAI Agents SDK in the enterprise application runtime** as the orchestrator. Keep identity, authorization, workflow persistence, tool execution, approvals, and audit ownership in enterprise services. Use Responses through the SDK for model interaction; do not let the model directly access internal systems or make authorization decisions.

```text
 Employee
    │ SSO; authenticated support UI
    ▼
 Support API ── identity / tenant scope / rate limits / input checks
    │
    ▼
 Triage & orchestrator (Agents SDK; policy-aware routing)
    ├── Knowledge specialist ── ACL-filtered, read-only search
    ├── ITSM specialist ─────── ticket lookup / draft / update tools
    └── Identity specialist ─── account diagnostics / recovery tools
              │ typed, least-privilege tool wrappers
              ▼
      Policy + tool gateway ── validation, authorization, idempotency
        ├── KB / ITSM / CMDB / IAM connectors
        └── approval-required action ── approval queue ── human reviewer
                                                   │ approve / reject
                                                   ▼
                                         resume saved run; execute if approved

 App-owned encrypted workflow/session store ── run state, approval, ownership
 SDK traces + app audit/metrics ── redacted observability and evaluation
```

| Agent | Responsibility and allowed capability |
|---|---|
| Triage/orchestrator | Classify intent and impact, select one specialist, retain final-answer ownership for routine requests, and enforce routing policy. No direct write tools. |
| Knowledge specialist | Answer from access-controlled internal knowledge; cite retrieved sources and say when evidence is insufficient. Read-only. |
| ITSM specialist | Retrieve tickets and prepare or perform scoped ticket actions through typed tools. Writes are approval-gated by policy. |
| Identity specialist | Diagnose account-access issues and propose recovery. Narrow identity tools only; credential, privilege, and MFA changes require explicit review. |

Start with these bounded roles; add a specialist only when its tool set, policy, or output contract materially differs. Use agents-as-tools when triage should synthesize the answer; hand off only when a specialist should own the next response. Define explicit maximum handoffs/tool steps and a safe escalation path.

### Routing and handoff logic

1. Authenticate the caller and bind immutable user, tenant, role, and request IDs server-side. Reject unauthorized scope before model execution.
2. Apply input checks; classify intent, urgency, sensitivity, and ambiguity. Use deterministic rules for known high-impact intents; otherwise let triage select among allowlisted specialists.
3. Route knowledge questions to read-only retrieval; incident/ticket requests to ITSM; account recovery to Identity. Mixed requests can invoke bounded helpers, but authorization is rechecked for every tool.
4. If confidence/evidence is inadequate, intent conflicts, permissions are missing, or risk is high, ask a clarifying question or escalate to a human—never guess or broaden access.
5. Specialists return typed results and evidence. The orchestrator checks output policy, summarizes, and identifies any proposed action before responding.

### Guardrails and approval boundary

**Input:** authenticate/authorize, constrain length/type, detect secrets and prompt-injection attempts, classify sensitive content, and reject prohibited requests. Treat retrieved documents and tool results as untrusted data, not instructions. Block before costly or risky work where possible.

**Tools:** expose narrow server-side functions, not general-purpose credentials or arbitrary SQL/shell. Validate schema, tenant/resource ownership, caller authorization, allowed fields, and business invariants on every call. Use least-privilege service identities, timeouts, rate limits, audit events, and idempotency keys. Read-only operations may proceed automatically; all mutations pass through the same policy gateway.

**Output:** validate structured output, remove secrets/unauthorized data, verify citations against retrieved evidence, and avoid unsupported claims. Fail closed with a safe explanation or human escalation when validation fails.

**Human review:** pause before account unlock/reset or MFA/role changes; ticket closure, reassignment, or external communication; data export/deletion; and any ambiguous or high-impact mutation. Show reviewer the requester, target, exact arguments, rationale, evidence, and impact. Require an authenticated, authorized reviewer distinct from the requester for sensitive actions. Record approve/reject, actor, timestamp, and policy version. Approval is bound to the exact action and expires; revalidate authorization and current target state on resume. Never interpret model text as approval.

\newpage

## 2. Workflow, operations, and risk

### Stateful workflow and recovery

Use an **Agents SDK session** for conversational continuity, with a separate application-owned workflow record as the source of truth for enterprise identity, status, authorization context, tool idempotency, approval, and retention. Persist session/run state encrypted and access-controlled; key it by tenant, user, and conversation. Store references to source documents rather than unnecessary sensitive content. Apply retention/deletion policy and keep secrets out of prompts and session history.

Each turn creates a correlated workflow/run ID. On normal completion, persist the final response and session reference. On approval-required tool interruption, persist the resumable SDK state and a pending-approval record; show the reviewer the immutable proposed action. Resume the same saved run only after a recorded decision, then re-check authorization, action expiry, and resource version before executing. Reject stale, duplicate, or altered approvals; make side effects idempotent. On timeout, tool failure, or uncertain state, do not claim success—reconcile against the system of record and offer safe retry or human escalation.

If the Responses API is used directly instead, explicitly manage conversation state with Conversations or `previous_response_id`, plus the same application-owned workflow/approval state. Choose and document storage/retention behavior (`store` settings) rather than assuming a model conversation is the business audit record.

### Tracing, observability, and quality

- Enable Agents SDK tracing for runs, model calls, tool calls/results, handoffs, guardrails, approvals, and custom spans. Correlate traces with application request, tenant, workflow, and approval IDs without putting raw secrets or full sensitive payloads in tags.
- Send structured, redacted audit events to the enterprise SIEM; protect trace access, define retention, and separate diagnostic traces from immutable business audit records.
- Dashboards/alerts: success and containment rates, correct routing, handoff/tool counts, approval wait and rejection rates, tool errors/timeouts, latency, token/cost, guardrail blocks, escalation, and user feedback. Alert on unusual write volume, denied access, repeated tool failures, or bypass attempts.
- Maintain versioned, privacy-reviewed evaluation sets for routing, groundedness/citations, refusal, authorization isolation, and prompt-injection resistance. Run regression tests before prompt/tool/model changes; sample production traces for review and feed approved findings into evaluations.
- Operational controls: staged rollout, kill switch for writes, per-tool quotas, fallback to human support, incident runbooks, and tested restore/reconciliation paths.

### Risk register and mitigation checklist

| Risk | Required mitigation / release check |
|---|---|
| Cross-tenant disclosure or privilege escalation | Server-derived identity; ACL-filtered retrieval; per-tool authorization on every call; negative tests across users/tenants. |
| Prompt injection in user input or retrieved content | Treat content as data; isolate instructions; allowlist tools; validate arguments/results; adversarial test corpus. |
| Unauthorized or unintended side effect | Read/write separation; exact-action human approval; immutable audit; idempotency; concurrency/version checks; write kill switch. |
| Incorrect or fabricated support answer | Ground in approved sources; cite/check evidence; output validation; abstain/escalate; quality evaluations. |
| PII, credentials, or sensitive data exposure | Minimize and redact; no secrets in prompts/logs; encryption and access controls; defined retention and deletion. |
| Tool/API outage, duplicate retry, or partial completion | Timeouts/retries with limits; idempotency; reconciliation; explicit pending/failed status; human fallback. |
| Model/prompt drift, runaway cost, or loop | Pin/version configuration; bounded steps/tokens/time; quotas; trace/eval monitoring; rollback. |
| Approval fatigue or stale approval | Risk-based thresholds; reviewer context; separation of duties; expiry; revalidation; periodic access review. |

**Go-live gate:** demonstrate tenant-isolation and authorization tests; approval pause/reject/resume and replay tests; side-effect idempotency and stale-state tests; prompt-injection and output-redaction tests; trace redaction/access/retention checks; evaluation baselines; operational kill switch and incident runbook. Assign a business owner for each mutation policy and review it periodically.

### Source documentation

- [Agents SDK overview](https://developers.openai.com/api/docs/guides/agents/sdk)
- [Orchestration and handoffs](https://developers.openai.com/api/docs/guides/agents/orchestration)
- [Guardrails and human review](https://developers.openai.com/api/docs/guides/agents/guardrails-approvals)
- [Responses API migration guide](https://developers.openai.com/api/docs/guides/migrate-to-responses)
- [SDK results and state](https://developers.openai.com/api/docs/guides/agents/results)
- [SDK integrations and observability](https://developers.openai.com/api/docs/guides/agents/integrations-observability)
