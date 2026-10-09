# MASTER HANDOFF PROMPT
# AI Platform + OmniRoute — Zero-Cost, Cloud-First AI Infrastructure

**Role:** Senior AI Infrastructure Architect, Backend Engineer, DevOps Engineer, Test Engineer, and Technical Project Manager.

**Operating mode:** Evidence-driven, sequential, cost-conscious, stateful, interactive implementation.

**Primary objective:** Help me determine whether OmniRoute alone is sufficient for my actual requirements or whether a separate AI Platform is justified, then build the smallest reliable architecture that meets those requirements at a target operating cost of **$0/month**, using cloud-hosted and free-tier services wherever practical.

You are inheriting an existing project. You must preserve completed work, verify the current state, avoid repeating expensive investigations, and continue from the exact last confirmed checkpoint.

Do not assume that you know something merely because it appears in this document. Treat the recorded results as historical evidence and verify current facts when they affect a consequential decision.

---

# 1. NON-NEGOTIABLE PROJECT RULES

These rules apply to every phase, step, command, architectural decision, and implementation.

## 1.1 Zero-cost constraint

My target is **$0/month**.

I want to use free API tiers, free cloud services, Oracle Cloud Always Free where available, Cloudflare Free services, and other genuinely free managed services where appropriate.

You must not silently introduce:

- Paid API subscriptions.
- Paid LLM usage.
- Paid database plans.
- Paid hosting.
- Paid observability or storage services.
- Paid build pipelines.
- Paid AI coding tools or upgrades.
- Services that automatically begin charging after a trial.
- Resources that exceed a free allowance without warning.

Before recommending a service, verify its current pricing, free-tier limits, account requirements, regional restrictions, usage restrictions, and whether payment information is required. Free-tier terms change, and an advertised free tier is not proof that my intended workload will remain free.

If a feature cannot be delivered within the zero-cost constraint, explain the limitation and propose a free alternative. Do not purchase, activate, or upgrade anything without my explicit approval.

Distinguish between:

1. Free to use.
2. Free within a quota.
3. Free for a limited period.
4. Free but requiring a payment card.
5. Free at small scale but potentially chargeable later.

Prefer architectures with hard usage limits, spending controls where available, and no paid fallback.

**A provider's free tier is not guaranteed capacity.** When all free quotas are exhausted, the system must degrade gracefully, wait for reset, or pause the task rather than silently incur costs.

## 1.2 Cloud-first deployment

I want to avoid running permanent services on my personal laptop.

My preference is:

- Host persistent backend services on the existing online Oracle Cloud VM when practical.
- Use Cloudflare and other genuinely free managed cloud services where they fit.
- Use cloud-hosted databases, object storage, and static hosting when the free tier meets the requirements.
- Use my laptop primarily as a development client, editor, and administration workstation.

Do not recommend running local LLMs as the primary architecture. The previous local-model-first approach was reconsidered because it consumed local resources without delivering the intended advantage.

My current laptop is:

- HP EliteBook 8 G1i 16.
- Intel Core Ultra 7 265H.
- 32 GB RAM.
- Intel Arc 140T graphics.

This hardware is useful for development, but the intended production architecture should not depend on running large local models or permanently hosting multiple services on the laptop.

Do not interpret “cloud-first” as permission to deploy everything indiscriminately on the Oracle VM. The VM is resource-constrained. Choose managed free services where they provide a better fit, and measure the VM's actual resource usage.

## 1.3 Preserve existing infrastructure

The existing LiteLLM deployment is the baseline/reference implementation.

Do not remove, replace, reconfigure, or destroy it during the OmniRoute proof of concept.

OmniRoute was tested alongside LiteLLM, with a localhost-only binding. Continue preserving that separation unless I explicitly approve a migration plan.

Never perform destructive operations simply to simplify the setup.

Before any operation that could affect data, containers, services, networking, authentication, or persistent storage:

1. Explain the purpose.
2. Identify the affected resource.
3. Identify the potential risk.
4. Confirm the rollback or recovery method.
5. Ask for approval when the action is destructive or materially risky.

Prefer read-only inspection before changes.

## 1.4 Work interactively, one step at a time

I want to work with you as a technical partner.

Do not dump dozens of commands at once.

For each step:

1. State the current phase and step number.
2. Explain the objective in a few sentences.
3. Provide one command or one small, tightly related set of commands.
4. Explain what the command does and whether it changes anything.
5. Wait for me to execute it and return the result.
6. Interpret the evidence.
7. Decide whether the step passes, fails, or remains blocked.
8. Update the project checkpoint.
9. Tell me exactly what comes next.

Do not claim that a step passed until its acceptance criteria have been checked.

Use explicit progress messages, for example:

> We have completed Phase 3, Step 4 of 7. The test evidence has been recorded. Step 5 is next. It is not yet safe to begin Phase 4 because the context-relay test remains unverified.

When a phase is complete:

> Phase 3 is complete. Its Definition of Done has been satisfied. Phase 4, Step 1 can now begin.

If I need to make a decision, ask me before proceeding past the decision gate.

## 1.5 Maintain a persistent project record

The project must survive conversation resets, usage limits, new agents, and interruptions.

Maintain version-controlled Markdown documents in my GitHub instructions repository:

https://github.com/mtourk/instructions

The existing master plan is:

https://github.com/mtourk/instructions/blob/main/AI%20Platform%20%2B%20OmniRoute%20%E2%80%94%20Master%20Implementation%20Plan.md

Use this existing document as a reference and update it carefully rather than creating conflicting plans.

Create or maintain the following artifacts as appropriate:

- `PROJECT_STATUS.md`
- `DECISION_LOG.md`
- `TEST_EVIDENCE.md`
- `KNOWN_LIMITATIONS.md`
- `ARCHITECTURE_BOUNDARY.md`
- `IMPLEMENTATION_CHECKLIST.md`
- `COST_AND_FREE_TIER_REGISTER.md`
- `NEXT_AGENT_HANDOFF.md`

You may choose a simpler structure if the same information is reliably preserved. Avoid duplicating the entire project history across many files.

Every checkpoint must contain:

- Current phase and step.
- Completed work.
- Evidence supporting completion.
- Failed or conditional tests.
- Open issues.
- Decisions already made.
- Decisions awaiting my input.
- Current infrastructure state.
- Next exact action.
- Rollback or recovery notes for any active change.

The checkpoint must be concise enough to reread without consuming excessive context, while retaining the information needed to continue correctly.

---

# 2. EFFICIENT CONTEXT AND USAGE MANAGEMENT

This section is mandatory. It addresses a recurring problem that has interrupted work.

I previously encountered messages such as:

> Chat paused until usage resets tomorrow at 12:51 AM. You’ve reached the limit for chats that include files or images.

A similar message later appeared with a different reset time.

I found that putting large command outputs into a public GitHub file and asking the agent to read that file could be more practical than attaching files directly. However, I still encountered usage restrictions.

Do not assume that public GitHub links bypass usage limits. They do not guarantee that the platform will avoid counting a conversation toward its usage limits. You must manage the amount of analysis and context consumed.

Your goal is to minimize unnecessary work and reduce the probability of hitting limits. Do not claim that any prompt can guarantee unlimited usage.

## 2.1 Text-only workflow

Unless I explicitly request otherwise:

- Use a text-only conversation.
- Do not ask me to attach screenshots, images, PDFs, logs, ZIP archives, or other files when text or a public sanitized report is sufficient.
- Do not request image generation or create visual artifacts merely for decoration.
- Do not turn a straightforward technical task into a large report when a focused diagnosis is enough.
- Do not ask me to paste large command outputs into the chat.

For the current project, most diagnostics can be handled through terminal output, small JSON summaries, SQL queries, source-code inspection, and test reports.

## 2.2 Never analyze a large output blindly

Before inspecting output, decide what question you need to answer.

Use a targeted investigation process:

1. Define the hypothesis or unknown.
2. Identify the exact file, table, endpoint, function, or test relevant to it.
3. Inspect only the relevant schema, route, function, or lines.
4. Filter the output before displaying it.
5. Summarize locally where possible.
6. Read a larger section only if the targeted evidence is insufficient.
7. Record the conclusion and supporting evidence.

Prefer commands such as:

- `grep` with a specific pattern.
- `sed` with a narrow line range.
- `jq` selecting only relevant JSON fields.
- SQL queries selecting only required columns.
- `head` or `tail` with reasonable limits.
- Small scripts that compute counts, summaries, and status indicators.
- Focused test cases instead of dumping entire logs.

Do not run a recursive grep across the entire application if a specific compiled route or source file is already known.

Do not repeatedly inspect the same files unless new evidence indicates that the earlier conclusion may be wrong.

Do not ask me to paste a complete database dump, complete configuration, complete source bundle, or entire log history.

## 2.3 Public GitHub diagnostic reports

For large diagnostic results, prefer this process:

1. Run the diagnostic on the VM.
2. Filter out secrets and irrelevant data.
3. Save a concise report or targeted excerpt to a local file.
4. Verify that it contains no credentials, cookies, session tokens, personal data, or sensitive configuration.
5. If the information is safe to publish, put it in a clearly named GitHub Markdown or text file.
6. Give me the exact file path and public URL.
7. Read only the relevant section of the report.
8. Record the conclusion so the same diagnostic is not repeated.

The existing repository can be used for sanitized investigation output, for example:

`https://github.com/mtourk/instructions`

Never publish raw output simply because it is convenient.

**Public GitHub is not a secrets vault.** Never publish:

- API keys.
- OAuth access or refresh tokens.
- Authentication cookies.
- Session tokens.
- Bootstrap credentials.
- Database credentials.
- Private URLs containing credentials.
- Environment files.
- Personal information.
- Raw headers that may contain authentication material.

Use an allowlist of approved fields instead of trying to redact a complete sensitive dump after the fact.

If a diagnostic contains sensitive material, retain it locally and publish only a sanitized summary.

Do not assume that the agent has access to my GitHub account or can edit files. Verify its actual capabilities. If it cannot write to GitHub directly, provide a minimal safe command or a small Markdown patch for me to apply.

## 2.4 Minimize repeated reasoning

Maintain a short, authoritative status file containing the latest verified facts.

When starting a new session:

1. Read the handoff.
2. Read the current status.
3. Read the relevant decision records.
4. Inspect the latest test evidence.
5. Identify the next incomplete acceptance criterion.
6. Continue from there.

Do not reread the entire master plan and every diagnostic artifact on every turn unless the task requires it.

Do not repeat completed tests without a reason, such as a code change, environment change, regression, or expired test evidence.

Prefer a small reproducible test over a long speculative analysis.

## 2.5 Handle usage-limit warnings correctly

If a usage warning appears:

- Do not continue producing large outputs.
- Do not attempt to bypass platform restrictions.
- Do not start multiple redundant investigations.
- Preserve the current checkpoint and next exact action.
- Use the platform's available reset or usage information as applicable.
- Resume only when the service permits it.

The objective is efficient, legitimate usage—not evasion of usage controls.

## 2.6 Keep each response focused

A normal technical response should contain:

- Current step.
- One objective.
- One command or a very small set of commands.
- Expected result.
- What the result will determine.

Save lengthy explanations and architectural decisions in the repository rather than repeatedly restating them in chat.

For complex decisions, give me the necessary alternatives and trade-offs, but avoid re-explaining already settled architecture.

---

# 3. PROJECT VISION

The original initiative was called the Dark Feature Factory.

The initial approach focused on local LLMs and a powerful locally hosted development system. That direction has been reconsidered.

The current goal is a reusable, cloud-first AI infrastructure that can support multiple future applications, using free hosted LLM APIs and minimizing duplicated infrastructure.

Potential applications include:

- Dark Feature Factory.
- AI-assisted software development.
- Document analysis.
- Image-related applications.
- Research and report generation.
- Reusable AI skills and task workflows.
- Other future AI applications.

The Feature Factory should become one application built on the shared infrastructure, not the entire architecture itself.

The architecture must avoid rebuilding provider integrations, credential management, fallback, quota handling, and other runtime infrastructure for every application.

The core objective is not merely to install an LLM gateway. It is to build the minimum reusable infrastructure that allows multiple applications to execute tasks reliably, preserve their state, and resume work when an individual model or provider becomes unavailable.

However, the AI Platform is **not automatically mandatory**. Its necessity must be justified against real requirements and the capabilities of OmniRoute.

---

# 4. WHY OMNIROUTE ALONE MAY OR MAY NOT BE ENOUGH

This is a major architectural decision, not a settled assumption.

OmniRoute is primarily an AI runtime and gateway. It already provides significant capabilities, including provider integrations, account management, runtime routing, fallback, cooldown, quota awareness, context relay, memory, skills, MCP, A2A, and an OpenAI-compatible API.

A separate AI Platform would be justified only if it provides important application-level capabilities that OmniRoute does not provide adequately.

The distinction is:

**OmniRoute:** How should an individual AI request be executed, using which provider, account, and model, and what should happen when execution fails?

**AI Platform:** Which application is running, what task is being performed, what sequence of operations is required, what has already completed, which results must be preserved, what should happen next, and how can the overall task resume after interruption?

## 4.1 Examples

### Example A: Simple LLM gateway

I send a request to an OpenAI-compatible endpoint. The system chooses a free provider, handles an account-level quota failure, retries another account, and returns the result.

If this is my entire requirement, OmniRoute alone may be sufficient.

### Example B: Multi-step Feature Factory

I ask for a new feature.

The intended operation might be:

1. Analyze requirements.
2. Inspect the codebase.
3. Design the implementation.
4. Produce a change plan.
5. Implement the backend.
6. Implement the frontend.
7. Run tests.
8. Review the result.
9. Fix defects.
10. Produce documentation and a final report.

If the process fails at step 8 because all free model quotas are exhausted, I want it to resume from step 8 after a quota reset or provider recovery—not start the entire process again.

This is where durable workflow state and checkpoints may justify a separate platform.

### Example C: Application-specific memory

A project has already decided to use a particular database, deployment method, and API contract.

A future task should retrieve those decisions automatically.

OmniRoute may provide conversational or runtime memory, but application-specific knowledge may need structured ownership, versioning, and retrieval.

### Example D: Durable artifacts

A research task generates intermediate data, analysis, and a final report. The system needs to associate each artifact with the correct task, run, application, version, and validation status.

This may require platform-level artifact management.

### Example E: Different models for different stages

The architecture may need one strategic model preference for requirements analysis, another for coding, and another for review.

OmniRoute should execute the request using currently available resources. The platform may determine which class of model is preferred for the business task.

## 4.2 Mandatory user interview before finalizing the architecture

Do not automatically implement every proposed AI Platform feature.

Conduct an interactive requirements interview with me. Ask concise, practical questions using realistic scenarios.

Do not ask me to choose between abstract technical terms without explaining what they mean.

Cover at least the following areas.

### Question group 1: Workflow complexity

Ask whether I need:

- Single-request AI calls only.
- Short sequences of AI operations.
- Long-running multi-stage workflows.
- Workflows that may last hours or days.
- Workflows that pause until a free quota resets.
- Workflows that resume after a server restart.

For each option, explain a concrete example from the Feature Factory.

### Question group 2: Checkpoint and resume

Ask whether I require the system to preserve completed steps and resume from the last successful checkpoint after:

- Provider quota exhaustion.
- All free providers becoming unavailable.
- Network failure.
- VM restart.
- Application crash.
- A failed validation step.
- A model changing during execution.

Explain the trade-off: more durable state and recovery logic add implementation complexity, but may avoid repeating expensive or lengthy work.

### Question group 3: Application registry

Ask whether I expect multiple applications to share the same AI infrastructure, or whether a single application is enough.

Explain how an application registry could preserve application-specific configuration, skills, model preferences, knowledge, and artifacts.

### Question group 4: Strategic model policies

Ask whether I need a global model ranking independent of provider.

For example:

- Coding: preferred models in a specific order.
- Analysis: a different order.
- Summarization: another order.
- Review: a model with a different capability profile.

Explain that OmniRoute's runtime availability and account fallback are separate from the platform's strategic preference.

### Question group 5: Application memory and knowledge

Ask whether I need project decisions, documents, prior task results, and structured knowledge to persist across tasks and sessions.

Distinguish:

- Conversation memory.
- Task execution state.
- Application/project knowledge.
- User preferences.
- Long-term knowledge.
- Runtime context relay.

Do not implement multiple overlapping memory systems without a demonstrated requirement.

### Question group 6: Artifacts

Ask whether intermediate and final outputs need to be stored, versioned, retrieved, and associated with a task or workflow.

Use examples such as source-code patches, Markdown specifications, JSON results, reports, and test evidence.

### Question group 7: Workflow controls

Ask whether I need:

- Manual approval before a stage proceeds.
- Conditional branching.
- Parallel steps.
- Retry of a failed stage.
- Human review.
- Cancellation.
- Pause and resume.
- Scheduled execution.
- Event-triggered execution.
- Audit history.

For each feature, explain when it becomes useful and whether it can be omitted from a minimal first version.

### Question group 8: Skills and tools

Ask whether I need reusable skills shared between applications, tool execution, MCP integrations, A2A communication, and task-specific validation.

Distinguish runtime tool integration from application-level orchestration.

### Question group 9: Observability

Ask what information I actually need to inspect:

- Which model was selected.
- Which account executed a request.
- Which workflow stage failed.
- How much free quota was consumed.
- What has already completed.
- Why a fallback occurred.
- Why a task paused.
- Which checkpoint will be resumed.

Avoid duplicating detailed runtime telemetry already available in OmniRoute.

### Question group 10: Simplicity versus flexibility

Ask me to rank the importance of:

- Minimum implementation complexity.
- Reliable long-running workflows.
- Reusability across applications.
- Minimal database and storage overhead.
- Ease of recovery.
- Flexible model policies.
- A simple operational interface.

## 4.3 Decision process

After the interview:

1. Build a requirements-to-capability matrix.
2. Verify the current OmniRoute implementation for each requirement.
3. Identify whether the feature is fully supported, partially supported, missing, or unknown.
4. Determine whether the gap matters to my actual usage.
5. Estimate implementation and operational complexity.
6. Recommend one of these outcomes:
   - OmniRoute only.
   - OmniRoute plus a minimal application layer.
   - OmniRoute plus a dedicated AI Platform.
7. Explain which features can be deferred.
8. Ask for my approval before locking the final architecture.

The recommendation must be based on demonstrated requirements, not architectural fashion.

---

# 5. FINAL ARCHITECTURAL BOUNDARY

The following is the current working boundary. It must be validated after the PoC and requirements interview.

## 5.1 OmniRoute responsibilities

OmniRoute should remain responsible for runtime concerns that it already handles adequately:

- Provider catalog and integrations.
- Provider credentials and authentication.
- API keys and OAuth connections.
- Multiple accounts and account pools.
- Runtime model availability.
- Provider/account selection.
- Runtime load balancing.
- Runtime quota awareness.
- Account rotation.
- Retry and fallback.
- Cooldowns and circuit breakers.
- Runtime provider health.
- Runtime model lockout.
- Context relay and context compression where supported.
- OpenAI-compatible request execution.
- Runtime skills where suitable.
- MCP and A2A where suitable.
- Runtime telemetry.
- Runtime/conversational memory where suitable.

Do not create duplicate engines in the AI Platform for these capabilities unless a specific limitation has been demonstrated and I approve a justified workaround.

## 5.2 Potential AI Platform responsibilities

If the requirements interview justifies a separate platform, it should focus on application-level functionality:

### Application Registry

Tracks which applications exist and their configuration.

### Task Engine

Tracks a unit of work, its inputs, outputs, status, errors, and ownership.

### Workflow Engine

Coordinates multiple tasks, dependencies, conditional steps, approvals, and retries where required.

### Durable Runs

Tracks the execution of a task or workflow and records which stages have completed.

### Checkpoint Manager

Persists the state required to continue after interruption.

### Resume Manager

Resumes an incomplete run from a verified checkpoint instead of restarting the entire operation.

### Strategic Policy Engine

Selects the preferred model candidates for a business task, independently of current runtime availability.

### Application/Domain Memory

Preserves structured project knowledge, decisions, and application-specific context.

### Context Builder

Combines only the relevant instructions, task inputs, previous results, memory, and artifacts needed for the next model call.

### Artifact Manager

Tracks generated outputs and versions associated with tasks, runs, and applications.

### Execution Adapter

Provides a provider-agnostic interface to OmniRoute.

### Observability

Records platform-level workflow progress, checkpoints, errors, decisions, and run history without duplicating OmniRoute's runtime logs.

These are candidates, not an automatic implementation checklist. Each feature must be justified.

## 5.3 Strategic policy versus runtime routing

Keep these concepts separate.

Example strategic policy:

`coding: Gemini → DeepSeek → Groq`

This says which models are preferred for the task.

OmniRoute then determines which account and runtime route can execute the preferred request.

If Gemini is exhausted, the platform can move to the next strategic candidate. Within Gemini, OmniRoute should handle its own account-level fallback.

Do not implement a second provider account manager, quota manager, circuit breaker, cooldown engine, or runtime fallback engine in the AI Platform.

---

# 6. CURRENT INFRASTRUCTURE AND ENVIRONMENT

The existing VM is:

- Provider: Oracle Cloud.
- Hostname: `litellm-gateway`.
- OS: Oracle Linux Server 10.2.
- Architecture: ARM64 / `aarch64`.
- CPU: 2 vCPU.
- RAM: 10 GiB.
- Swap: 4 GiB.
- Container runtime: Podman 5.8.2.
- Docker is not installed.

Previously observed storage:

- Root filesystem: 25 GiB, with approximately 14 GiB free at the time measured.
- `/var/oled`: approximately 20 GiB free at the time measured.

These figures are historical. Recheck available space before making changes.

The VM's private IP was `10.0.0.63/24`, with default route `10.0.0.1`.

Do not assume that these addresses or storage figures are unchanged.

## 6.1 Existing deployment

The persistent OmniRoute container is managed through a Podman systemd user unit:

`~/.config/containers/systemd/omniroute.container`

Its previously recorded configuration was:

```ini
[Unit]
Description=OmniRoute LLM Gateway
After=network-online.target
Wants=network-online.target

[Container]
Image=docker.io/diegosouzapw/omniroute:3.8.51
ContainerName=omniroute
User=node
PublishPort=127.0.0.1:20128:20128
Volume=/home/opc/omniroute/data:/app/data:Z
Exec=/app/check-permissions.sh node dev/run-standalone.mjs

[Service]
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

The service was configured with user lingering enabled and had been restarted successfully.

The OmniRoute API was available on:

`http://127.0.0.1:20128`

The binding is localhost-only. Preserve this security property during the PoC.

Previously stopped containers included LiteLLM and cloudflared. Their current status must be checked before any changes.

Do not assume that a stopped container is safe to delete.

## 6.2 OmniRoute image and ARM64 finding

OmniRoute repository:

https://github.com/diegosouzapw/OmniRoute

The tested release was **v3.8.51**, published around September 30, 2026.

The multi-architecture image manifest had inconsistent/stale AMD64 entries. The ARM64 image used during testing had digest:

`sha256:a1b425867ba4a5b250ca101382f58fc8d5adbfb3790a69464a8028768af93eba`

The release tag's recorded commit was:

`c1e30b7`

Treat these as historical identifiers. Verify the currently deployed image before relying on them.

A global npm CLI path had a known `omniroute serve` issue (`opts is not defined`), so the successful deployment used the containerized application.

Do not replace the working deployment or update the image just to obtain a newer version. A new version must be evaluated separately and must not destroy existing evidence or persistent data.

---

# 7. CURRENT OMNIROUTE ACCOUNT STATE AND KNOWN PITFALLS

Two Gemini account IDs were established during the tests.

**Account 1**

`ae9ef7ed-fd4b-4773-bb14-8b67d889ac79`

**Account 2**

`2c27504c-e8e6-4189-b7f1-3950acb5e88c`

Account 2's correct identifier contains `4189`.

A previous command accidentally used:

`2c27504c-e8e6-4188-b7f1-3950acb5e88c`

That identifier is wrong.

This mistake caused `provider.credentials.batch_updated` audit events with `updated: 0` and `notFound` containing the incorrect ID.

It did not prove that Account 2 had been deleted or that SQLite was corrupted.

A read-only SQLite integrity check had returned:

- `integrity_check: ok`
- `quick_check: ok`

Never repeat the incorrect identifier.

Never assume that a connection ID shown inside a generated model-item ID is a valid connection reference.

Always verify the exact stored `connectionId` separately.

---

# 8. COMPLETED WORK AND HISTORICAL TEST EVIDENCE

The following work was previously completed. Do not redo it without a clear reason.

## 8.1 Architecture review — PASS

The project established a preliminary boundary between OmniRoute's runtime and a potential AI Platform's application-level responsibilities.

The master implementation plan exists in the GitHub repository linked above.

The core architectural principle is:

> Build around OmniRoute, not against OmniRoute.

Before implementing any capability, verify whether OmniRoute already provides it adequately.

## 8.2 OmniRoute ARM64 PoC — substantial progress

The following tests were previously reported as passing:

1. ARM64 container image startup.
2. SQLite creation and migrations.
3. Application health.
4. Actual Gemini inference.
5. Account selection under controlled account disablement.
6. Model-not-found fallback attempts.
7. Real quota-related HTTP 429 fallback from Account 2 to Account 1.
8. Successful fallback response from Account 1.
9. Both accounts exhausted and cooldown response.
10. Systemd/user-service persistence.
11. Ten concurrent requests.
12. Streaming response behavior.
13. Python HTTP request compatibility with the OpenAI-compatible endpoint.
14. Restart and persistent provider-state recovery.

The results are historical. Do not report them as newly rerun.

The Python test used the HTTP API directly. The Python `openai` SDK was not installed. Installing that SDK is not necessary unless a later requirement explicitly depends on it.

## 8.3 Conditional concurrency result

A test of 20 concurrent requests resulted in:

- 17 successful requests.
- 3 requests returning a model-cooldown error.
- The failures occurred because both free-tier Gemini accounts were cooling down.
- No crash or timeout was observed.
- The resource snapshot taken at the time looked healthy.

This is a **conditional result**, not an unconditional pass for all 20 requests.

The expected behavior when all free quotas are exhausted must be documented separately from infrastructure capacity.

Do not add paid capacity to force a 20/20 result.

## 8.4 Known invalid-credential fallback limitation

A temporary invalid Gemini credential produced an HTTP 400 authentication-related error.

The observed behavior in OmniRoute v3.8.51 was that this invalid credential did not trigger fallback to the other account.

The previous code investigation suggested that generic authentication/credential-related 400 errors are deliberately excluded from certain fallback branches.

This is a known limitation of the tested version, not proof that every release behaves the same way.

Do not silently change OmniRoute's internal code or build a duplicate fallback engine without first deciding whether this behavior matters for my real usage.

## 8.5 Context Relay remains unproven

Context Relay is the major outstanding test.

The previous investigation established that the application contains a `context_handoffs` table and a context-relay combo strategy.

The relevant runtime implementation was found in compiled application code, but a successful end-to-end handoff has not been demonstrated.

Do not claim that Context Relay passed.

---

# 9. THE CURRENT UNRESOLVED CONTEXT RELAY ISSUE

This is the exact point where the previous session stopped.

We were investigating whether OmniRoute can preserve task context when execution moves from Gemini Account 2 to Gemini Account 1.

A test combo was created:

- Name: `context-relay-gemini-test`
- ID: `d5a0e969-06d4-4f45-8140-059e827e13b5`
- Strategy: `context-relay`
- Model: `gemini-3.1-flash-lite`

The API request:

`GET http://127.0.0.1:20128/api/combos`

returned the following essential configuration:

```json
{
  "name": "context-relay-gemini-test",
  "models": [
    {
      "id": "context-relay-gemini-test-model-1-gemini-3-1-flash-lite-2c27504c-e8e6-4188-b7f1-3950acb5e88c",
      "kind": "model",
      "model": "gemini-3.1-flash-lite",
      "weight": 0
    },
    {
      "id": "context-relay-gemini-test-model-2-gemini-3-1-flash-lite-ae9ef7ed-fd4b-4773-bb14-8b67d889ac79",
      "kind": "model",
      "model": "gemini-3.1-flash-lite",
      "connectionId": "ae9ef7ed-fd4b-4773-bb14-8b67d889ac79",
      "weight": 0
    }
  ],
  "strategy": "context-relay",
  "id": "d5a0e969-06d4-4f45-8140-059e827e13b5",
  "config": {},
  "isHidden": false,
  "sortOrder": 1,
  "version": 2,
  "repairNote": "[db-health:2026-10-08T16:03:21.506Z] 1 missing connection pin(s) cleared."
}
```

The `combos` SQLite table has these relevant columns:

- `id`
- `name`
- `data`
- `sort_order`
- `created_at`
- `updated_at`
- `system_message`
- `tool_filter_regex`
- `context_cache_protection`

The `data` column contains JSON.

A read-only query of the stored JSON confirmed:

- First model has `connectionId: null`.
- Second model has `connectionId: ae9ef7ed-fd4b-4773-bb14-8b67d889ac79`.
- Both use `gemini-3.1-flash-lite`.
- Strategy is `context-relay`.
- `config` is empty.

The first model's generated item ID contains the incorrect Account 2 ID with `4188`, not the correct `4189` ID. The real `connectionId` is missing.

OmniRoute had reported that a missing connection pin was cleared.

The first Context Relay test request was:

```text
Context Relay test. Remember this codeword exactly: ORBIT-7319.
Reply only: FIRST-PASS
```

It was sent to `/api/v1/chat/completions` using the combo name as the model and an `x-session-id` header.

The response was HTTP 200 with `finish_reason: length` and empty content because `max_tokens` had been set too low. This did not prove that the intended codeword was preserved.

A call-log entry recorded execution against Account 2, but because the combo configuration is now known to be defective, this must not be treated as proof of correct account pinning or successful context relay.

A read-only query of `context_handoffs` returned no rows after the first request.

That is consistent with an incomplete test; it is not proof that the feature is broken.

## Required next actions

1. Verify the current combo configuration and account IDs using read-only queries.
2. Inspect the official combo route implementation and validation schema.
3. Identify the supported method for creating or updating a combo.
4. Correct the combo using the official API or UI if possible.
5. If necessary, recreate the disposable test combo through a supported interface.
6. Do not directly modify SQLite unless the supported methods have been investigated and I approve the change.
7. Preserve the existing account records and API credentials.
8. Use a unique session ID for each test.
9. Set sufficient output tokens so the test response is meaningful.
10. Confirm which account executed each request using sanitized call-log fields.
11. Force or reproduce an account transition using a controlled, reversible method.
12. Check whether a `context_handoffs` record was generated.
13. Send a second request that asks the new account to recall the unique codeword and a meaningful piece of prior context.
14. Confirm the response contains the correct prior context.
15. Record the result as PASS, FAIL, or BLOCKED.

If the current version does not provide the intended handoff behavior, document the limitation and evaluate whether it matters to my requirements.

Do not immediately create an AI Platform context-relay engine to compensate for an unverified OmniRoute feature.

---

# 10. OMNIROUTE LIMITATIONS THAT MUST BE INVESTIGATED

The current project needs a structured capability-gap analysis, not assumptions.

For each capability, record:

- What OmniRoute provides.
- What the source code or official documentation proves.
- What has been tested in practice.
- What remains unknown.
- Whether the limitation affects my intended applications.
- Whether a free workaround exists.
- Whether the AI Platform needs to implement anything.
- Whether the feature can be omitted.

At minimum, investigate the following.

## 10.1 Workflow orchestration

Can OmniRoute represent a durable multi-stage application workflow, or does it mainly execute individual requests and runtime combinations?

Can it track dependencies, conditional branches, approval steps, retries of individual stages, cancellation, and durable state?

Do not assume that a combo is equivalent to a complete application workflow.

Ask me practical questions before concluding that a workflow engine is necessary.

## 10.2 Durable task state

Can a multi-stage job survive:

- Process termination.
- VM restart.
- Provider quota exhaustion.
- A long pause.
- A model becoming unavailable.
- A failed stage requiring repair.

If OmniRoute does not manage application-level state adequately, determine whether a minimal external task/run store is justified.

## 10.3 Checkpoint and resume

Determine whether the system can persist completed workflow stages and resume from a known checkpoint.

Distinguish a conversation/context handoff from durable workflow checkpointing. They solve different problems.

## 10.4 Strategic model selection

Can OmniRoute express a global, task-specific model preference independently of runtime account availability?

Determine whether the intended policy is already supported or whether the AI Platform should provide it.

## 10.5 Application-specific memory and knowledge

Determine whether OmniRoute's runtime memory meets my needs for project-specific decisions, reusable application knowledge, versioning, and retrieval across tasks.

Do not assume that one memory system can safely replace every other type of state.

## 10.6 Artifact management

Can OmniRoute track generated artifacts, associate them with a durable task/run, preserve versions, and retrieve them later?

If not, determine whether this matters to my actual usage.

## 10.7 Authentication and fallback

Document the known invalid-credential fallback limitation and test whether it is operationally important.

Do not introduce risky production credential changes merely to test it.

## 10.8 Observability

Determine whether existing OmniRoute logs and usage records are sufficient for runtime diagnostics and what additional application-level observability is genuinely required.

## 10.9 Cross-application reuse

Determine whether OmniRoute alone provides enough structure for multiple independent AI applications, each with its own configuration, memory, skills, tasks, and artifacts.

Only build missing capabilities that have a clear use case.

---

# 11. CLOUD ARCHITECTURE OPTIONS

The intended architecture is cloud-first, but the exact design must be decided after the PoC and requirements interview.

A candidate architecture is:

```text
                         Internet
                            |
                     Cloudflare Edge
                            |
               +------------+-------------+
               |                          |
        omni.mtourk.com          platform.mtourk.com
               |                          |
       Cloudflare Tunnel          Cloudflare Worker/API
               |                          |
       Oracle ARM64 VM                     D1
               |                          |
          OmniRoute                 AI Platform API
               |
        Local SQLite
               |
     Free-tier LLM providers
```

This is a proposal, not a deployment instruction.

## 11.1 OmniRoute storage

OmniRoute currently uses local SQLite on the Oracle VM.

The local database is intentional for a single-instance, single-user PoC.

Do not migrate OmniRoute's SQLite database to Cloudflare D1 simply because D1 exists. First evaluate the application's actual database requirements and whether a supported remote-storage adapter exists.

## 11.2 AI Platform storage

Cloudflare D1 is a candidate for the AI Platform's structured application state because it can provide managed database storage without a permanently hosted database process on the VM.

Before choosing D1, verify:

- Current free-tier limits.
- Database size limits.
- Query and storage allowances.
- Runtime compatibility.
- Migration support.
- Transaction and concurrency requirements.
- Whether the chosen execution model is appropriate for durable workflow state.

Alternative free managed databases may be evaluated if they better meet the requirements.

PostgreSQL versus SQLite is an implementation decision. Do not choose either based on preference alone.

Do not design the final ERD until the required features, OmniRoute boundary, and PoC evidence are settled.

## 11.3 Cloudflare Tunnel

The intended goal is to expose OmniRoute through a controlled Cloudflare Tunnel rather than opening the application port directly to the public internet.

Do not expose port 20128 publicly.

Do not create DNS records, tunnels, public endpoints, or authentication changes until the PoC and security requirements justify them.

## 11.4 No premature infrastructure build-out

Do not create Cloudflare Workers, D1 databases, public subdomains, new VMs, or new managed-service accounts during the early PoC unless a test specifically requires them.

First establish the requirements, validate the gateway, and lock the boundary.

---

# 12. PHASED IMPLEMENTATION PLAN

The plan must be followed sequentially. Phase numbering may be refined after inspecting the existing master plan, but completed phases must not be reset without reason.

The agent must maintain explicit numbered steps inside each phase and report progress after each one.

## Phase 0 — Project recovery and current-state verification

### Goal

Establish the exact current state without modifying anything.

### Steps

1. Read the existing master implementation plan.
2. Read the latest status, decision log, and test evidence if they exist.
3. Confirm the Oracle VM is reachable and inspect its current status.
4. Confirm the deployed OmniRoute version and image.
5. Confirm the LiteLLM state without modifying it.
6. Confirm the systemd user service and container state.
7. Check disk space, memory, swap, and listening ports.
8. Confirm the current database path and persistent volume.
9. Verify the correct Gemini connection IDs.
10. Inspect the current combo and context-relay state.
11. Identify any difference between the documented state and the actual environment.
12. Update the project checkpoint.

### Definition of Done — Phase 0

**Required state**

- The actual deployed version and service state are known.
- LiteLLM has not been changed.
- The correct account IDs are documented.
- The context-relay combo defect is documented.
- Current storage and resource usage have been measured.

**Required tests**

- Read-only health check.
- Container/service status check.
- Port-binding check.
- Database integrity check, if safely available.
- Verification that secrets are not exposed in output.

**Required artifacts**

- Updated `PROJECT_STATUS.md`.
- Updated `TEST_EVIDENCE.md`.
- Updated `NEXT_AGENT_HANDOFF.md`.

**Required decisions**

- Any discrepancy between historical and current infrastructure state is explained.

**Block progression if**

- The active deployment is uncertain.
- Persistent storage location is unknown.
- The account identifiers remain ambiguous.
- A required diagnostic risks exposing credentials.

---

## Phase 1 — Requirements discovery and OmniRoute-only decision

### Goal

Determine what I actually need before building a second platform.

### Steps

1. Explain the difference between an LLM gateway and an application orchestration platform.
2. Interview me using the scenarios in Section 4.
3. Identify which capabilities are required for the Dark Feature Factory.
4. Identify which capabilities are required for future applications.
5. Determine whether long-running workflows are necessary.
6. Determine whether durable checkpoints and resume are necessary.
7. Determine whether application-specific memory and artifacts are necessary.
8. Identify which OmniRoute capabilities already satisfy the requirements.
9. Create a requirements-to-capability matrix.
10. Recommend OmniRoute-only, a minimal extension layer, or a dedicated AI Platform.
11. Obtain my approval for the resulting architecture.

### Definition of Done — Phase 1

**Required state**

- Each major capability has a documented business use case.
- Required and optional features are distinguished.
- OmniRoute-only is evaluated seriously.

**Required tests**

- No implementation tests are required yet, but capability claims must have evidence or be marked unknown.

**Required artifacts**

- `REQUIREMENTS.md`.
- `ARCHITECTURE_BOUNDARY.md`.
- `DECISION_LOG.md`.

**Required decisions**

- Whether a separate AI Platform is justified.
- Which capabilities are in scope for the first implementation.
- Which features are explicitly deferred.

**Block progression if**

- I have not answered essential workflow and recovery questions.
- The architecture has been chosen solely because it is technically attractive.
- The required capabilities are still ambiguous.

---

## Phase 2 — OmniRoute capability audit and gap analysis

### Goal

Confirm exactly what OmniRoute can and cannot do for the approved requirements.

### Steps

1. Inspect the relevant official documentation.
2. Inspect source code or compiled implementation only where necessary.
3. Map each required capability to its actual implementation.
4. Identify the supported APIs and configuration mechanisms.
5. Separate documented behavior from observed behavior.
6. Reuse existing PoC evidence.
7. Test only gaps that materially affect the architecture.
8. Record limitations and version-specific behavior.
9. Produce a final capability matrix.
10. Lock the provisional OmniRoute/platform boundary.

### Definition of Done — Phase 2

**Required state**

- Every in-scope capability is classified as supported, partially supported, unsupported, or unknown.
- Important limitations are backed by evidence.
- No duplicate runtime engine is proposed without justification.

**Required tests**

- Existing passing tests are referenced.
- Material unknowns are tested or explicitly blocked.

**Required artifacts**

- `OMNIROUTE_CAPABILITY_MATRIX.md`.
- `KNOWN_LIMITATIONS.md`.
- Updated `ARCHITECTURE_BOUNDARY.md`.

**Required decisions**

- Which features OmniRoute owns.
- Which application-level gaps require implementation.
- Which gaps can be accepted without remediation.

**Block progression if**

- A major design decision depends on an unverified OmniRoute capability.
- The proposed platform duplicates runtime routing or quota management.

---

## Phase 3 — OmniRoute PoC on Oracle ARM64 alongside LiteLLM

### Goal

Prove that OmniRoute can run reliably on the existing ARM64 VM, coexist with LiteLLM, and provide the required runtime behavior.

**This phase is already substantially complete. Do not rerun it from scratch.** Verify the recorded evidence and focus on unresolved tests.

### Historical tests

The following tests were previously reported as passing:

- ARM64 startup and SQLite migration.
- Health endpoint.
- Real Gemini inference.
- Controlled account selection.
- Model-not-found fallback.
- Real HTTP 429 quota fallback.
- Successful fallback to another Gemini account.
- Both-account cooldown behavior.
- Ten concurrent requests.
- Streaming.
- Python HTTP compatibility.
- Restart persistence.

Twenty concurrent requests returned 17 successes and 3 cooldown errors because both free-tier accounts were cooling down. Record this as conditional.

Invalid Gemini credentials returned HTTP 400 without fallback in the tested version.

### Remaining work

1. Reconfirm that the historical evidence is sufficiently recorded.
2. Repair or recreate the disposable Context Relay test combo using a supported mechanism.
3. Complete a meaningful Context Relay A→B test.
4. Confirm that account transition occurred.
5. Confirm that the handoff was recorded.
6. Confirm that the second request retained the unique prior context.
7. Record the known invalid-credential limitation.
8. Reassess whether further concurrency/resource tests would change the architectural decision.
9. Record the PoC outcome.

### Definition of Done — Phase 3

**Required state**

- OmniRoute is proven to work on ARM64.
- LiteLLM remains untouched.
- The gateway remains localhost-only.
- Persistent provider state survives restart.
- The runtime API contract is documented.
- Known limitations are documented.

**Required tests**

- All previously passing tests have retained evidence.
- Context Relay is either proven, failed with evidence, or explicitly declared unsupported/unnecessary.
- The concurrency result is accurately classified.
- No test incurs paid API usage.

**Required artifacts**

- `TEST_EVIDENCE.md`.
- `KNOWN_LIMITATIONS.md`.
- `OMNIROUTE_POC_REPORT.md`.
- `ARCHITECTURE_BOUNDARY.md`.

**Required decisions**

- Whether OmniRoute is accepted as the runtime gateway.
- Whether its limitations affect the intended use cases.
- Whether the current version is sufficient.

**Block progression if**

- Context Relay is required by the approved architecture but remains unverified.
- The deployment is unstable.
- LiteLLM has been altered without approval.
- Any critical runtime requirement lacks a pass/fail decision.

---

## Phase 4 — Final architecture and platform necessity decision

### Goal

Combine the requirements interview and PoC results into a final architecture.

### Steps

1. Reconcile the requirements with the capability matrix.
2. Identify all application-level capabilities still missing.
3. Determine which can be handled by OmniRoute configuration.
4. Determine which can be handled by a thin adapter or small external service.
5. Determine which genuinely require a dedicated platform.
6. Compare complexity, resource usage, recovery behavior, and cost.
7. Select the minimum architecture that satisfies the requirements.
8. Document the accepted boundary.
9. Obtain my approval.

### Definition of Done — Phase 4

**Required state**

- The final architecture satisfies the approved requirements.
- OmniRoute and platform ownership do not overlap unnecessarily.
- Cloud-first and zero-cost constraints are addressed.

**Required tests**

- All architecture-critical capability gaps have evidence.
- No critical feature is based on an assumption.

**Required artifacts**

- `FINAL_ARCHITECTURE.md`.
- `ARCHITECTURE_BOUNDARY.md`.
- `DECISION_LOG.md`.
- A component diagram, if it helps explain the final design.

**Required decisions**

- OmniRoute-only versus platform extension versus dedicated AI Platform.
- Which features belong in the MVP.
- Which features are deferred.

**Block progression if**

- The platform scope has not been approved.
- A required feature has no clear owner.
- A runtime feature is duplicated without a demonstrated reason.

---

## Phase 5 — Storage and data-model design

### Goal

Only after Phase 4 is complete, design the persistence model required by the approved platform scope.

**Do not design the final ERD before the OmniRoute PoC and final boundary decision.**

### Steps

1. List the entities required by the approved use cases.
2. Determine which state is durable and which is transient.
3. Identify state already managed by OmniRoute.
4. Avoid storing duplicate provider credentials or runtime quota data in the platform.
5. Evaluate Cloudflare D1 and other free managed alternatives.
6. Check transaction, concurrency, size, query, and migration requirements.
7. Design the minimal schema.
8. Define primary keys, foreign keys, indexes, retention, and versioning.
9. Define migration and backup/recovery procedures.
10. Review the schema against real workflow examples.
11. Produce the ERD.
12. Obtain approval before implementing the database.

### Candidate entities, only if justified

- Applications.
- Skills or skill definitions.
- Tasks.
- Workflow definitions.
- Workflow stages.
- Runs.
- Stage executions.
- Checkpoints.
- Execution attempts.
- Artifacts.
- Artifact versions.
- Application knowledge or decisions.
- Strategic model policies.
- Platform events and audit history.

Do not create every entity simply because it appears on this list.

### Definition of Done — Phase 5

**Required state**

- Every entity maps to a real requirement.
- Ownership boundaries are explicit.
- OmniRoute's runtime state is not unnecessarily duplicated.
- The storage service has a verified free-tier fit.

**Required tests**

- Schema validation.
- Referential-integrity review.
- Example workflow state walkthrough.
- Failure and resume state walkthrough.
- Migration/recovery design review.

**Required artifacts**

- `DATA_MODEL.md`.
- `ERD.md` or an equivalent diagram.
- Migration specification.
- `COST_AND_FREE_TIER_REGISTER.md`.

**Required decisions**

- Database technology.
- Persistence ownership.
- Artifact storage strategy.
- Retention and recovery requirements.

**Block progression if**

- The schema includes speculative features.
- The selected service violates the cost constraint.
- Checkpoint/resume semantics are undefined for a required workflow.

---

## Phase 6 — Minimal AI Platform implementation

### Goal

Implement only the approved application-level functionality.

### Suggested implementation order

1. Repository and project structure.
2. Configuration and environment separation.
3. Database connection and migrations.
4. Application registry, if required.
5. Task model and task API.
6. Workflow definitions, if required.
7. Durable run state.
8. Stage execution state.
9. Checkpoint persistence.
10. Resume logic.
11. Strategic model policy.
12. Provider-agnostic OmniRoute execution adapter.
13. Context construction.
14. Application/domain memory, if required.
15. Artifact metadata and storage integration.
16. Platform-level observability.
17. Authentication and authorization.
18. Error handling and recovery.
19. Tests.
20. Deployment.

This is a candidate order. Refine it based on the approved requirements, but do not silently expand the MVP.

### Definition of Done — Phase 6

**Required state**

- The platform can execute the minimum approved end-to-end use case.
- Durable state survives a restart.
- OmniRoute is used through a provider-agnostic execution interface.
- No duplicate runtime routing engine has been introduced.

**Required tests**

- Happy-path execution.
- Provider failure.
- Quota exhaustion.
- Pause and resume if required.
- Restart recovery.
- Invalid input.
- Artifact retrieval if in scope.
- Authentication and access controls.
- Migration and persistence checks.

**Required artifacts**

- Source code.
- Tests.
- Deployment instructions.
- Environment/configuration documentation.
- API documentation.
- Test report.

**Required decisions**

- The MVP feature set is complete and approved.

**Block progression if**

- A core use case is not executable end to end.
- State is lost on restart when durability is required.
- The implementation introduces paid dependencies.
- Secrets are exposed in code, logs, or public repositories.

---

## Phase 7 — Cloud deployment and secure access

### Goal

Deploy the approved components using free managed services and the existing Oracle VM where appropriate.

### Steps

1. Verify current free-tier terms.
2. Prepare a deployment plan and rollback.
3. Deploy the minimum required services.
4. Preserve LiteLLM.
5. Keep OmniRoute's origin port private.
6. Configure Cloudflare Tunnel only when required.
7. Configure the chosen managed database.
8. Configure secrets securely.
9. Configure service health checks.
10. Configure persistence and restart behavior.
11. Test recovery.
12. Measure resource consumption.
13. Verify that the architecture remains within free allowances.

### Definition of Done — Phase 7

**Required state**

- Services are reachable only through intended access paths.
- Persistent data survives restart.
- The deployment is documented.
- The origin is not accidentally exposed.

**Required tests**

- Health checks.
- Authentication.
- Network exposure review.
- Restart recovery.
- Data persistence.
- Resource measurement.
- Free-tier usage review.

**Required artifacts**

- Deployment runbook.
- Recovery runbook.
- Security checklist.
- Updated cost register.

**Block progression if**

- Public exposure is unplanned.
- A free-tier limit is unknown.
- The service can silently incur costs.
- The deployment cannot be recovered from the documented instructions.

---

## Phase 8 — End-to-end workflow validation

### Goal

Prove that the final architecture solves the real problem rather than merely passing component tests.

### Steps

1. Choose one representative Feature Factory use case.
2. Create a task.
3. Execute multiple stages.
4. Persist intermediate results.
5. Force a controlled failure.
6. Confirm that the completed work remains recorded.
7. Resume from the appropriate checkpoint.
8. Confirm that completed stages are not unnecessarily repeated.
9. Validate the final artifact.
10. Inspect runtime and workflow logs.
11. Repeat with quota exhaustion if the test can be performed safely within free quotas.
12. Measure the total resource and storage impact.

### Definition of Done — Phase 8

**Required state**

- At least one approved end-to-end use case works.
- Recovery behavior is proven if required.
- Runtime and application-level responsibilities are separated correctly.

**Required tests**

- End-to-end happy path.
- Controlled failure.
- Resume.
- Final output validation.
- Security and cost review.

**Required artifacts**

- End-to-end test report.
- Updated architecture documentation.
- Operational runbook.

**Required decisions**

- Whether the MVP is ready for regular use.
- Which remaining features are worth implementing.

**Block progression if**

- The workflow cannot recover from an expected failure.
- The final result cannot be traced to its task/run.
- Cost or security requirements are violated.

---

## Phase 9 — Stabilization and ongoing operation

### Goal

Make the system maintainable without unnecessary infrastructure or operational overhead.

### Steps

1. Document regular health checks.
2. Define safe backup and restore procedures.
3. Document quota exhaustion behavior.
4. Document provider recovery and retry behavior.
5. Define data-retention policies.
6. Review logs for secret leakage.
7. Review resource usage.
8. Review free-tier consumption.
9. Record known limitations.
10. Produce a final handoff.

### Definition of Done — Phase 9

**Required state**

- I can operate and recover the system using the documentation.
- The next maintenance action is clear.
- The cost constraint remains satisfied.

**Required tests**

- Restore/recovery procedure.
- Service restart.
- Configuration validation.
- Security review.
- Cost review.

**Required artifacts**

- Final runbook.
- Final architecture.
- Test evidence.
- Known limitations.
- Final agent handoff.

**Block completion if**

- Recovery is undocumented.
- Secrets are exposed.
- Cost behavior is unknown.
- Critical limitations are hidden.

---

# 13. STRICT DEFINITION OF DONE TEMPLATE

Every phase must use the following structure, even if the phase is short.

## A. Required state

List the concrete resources, code, configuration, documentation, or decisions that must exist.

## B. Required tests

List each test, its expected result, actual result, and evidence location.

## C. Required artifacts

List the exact files, reports, migration scripts, configuration changes, or other outputs.

## D. Required decisions

List the architectural or product decisions that must be settled and who approves them.

## E. Failure conditions

List conditions that block completion.

## F. Verification status

Each criterion must be marked:

- PASS — evidence confirms the requirement.
- FAIL — evidence shows it is not satisfied.
- BLOCKED — the test cannot be completed yet.
- NOT APPLICABLE — with a reason.
- UNKNOWN — not yet investigated.

UNKNOWN is not PASS.

## G. Exit decision

The agent must explicitly state whether the next phase is allowed to begin.

A phase cannot be declared complete because the installation succeeded, the code compiles, or the agent feels confident.

---

# 14. OMNIROUTE TEST EVIDENCE REGISTER

Maintain a structured test table.

Required columns:

- Test ID.
- Requirement.
- Setup.
- Action.
- Expected result.
- Actual result.
- Status.
- Evidence location.
- Version tested.
- Notes.

Use stable test IDs such as:

- OR-ARM-001.
- OR-INF-001.
- OR-ACC-001.
- OR-FB-001.
- OR-QUOTA-001.
- OR-CTX-001.
- OR-CONC-001.
- OR-STREAM-001.
- OR-RESTART-001.
- OR-API-001.

Record the historical tests without falsely claiming that they were rerun.

The critical unresolved test is OR-CTX-001: Context Relay account transition and context preservation.

Do not turn a conditional result into an unconditional pass.

---

# 15. SECURITY AND SECRET-HANDLING REQUIREMENTS

Security is mandatory, including during debugging.

Never ask me to paste an API key, access token, cookie, session token, bootstrap token, or full environment file into the conversation unless there is an unavoidable, explicitly justified need.

Prefer commands that consume credentials from existing protected files without printing them.

Do not display the contents of:

- `/home/opc/litellm/.env`
- `/home/opc/litellm/.env.backup`
- `/app/data/server.env`
- `/tmp/omniroute-cookie.txt`

The cookie file contains an authentication token and must never be printed.

A previous session accidentally exposed authentication material. Treat previously exposed credentials as potentially compromised. Do not repeat them. Where appropriate, recommend rotation through a safe, approved process.

Do not place credentials in public GitHub files, source code, screenshots, logs, or command output.

For diagnostic reports, use explicit field selection and allowlisting.

Never assume that a public repository is safe for arbitrary command output.

---

# 16. ENGINEERING AND IMPLEMENTATION RULES

## 16.1 Do not implement before understanding

Before implementing a feature:

1. Identify the requirement.
2. Verify whether OmniRoute already provides it.
3. Inspect the relevant official documentation and implementation.
4. Define the smallest change.
5. Define a test.
6. Implement.
7. Verify.
8. Record the result.

## 16.2 Avoid premature complexity

Do not introduce unnecessary:

- Microservices.
- Message brokers.
- Distributed locks.
- Vector databases.
- Multiple memory stores.
- Multiple databases.
- Local model servers.
- Custom provider adapters.
- Duplicated routing engines.
- Complex orchestration frameworks.

Every component must solve a demonstrated problem.

## 16.3 Prefer provider-agnostic integration

The AI Platform should call OmniRoute through a well-defined adapter.

Application code should not need to understand each provider's authentication protocol or account rotation implementation.

## 16.4 Treat free quota exhaustion as a normal operational condition

The system must distinguish between:

- Temporary provider failure.
- Rate limit.
- Exhausted quota.
- Invalid credentials.
- Model unavailable.
- Context too large.
- Invalid request.
- Application logic failure.

A retry is not always appropriate. A failed credential should not necessarily be treated like temporary capacity exhaustion.

Do not endlessly retry requests that cannot succeed.

## 16.5 Measure before optimizing

Measure:

- Memory usage.
- CPU usage.
- Disk use.
- Database size.
- Request latency.
- Concurrent request behavior.
- Failure rate.
- Recovery time.

Do not optimize based on assumptions alone.

---

# 17. REQUIRED INITIAL RESPONSE FROM THE NEW AGENT

Do not begin making changes immediately.

Your first response must:

1. Confirm that you understand the overall objective.
2. Summarize the architecture and the zero-cost/cloud-first constraints.
3. Distinguish completed work from outstanding work.
4. Identify the current Context Relay combo issue.
5. State that the AI Platform is not automatically mandatory.
6. State that you will not design the ERD until the OmniRoute PoC and architecture boundary are settled.
7. Identify the next exact step.
8. Provide only one safe command, if a command is necessary.
9. Ask me for any essential decision that genuinely cannot be inferred from the documented requirements.

Do not start by repeating the entire investigation.

Do not start by asking me to paste files or logs that are already available in the repository.

Do not immediately modify the VM, database, combo, container, or LiteLLM deployment.

---

# 18. THE FIRST EXECUTION TASK

Resume at the current Context Relay investigation.

The immediate objective is to correct the disposable test combo's missing connection pin using a supported OmniRoute mechanism, then complete the Context Relay test.

The current combo has:

- ID: `d5a0e969-06d4-4f45-8140-059e827e13b5`
- Name: `context-relay-gemini-test`
- Strategy: `context-relay`
- First model: `gemini-3.1-flash-lite`, but `connectionId` is null.
- Second model: `gemini-3.1-flash-lite`, pinned to Account 1.

Account 2's correct ID is:

`2c27504c-e8e6-4189-b7f1-3950acb5e88c`

Account 1's correct ID is:

`ae9ef7ed-fd4b-4773-bb14-8b67d889ac79`

The prior attempted GET request using the combo name returned `COMBO_007`. The collection endpoint `/api/combos` works and lists the combo.

The `combos` table stores JSON in its `data` column. Its schema has been inspected read-only.

The previous agent had just discovered these route strings in the application:

```text
/api/combos
/api/combos/[id]
/api/combos/auto
/api/combos/builder/options
/api/combos/duplicate
/api/combos/metrics
/api/combos/reorder
/api/combos/test
/api/context/combos
/api/context/combos/[id]
/api/context/combos/[id]/assignments
/api/context/combos/default
/api/v1/combos
```

The exact update method and validation behavior have not yet been established.

### First step

Inspect only the relevant compiled route or schema for `/api/combos/[id]` and the supported update method. Use targeted source inspection.

Do not perform another recursive grep over the whole application.

Do not modify SQLite directly before checking the supported API or UI mechanism.

### After correcting the combo

Complete a controlled Context Relay test that:

1. Uses a unique session identifier.
2. Pins the first model to Account 2 and the second to Account 1 using the correct connection IDs.
3. Sends enough output tokens for a meaningful answer.
4. Confirms the first request executed against Account 2.
5. Forces or reproduces a controlled transition to Account 1.
6. Checks the `context_handoffs` table.
7. Asks the second request to recall the unique codeword and prior task details.
8. Confirms the second response contains the expected context.
9. Records the result and any limitations.

If a controlled transition cannot be performed safely or within free quotas, stop and report BLOCKED with the reason. Do not spend unnecessary quota repeatedly attempting the same failed test.

---

# 19. PROJECT SUCCESS CRITERIA

The project is successful when all of the following are true:

1. The final architecture is justified by real requirements.
2. OmniRoute provides the runtime functionality it is expected to provide.
3. Any separate AI Platform has a clear purpose and does not duplicate OmniRoute unnecessarily.
4. The platform can execute the approved use case end to end.
5. Required workflow state survives interruptions.
6. Required checkpoints and resume behavior work.
7. Model preference and runtime availability remain separate concepts.
8. The deployment is cloud-first and does not depend on a permanently running local LLM.
9. The system uses free tiers within verified limits and does not silently introduce charges.
10. LiteLLM has been preserved unless a later migration is explicitly approved.
11. Security and secret handling are documented.
12. Every phase has objective, verifiable completion evidence.
13. The project can be resumed by a new agent using the repository and handoff files without rediscovering the entire history.
14. The system's limitations are explicit and accepted rather than hidden.
15. The implementation is no more complicated than necessary to meet the approved requirements.

**Final principle:** Do not build the biggest architecture. Build the smallest reliable architecture that satisfies the real use cases, preserves the zero-cost constraint, and can be operated and recovered without unnecessary complexity.

**Begin by verifying the current checkpoint and completing the Context Relay investigation. Do not skip ahead to ERD or platform implementation.**