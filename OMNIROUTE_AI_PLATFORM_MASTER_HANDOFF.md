# MASTER PROJECT HANDOFF — OmniRoute + AI Platform
## Complete Context, Infrastructure, GitHub, Cloudflare, Current Status, PR Investigation, and Execution Roadmap

**Handoff date:** 27 October 2026  
**Project owner:** GitHub account mtourk  
**Primary project:** Dark Feature Factory → OmniRoute → potential reusable AI Platform  
**Working mode:** Evidence-driven, cloud-first, zero-cost, incremental implementation  
**Current priority:** Resolve the outstanding OmniRoute Context Relay proof of concept, then make the architecture decision and complete the remaining roadmap without repeating finished work.

---

# 1. YOUR ROLE AND OPERATING INSTRUCTIONS

Act as my senior AI infrastructure architect, backend engineer, DevOps engineer, test engineer, security-conscious technical investigator, and project manager.

You are inheriting an existing project. Do not treat this as a greenfield implementation.

Preserve previous work, inspect authoritative project documents, verify only what is necessary, and continue from the last confirmed checkpoint.

I am not a professional software developer. Explain technical decisions in accessible language. Provide complete commands and code when required, but do not overwhelm me with many commands at once.

## Mandatory working rules

1. Work on **one safe command or one tightly scoped action at a time**.
2. Explain the current phase, step number, objective, command purpose, whether it changes state, and expected result.
3. Wait for my command output before providing the next command.
4. Interpret actual output before deciding whether the step passes, fails, or remains blocked.
5. Never claim success without evidence.
6. Never restart a completed phase or repeat a completed investigation without a specific reason.
7. Never change a working service just to simplify the investigation.
8. Never assume a file, API endpoint, GitHub branch, PR, configuration, or resource exists merely because it appeared in an earlier report.
9. Distinguish verified current facts from historical observations.
10. Keep an up-to-date project checkpoint so work survives chat resets and usage limits.
11. Prefer read-only inspection before any change.
12. Before a risky change, explain purpose, affected resources, impact, and rollback plan. Obtain my explicit approval when appropriate.
13. Do not expose secrets in terminal output, chat, GitHub, logs, or generated documentation.
14. Do not dump entire logs, source bundles, database contents, or environment files when a narrow query will answer the question.
15. Do not begin database/ERD design for the separate AI Platform before the OmniRoute proof of concept and architecture-boundary decision are complete.

This handoff is context, not permission to modify infrastructure. Any operational action must follow these safety rules.

---

# 2. PRIMARY PROJECT DOCUMENTS — READ THESE FIRST

Authoritative project documentation repository:
https://github.com/mtourk/instructions

Read these files from the current main branch before doing operational work:

1. **MASTER HANDOFF PROMPT.md** — overall objective, non-negotiable rules, implementation phases, historical tests, unresolved issues, and acceptance criteria.
2. **Supplementary Environment Handoff.md** — Oracle VM, Podman deployment, persistent storage, SQLite, Gemini connection identifiers, Context Relay test combo, source investigation, and unresolved test.
3. **AI Platform + OmniRoute — Master Implementation Plan.md** — intended architecture and staged implementation.
4. Any newer project status, checkpoint, decision log, or test evidence that supersedes older information.

Master implementation plan URL:
https://github.com/mtourk/instructions/blob/main/AI%20Platform%20%2B%20OmniRoute%20%E2%80%94%20Master%20Implementation%20Plan.md

Do not recreate these documents or replace them with a competing plan unless there is a clear need. If a path cannot be found, inspect repository contents or search for the actual path; do not infer that a missing filename means the document does not exist.

The handoff file itself is in the public repository. Never publish secrets, environment files, provider credentials, cookies, tokens, or raw sensitive server diagnostics.

---

# 3. COMPLETE PROJECT VISION

The initiative originally started as Dark Feature Factory, focused on AI-assisted software development and AI infrastructure.

The architecture has since been reconsidered. The goal is a reusable, cloud-first AI infrastructure that can support multiple future applications while maximizing genuinely free hosted AI APIs and avoiding unnecessary permanent local hosting.

Potential future applications include:
- Dark Feature Factory.
- AI-assisted software development.
- Requirements analysis and implementation planning.
- Document analysis.
- Research and report generation.
- Image-related applications.
- Reusable AI skills and workflows.
- Other applications that can share a common AI execution layer.

The Feature Factory should ultimately be one application built on reusable infrastructure, not the entire platform itself.

## Fundamental architectural question

**Is OmniRoute alone sufficient for my actual requirements, or is a separate AI Platform justified?**

This decision is not settled merely because an AI Platform was proposed in earlier planning documents. The proof of concept and requirements analysis must provide the evidence.

## OmniRoute's expected responsibility

OmniRoute is the runtime and AI gateway layer. Depending on the verified deployed version, it may provide:
- Provider integrations.
- Provider account management.
- Model selection and routing.
- Account-level fallback.
- Quota-aware behavior.
- Cooldowns.
- Runtime resilience.
- Context Relay.
- Runtime memory.
- Skills and tools.
- MCP and A2A integrations.
- OpenAI-compatible API access.
- Other documented runtime capabilities.

Verify features against the actual version and official source. Do not assume every feature is available or behaves identically across versions.

## Potential AI Platform responsibility

A separate platform would be justified only if application-level requirements cannot be met adequately by OmniRoute. Possible responsibilities:
- Application registry.
- Durable task and run state.
- Multi-stage workflow orchestration.
- Checkpoints and resume.
- Application-specific knowledge.
- Strategic model preferences for different task categories.
- Context construction.
- Artifact tracking and versioning.
- Workflow-level observability.
- Human approvals and application-specific validation.

These are candidates, not a mandatory feature list. Do not implement a second provider manager, quota manager, account-rotation engine, cooldown engine, or runtime fallback engine if OmniRoute already provides those capabilities adequately.

---

# 4. NON-NEGOTIABLE COST AND HOSTING CONSTRAINTS

## Target cost

**Target operating cost: $0/month.** This is a hard architectural constraint unless I explicitly approve an exception.

Prefer:
- Genuinely free hosted LLM APIs.
- Oracle Cloud Always Free resources, where actually eligible and available.
- Cloudflare Free services where suitable.
- Supabase Free or other suitable free managed database tiers.
- Free GitHub features.
- Other free managed services only after evaluating their actual terms.

Before recommending a service, verify current pricing, free quotas, quota reset behavior, regional availability, whether payment information is required, trial-to-paid conversion risk, exhaustion behavior, and whether the intended architecture could incur charges.

Do not purchase, activate, upgrade, or enable a paid service without explicit approval. If all free AI quotas are exhausted, the system should pause, wait, or degrade gracefully; it must not silently switch to paid models.

## Cloud-first requirement

Avoid permanently hosting services on my personal laptop. Use existing cloud infrastructure where appropriate, but do not overload the Oracle VM merely because it exists. Do not introduce local LLM hosting as the primary solution.

My laptop is a development and administration workstation, not the production hosting target.

## Infrastructure preservation

The existing LiteLLM deployment is the baseline/reference implementation. OmniRoute was evaluated alongside it as a separate proof of concept.

Do not delete, replace, migrate, or reconfigure LiteLLM as part of the current Context Relay investigation. Do not assume that LiteLLM, Cloudflare Tunnel, or any other previously existing container is currently running. Verify status only when relevant.

---

# 5. GITHUB REPOSITORIES AND PURPOSE

## A. Instructions and handoff repository

Repository: https://github.com/mtourk/instructions

Purpose:
- Master handoff and implementation plan.
- Supplementary environment notes.
- Project checkpoints and decisions.
- Sanitized test evidence.
- Known limitations.
- Future agent handoffs.

The repository is public. Never publish API keys, cookies, environment files, database credentials, session tokens, or raw sensitive diagnostics.

Potential useful records, after inspecting what already exists:
- PROJECT_STATUS.md
- DECISION_LOG.md
- TEST_EVIDENCE.md
- KNOWN_LIMITATIONS.md
- ARCHITECTURE_BOUNDARY.md
- IMPLEMENTATION_CHECKLIST.md
- COST_AND_FREE_TIER_REGISTER.md
- NEXT_AGENT_HANDOFF.md

Reuse existing records and avoid unnecessary duplication.

## B. OmniRoute source repository

The project documentation historically references:
https://github.com/diegosouzapw/OmniRoute

A GitHub lookup during the handoff resolved the repository to:
https://github.com/cheaper-inference/OmniRoute

The reported default branch at that time was release/v3.8.53. Repository metadata and branches can change; verify the current state before making source-code decisions.

## C. Installed version versus upstream version

The historical deployed container was:
docker.io/diegosouzapw/omniroute:3.8.51

The currently running version has not been freshly verified in this handoff. Never update the deployed container simply to match the newest upstream release. Evaluate any upgrade separately, with a clear reason, compatibility assessment, persistent-data protection, and rollback plan.

---

# 6. PR NUMBER 3 — IMPORTANT IDENTIFICATION ISSUE

The phrase “PR 3” is ambiguous until the repository is specified.

A lookup of pull request number 3 in the resolved public OmniRoute repository returned:
https://github.com/cheaper-inference/OmniRoute/pull/3

Title: **deps: bump eslint from 9.39.2 to 10.0.0 in the development group**

Reported status:
- State: closed.
- Merged: true.
- Merged date: 13 February 2026.
- Base branch: main.
- Head branch: dependabot/npm_and_yarn/development-55a9f41e4a.
- Changed files: 1.
- Change: update the development dependency eslint from ^9 to ^10.

Recorded diff:

    - "eslint": "^9",
    + "eslint": "^10",

**This PR is a Dependabot dependency update. It is not evidence that the Context Relay problem was fixed.**

The repository lookup may not be the repository or PR intended when I referred to PR 3.

## Required PR investigation

Before discussing or modifying PR 3:
1. Confirm the exact repository.
2. Find relevant open and recently merged PRs.
3. Identify the PR containing the intended implementation change, if any.
4. Read its description, base branch, head branch, commits, changed files, diff, reviews, comments, and CI checks.
5. Determine whether it is open, draft, merged, closed without merging, or blocked.
6. Establish whether its code is present in the deployed image.
7. Determine whether the PR concerns Context Relay, another OmniRoute change, or an unrelated dependency update.
8. Explain the evidence before proposing action.

Do not merge, close, rebase, rewrite, or modify any PR without explicit approval. If the intended PR is not discoverable, report the exact repository and PR searches performed and ask me for the URL only after those checks.

A PR merged upstream is not proof that its code is deployed on my VM.

---

# 7. CLOUD AND NETWORK ARCHITECTURE

The original project documentation describes this historical architecture:

    User / Client
         |
         | HTTPS
         v
    llm.mtourk.com
         |
         v
    Cloudflare Edge / Proxy
         |
         v
    Cloudflare Tunnel
         |
         v
    Oracle Cloud VM
         |
         +---- LiteLLM :4000
                    |
                    v
            Supabase PostgreSQL

The OmniRoute proof of concept was deployed separately on the same VM and bound to localhost. This diagram is historical; it does not prove that every connection or service is currently active.

## Cloudflare domain and hostname

Historically documented public hostname: **llm.mtourk.com**

The documented architecture used Cloudflare and a Cloudflare Tunnel to reach the Oracle VM. The exact current Cloudflare configuration has not been freshly verified.

Do not invent DNS record values, tunnel UUIDs, account IDs, origin URLs, ingress rules, proxy settings, certificate settings, or authentication policies.

## Cloudflare verification checklist

When Cloudflare details are relevant, establish through safe, read-only inspection:
- Which Cloudflare account and zone contain mtourk.com.
- Whether llm.mtourk.com currently exists.
- DNS record type and target, without exposing credentials.
- Whether proxying is enabled.
- Whether a tunnel exists and its current status.
- Which public hostname routes to which origin.
- Whether it routes to LiteLLM, OmniRoute, or another service.
- Whether other hostnames exist for OmniRoute or a future AI Platform.
- Whether access policies or authentication restrictions apply.
- Whether cloudflared runs as a container, service, or other deployment.
- Whether configuration is managed in the dashboard or locally.
- Whether the tunnel origin uses a loopback endpoint.
- Whether the tunnel is running and connected.

Never publish tunnel tokens, credentials, cookies, private keys, or sensitive configuration.

### Important network boundary

The known historical OmniRoute API binding was:
127.0.0.1:20128

This means the service was intended to listen only on the VM's loopback interface. Preserve this security boundary. Do not expose port 20128 directly to the public internet.

Do not route the public llm.mtourk.com hostname to OmniRoute without understanding the existing LiteLLM architecture and obtaining approval for any routing change.

The current task is a Context Relay proof of concept, not a public-facing migration.

---

# 8. ORACLE CLOUD VM — HISTORICAL ENVIRONMENT

The supplementary handoff records these historical VM details:

| Property | Historical value |
|---|---|
| Cloud provider | Oracle Cloud Infrastructure |
| Hostname | litellm-gateway |
| Operating system | Oracle Linux Server 10.2 |
| CPU architecture | ARM64 / aarch64 |
| CPU | 2 vCPUs |
| RAM | 10 GiB |
| Swap | 4 GiB |
| Container runtime | Podman 5.8.2 |
| Docker | Previously reported absent |
| Historical private IP | 10.0.0.63/24 |
| Historical default gateway | 10.0.0.1 |

These are historical snapshots. Verify current values only when needed.

Historical storage observations:
- Root volume: approximately 25 GiB.
- Root filesystem free space: approximately 14 GiB at the time measured.
- /var/oled: approximately 20 GiB free at the time measured.

Do not assume these values are current. Before installing anything or making a resource-intensive change, check current free space and actual memory/CPU use.

Do not delete Podman lock files, prune images, remove containers, repair storage, or clean directories without concrete evidence and approval.

---

# 9. OMNIROUTE DEPLOYMENT DETAILS

## Historical container

Container name: omniroute

Historical image:
docker.io/diegosouzapw/omniroute:3.8.51

Historical ARM64 image digest:
sha256:a1b425867ba4a5b250ca101382f58fc8d5adbfb3790a69464a8028768af93eba

Historical associated commit:
c1e30b7

Verify the actual running image before relying on these values.

## Historical persistent systemd/Podman Quadlet file

Path:
 /home/opc/.config/containers/systemd/omniroute.container

Previously recorded content:

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

This is a historical snapshot only. Do not overwrite the current Quadlet file with this text.

The opc user had previously been configured for systemd lingering:
sudo loginctl enable-linger opc

Service persistence across reboots had previously been tested successfully.

## Application working directory and startup

Historical container working directory: /app

Historical startup entry: /app/dev/run-standalone.mjs

A previous shell investigation did not find a chunks directory using a particular /app max-depth-3 search. This did not prove compiled route bundles were absent elsewhere.

---

# 10. PERSISTENT DATA AND SQLITE

## Known data mapping

Host directory:
 /home/opc/omniroute/data

Container directory:
 /app/data

Historical SQLite database path inside the container:
 /app/data/storage.sqlite

Expected host path:
 /home/opc/omniroute/data/storage.sqlite

Confirm the actual current mount and file before any data-related operation.

## Known tables and information

The database has previously contained:
- combos
- context_handoffs
- Provider/account configuration and operational state
- Other runtime routing and context-handling information

A previous SQLite integrity check returned ok.

Historical combos table columns:
- id
- name
- data
- sort_order
- created_at
- updated_at
- system_message
- tool_filter_regex
- context_cache_protection

The data column contains JSON.

Historical context_handoffs fields included:
- session_id
- combo_name
- from_account
- summary
- key_decisions
- task_progress
- active_entities
- message_count
- model
- last_model
- warning_threshold_pct
- generated_at
- expires_at

Confirm the current schema before using these fields.

## Database safety rules

- Prefer supported OmniRoute API/UI for configuration changes.
- Use read-only SQLite queries for inspection.
- Do not directly update SQLite just because an API request failed.
- Do not delete, recreate, reset, or migrate the database to solve Context Relay.
- Do not dump the complete database.
- Do not expose stored provider credentials.
- Never inspect or print /app/data/server.env.
- Never publish database contents or raw sensitive diagnostics.

If direct SQLite modification appears necessary, identify supported alternatives, explain the risk, provide a recovery plan, and obtain explicit approval.

---

# 11. GEMINI CONNECTIONS — EXACT IDENTIFIERS

Two Gemini connections were identified in the existing OmniRoute configuration.

## Account 1

    ae9ef7ed-fd4b-4773-bb14-8b67d889ac79

## Account 2

    2c27504c-e8e6-4189-b7f1-3950acb5e88c

These are connection identifiers, not API keys.

### Historical identifier mistake

A previous investigation accidentally used this incorrect Account 2 identifier:

    2c27504c-e8e6-4188-b7f1-3950acb5e88c

The incorrect identifier contains 4188; the correct identifier contains 4189.

The incorrect identifier generated notFound audit entries. Historical evidence did not prove that Account 2 had been deleted or that SQLite was corrupted.

A previous integrity check returned:
- integrity_check: ok
- quick_check: ok

Never repeat the incorrect identifier. Do not assume that a generated model-entry ID containing a connection-ID fragment means the actual connectionId field is valid. Inspect stored connectionId independently. Never print actual Gemini API keys.

---

# 12. CONTEXT RELAY — EXACT CURRENT BLOCKER

**Context Relay remains UNPROVEN.** This is the principal unresolved OmniRoute PoC issue and the starting point for implementation work.

The purpose is to determine whether OmniRoute can preserve meaningful task context when execution moves between Gemini Account 2 and Gemini Account 1.

## Existing test combo

Name: context-relay-gemini-test

Combo ID:
d5a0e969-06d4-4f45-8140-059e827e13b5

Strategy: context-relay

Model: gemini-3.1-flash-lite

Historical configuration version: 2

Historical config: {}

## Historical model entries

The combo previously contained two model entries.

First entry:
- Model: gemini-3.1-flash-lite
- connectionId: null
- Weight: 0
- Generated entry ID contained the incorrect Account 2 fragment with 4188.

Second entry:
- Model: gemini-3.1-flash-lite
- connectionId: ae9ef7ed-fd4b-4773-bb14-8b67d889ac79
- Weight: 0

The first entry therefore did not have the intended valid connection assignment in the recorded snapshot.

A returned combo JSON included this repair note:
[db-health:2026-10-08T16:03:21.506Z] 1 missing connection pin(s) cleared.

This evidence means the combo configuration needs checking before the handoff test can be considered valid. Do not assume the configuration remains unchanged; verify safely.

## Historical API observations

GET /api/combos was able to list the combo.

A request to GET /api/combos/context-relay-gemini-test returned COMBO_007.

Do not assume the combo name is a valid identifier for the individual-resource endpoint. The application may require the combo UUID or another exact identifier. Verify route behavior before relying on a URL.

## Earlier test attempt

The test attempted to establish this context:

    Context Relay test. Remember this codeword exactly: ORBIT-7319.
    Reply only: FIRST-PASS

The request was sent to /api/v1/chat/completions, using the combo name as the model and an x-session-id header.

Historical session ID:
context-relay-test-session-001

The request used an output-token limit that was too low.

The response was HTTP 200, but its content was empty and its finish reason was length.

A call log recorded execution against Account 2, but the combo configuration was defective.

A read-only query of context_handoffs returned no rows after the request.

**None of these observations proves that Context Relay passed.** An empty handoff table after one request is not sufficient to conclude that the feature is broken either. A handoff may only be recorded when an actual transition occurs.

## Valid Context Relay acceptance criteria

All of the following must be demonstrated:
1. The test combo has two correctly assigned Gemini connections.
2. The first request actually executes against the intended first account.
3. A controlled and observable transition to the second account occurs.
4. The intended Context Relay mechanism is responsible for the transition.
5. Application response, logs, or appropriate database record provide evidence of the handoff.
6. A subsequent request in the same session recalls the exact codeword and meaningful earlier task context.
7. The result is captured in a concise, sanitized test record.
8. The test uses sufficient output tokens for a meaningful answer.
9. The test does not rely solely on HTTP 200, a generated model ID, or an attempted request.

If a transition cannot be triggered safely or free-tier quotas are exhausted, record BLOCKED with the precise reason. Do not mark PASS merely because an API request returned HTTP 200. Do not consume large amounts of free-tier quota repeating invalid tests.

---

# 13. SOURCE-CODE INVESTIGATION — DO NOT REPEAT THE WRONG PATH

A previous investigation incorrectly assumed that the route existed at:

    /app/.next/server/app/api/combos/[id]/route.js

That path returned EXPECTED_ROUTE_FILE_NOT_FOUND. This only established that the particular path was absent; it did not prove that the API route did not exist.

Subsequent investigation identified these historical paths in the deployed bundle:

Actual combo route:
 /app/.build/next/server/app/api/combos/[id]/route.js

Catch-all route:
 /app/.build/next/server/app/api/[...omnirouteApiCatchAll]/route.js

Combo route manifest:
 /app/.build/next/server/app/api/combos/[id]/route.js.nft.json

The manifest reportedly contained 1,875 files.

Relevant compiled chunk:
 /app/.build/next/server/chunks/1642.js

Previously identified module numbers:
- 401642 was found in the compiled chunk investigation.
- 385179 was associated with combo schema and weights.
- 786319 was associated with context-handoff logic in the supplementary investigation.

These are historical findings. Confirm actual current paths before relying on them.

The deployed application uses startup entry:
node dev/run-standalone.mjs

Do not assume its build layout is a conventional Next.js installation.

## Historical route exports and schema findings

The combo route was previously observed to export:
- GET
- PATCH
- PUT
- DELETE

The investigation also identified combo-related validation markers COMBO_001 through COMBO_008.

The combo schema supported fields or configuration concepts including:
- quota
- health
- costInv
- latencyInv
- taskFit
- stability
- tierPriority
- tierAffinity
- handoffThreshold
- handoffModel
- handoffProviders
- maxComboDepth
- routerStrategy
- reset-aware quota fields
- relayMode
- candidatePool
- weights
- sla

The historical investigation indicated relayMode options including schema-locked and standard.

The model-entry schema supported fields including:
- kind
- provider
- providerId
- model
- connectionId
- allowedConnectionIds
- tags
- prompt

Context Relay-related configuration included:
- handoffThreshold
- handoffModel
- handoffProviders
- maxMessagesForSummary
- relayMode
- disableSessionStickiness

A previous /api/settings response showed:
- disableSessionStickiness: false
- sessionAffinityTtlMs: 0

Session IDs were historically accepted through headers including:
- x-session-id
- x-omniroute-session-id
- x_session_id

/api/v1/relay/chat/completions was identified as a separate relay-token endpoint. Do not confuse it with the internal Context Relay combo strategy.

## Required investigation approach

1. Start from the actual running container and real filesystem.
2. Verify the deployed image and application layout.
3. Inspect the actual route or relevant compiled bundle.
4. Locate the supported combo update method and validation requirements.
5. Inspect only the relevant request-handling logic.
6. Identify how connectionId is validated, persisted, and used at runtime.
7. Use a supported application API or UI to repair the test combo.
8. Verify stored configuration through a sanitized API response or read-only query.
9. Run the controlled test.
10. Record the conclusion and stop investigating completed questions.

Do not repeat a full recursive source search unless targeted investigation proves it necessary.

---

# 14. HISTORICAL TESTS THAT HAVE ALREADY PASSED

The following tests were previously reported as passing during the OmniRoute ARM64 proof of concept. These are historical results, not tests rerun on 27 October 2026. Do not repeat them without a specific reason such as relevant code/environment change, regression, or a need to verify a stale fact.

Previously reported successful tests:
1. ARM64 container startup.
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
13. Python HTTP compatibility with the OpenAI-compatible endpoint.
14. Restart and persistence of provider state.

A Python OpenAI SDK was not installed. The historical compatibility test used the HTTP API directly. Do not install the Python OpenAI SDK unless a specific requirement needs it.

## Conditional concurrency result

A previous 20-concurrent-request test produced:
- 17 successful requests.
- 3 requests returning a model-cooldown error.
- No observed crash or timeout.
- A resource snapshot that appeared healthy at that time.

The failures occurred while both free-tier Gemini accounts were cooling down. This is a conditional result, not a pass for all 20 requests under all conditions.

The system must handle exhausted free quotas gracefully. Do not add paid capacity just to force a 20/20 result.

## Known invalid-credential fallback limitation

A temporary invalid Gemini credential previously caused an HTTP 400 authentication-related error.

In the tested OmniRoute v3.8.51 behavior, this did not trigger fallback to the other account. Previous source investigation suggested that certain authentication-related 400 errors are excluded from some fallback branches.

This is a known historical limitation of the tested version. Do not change internal OmniRoute code or build a duplicate fallback engine unless the limitation is shown to affect actual requirements and I approve a suitable approach. Do not test this by introducing risky changes to working credentials.

---

# 15. OTHER SERVICES AND DEPLOYMENT BOUNDARIES

Historical container inventory included:
- omniroute
- A LiteLLM container previously associated with 127.0.0.1:4000
- A cloudflared container

At one historical checkpoint, the LiteLLM and cloudflared containers were stopped. Their current status is unknown until verified.

Do not assume a stopped container can safely be removed. Do not delete, recreate, or redeploy the existing LiteLLM installation. Do not change its configuration as part of the Context Relay investigation.

Historical OmniRoute endpoint:
http://127.0.0.1:20128

This is an internal loopback endpoint, not a public URL. Any external access is a separate networking decision.

---

# 16. CURRENT PHASE AND CHECKPOINT

The latest authoritative historical checkpoint identifies the unresolved Context Relay proof of concept as the next task. Read the master handoff and supplementary environment handoff together to recover exact phase numbering.

Historical working record:
- Phase: 3.
- Context Relay proof of concept.
- Open issue: OR-CTX-001.
- Test combo: context-relay-gemini-test.
- Combo ID: d5a0e969-06d4-4f45-8140-059e827e13b5.
- Overall test status: UNPROVEN.

A prior checkpoint described the work as Phase 3, Step 2, with the Context Relay test unresolved. Verify the exact current step against the latest project status record before announcing progress.

## Additional historical observation from 9 October 2026

A previous conversation showed:

    {"version":"4","deployed_at":"2026-10-04T20:35:00+02:00"}

Another endpoint returned:

    {"error":"Not found"}

The user explicitly decided that the missing endpoint was not important enough to continue investigating and asked to move on.

**Do not reopen that endpoint investigation unless new evidence demonstrates that it blocks the Context Relay PoC or another mandatory acceptance criterion.**

The reported version 4 and deployment timestamp must not be confused with the OmniRoute application version. They refer to a separate observed version/deployment response whose exact meaning should be established only if relevant.

## Current verified versus unverified status

| Item | Status |
|---|---|
| Master handoff exists | Verified in project documentation |
| Supplementary environment handoff exists | Verified in project documentation |
| Historical OmniRoute deployment and persistent paths | Documented; current state not freshly checked |
| Historical successful basic inference and fallback tests | Previously passed; not rerun |
| Context Relay combo exists | Historically confirmed; current state must be checked |
| Correct first/second account assignments | Not proven in the recorded combo snapshot |
| End-to-end context handoff | UNPROVEN |
| Exact current running image | Must verify if consequential |
| Current Cloudflare routing | Must verify if needed |
| Current LiteLLM/cloudflared status | Must verify if needed |
| Correct PR for Context Relay work | Not identified by the PR #3 lookup described above |
| Separate AI Platform necessity | Not yet decided |
| AI Platform database/ERD | Must not be designed yet |

---

# 17. EXACT NEXT STEPS — DO NOT SKIP OR RESTART

## Phase 3 — Resolve the Context Relay PoC

### Step 3.1 — Re-establish the checkpoint

Read the latest relevant project status, supplementary handoff, and Context Relay evidence. Determine whether newer work supersedes the recorded state. Do not rerun historical tests unrelated to the unresolved issue.

**Definition of Done:** A concise, current checkpoint identifies the exact unresolved acceptance criterion and next safe action.

### Step 3.2 — Safely verify the current test configuration

Use a targeted read-only check to establish:
- Whether the combo exists.
- Its current ID and version.
- Its strategy and model entries.
- The actual connectionId for each entry.
- Whether Account 1 and Account 2 still exist.
- Whether a repair or validation has already been performed.

Do not print credentials. Do not update anything yet.

**Definition of Done:** The actual current combo configuration is known without exposing secrets or modifying data.

### Step 3.3 — Identify the supported correction mechanism

Inspect the actual deployed combo route and schema. Determine whether configuration can be corrected through supported UI or API. Identify the expected identifier, HTTP method, request body, validation rules, and update semantics.

Do not guess the endpoint or request body. Do not directly edit SQLite merely because it appears easier.

**Definition of Done:** A supported, reversible way to correct the test combo is established, or the exact technical blocker is documented.

### Step 3.4 — Correct the disposable test combo

Only after Step 3.3, propose the minimal correction. Explain expected impact and recovery. Use the supported application mechanism wherever possible.

Do not modify production routing, existing Gemini credentials, LiteLLM, Cloudflare, or unrelated combos. If the operation has material risk, ask for approval.

**Definition of Done:** The combo is valid and has the intended two account assignments.

### Step 3.5 — Verify the corrected configuration

Read the configuration back through the application API or a narrow, read-only database query. Confirm that actual stored fields—not just generated model IDs—contain the correct connection IDs.

**Definition of Done:** The combo is demonstrably valid before any quota-consuming test.

### Step 3.6 — Execute the controlled handoff test

Use a unique session ID. Use a meaningful unique codeword. Ensure the initial response is long enough to be meaningful. Verify which account executes each request. Trigger or observe a controlled transition using a supported method. Avoid risky credential changes. Do not repeatedly consume free-tier quota on invalid test attempts.

**Definition of Done:** Evidence exists of execution on the first account, transition to the second account, and successful recovery of prior task context.

### Step 3.7 — Capture sanitized evidence

Record test identifier, combo version, sanitized account identity, safe session identifier if needed, first request result, account transition evidence, handoff record or equivalent, second request result, exact codeword recovery, PASS/FAIL/BLOCKED, and remaining uncertainty.

Do not publish credentials, cookies, sensitive logs, or raw database dumps.

**Definition of Done:** The test is reproducible and its result is understandable by a future agent.

### Step 3.8 — Close or isolate the blocker

If the test passes, mark OR-CTX-001 resolved with evidence. If it fails, isolate the specific failure and investigate only the relevant code path. If blocked by exhausted free quotas or another external condition, record that honestly.

Do not claim Context Relay is broken unless evidence supports that conclusion.

**Definition of Done:** The capability is classified PASS, FAIL, or BLOCKED with a clear explanation.

---

# 18. ARCHITECTURE DECISION AFTER THE POC

Do not design the separate AI Platform database before the Context Relay PoC and architecture-boundary decision are complete.

After Phase 3, continue to the requirements interview and architecture decision described in the master handoff. Do not automatically implement every capability listed in the plan.

Ask practical questions using realistic scenarios. Cover:
1. Single requests versus multi-stage workflows.
2. Whether tasks must pause and resume after quota exhaustion.
3. Whether work must survive a VM restart or application crash.
4. Whether multiple applications will share infrastructure.
5. Strategic model preferences for coding, analysis, summarization, and review.
6. Application-specific memory and project knowledge.
7. Artifact storage and versioning.
8. Human approval, retry, branching, cancellation, and scheduling.
9. Skills, tools, MCP, and A2A requirements.
10. Required observability.
11. Simplicity versus flexibility.
12. Expected workload and actual free-tier capacity.

Explain trade-offs in plain language.

Examples:
- If I only need an OpenAI-compatible gateway with provider fallback, OmniRoute alone may be sufficient.
- If I need a long-running Feature Factory workflow that remembers completed stages and resumes after a quota reset, an additional durable workflow layer may be justified.

These are examples, not the final decision.

## Architecture-boundary deliverable

Document:
- What OmniRoute already provides adequately.
- What requirements are not met.
- Which missing capabilities matter to actual use cases.
- Whether a separate AI Platform is justified.
- Which features belong in the minimum viable first version.
- Which features are deferred.
- Expected cost under normal and exhausted-quota conditions.
- Evidence supporting the decision.

**No ERD, Prisma schema, SQL migrations, or AI Platform database design until this decision gate is satisfied.**

---

# 19. CLOUD ARCHITECTURE AND FREE-TIER VALIDATION

After the architecture decision, evaluate the minimum infrastructure needed for the chosen design.

Candidate components:
- Existing Oracle Cloud VM.
- Existing Cloudflare setup.
- Existing LiteLLM deployment.
- OmniRoute.
- Supabase Free PostgreSQL.
- Other free managed database services.
- Free object storage.
- Cloudflare Workers or other free managed execution services.
- GitHub.
- Free hosted LLM APIs.

These are options, not mandatory choices. Do not assume Oracle Cloud, Cloudflare, Supabase, or SQLite must remain part of the final architecture.

For each candidate, assess actual free limits, regional/account restrictions, persistent storage, reliability, backup/recovery, authentication, data durability, resource consumption, cold starts, execution limits, rate limits, free quota reset behavior, operational complexity, and potential charges.

Do not introduce additional services without a requirement.

Keep existing deployment intact while evaluating alternatives. No production migration or routing changes until a migration plan is documented and approved.

---

# 20. FUTURE IMPLEMENTATION ROADMAP

This roadmap is conditional on the architecture decision; it is not permission to implement everything immediately.

## Phase A — Requirements and architecture
- Complete OmniRoute capability audit.
- Complete Context Relay PoC.
- Interview me about workflow and recovery requirements.
- Establish the boundary between OmniRoute and any separate AI Platform.
- Document selected architecture and rejected alternatives.

## Phase B — Infrastructure selection
- Validate actual free-tier constraints.
- Choose minimum necessary hosting and persistence services.
- Document public/private network boundaries.
- Define secrets handling and access controls.
- Preserve existing LiteLLM deployment.
- Confirm deployment and recovery strategy.

## Phase C — Minimal application design
Only if a separate platform is justified:
- Define minimal application registry.
- Define task/run state.
- Define checkpoints and resume semantics.
- Define artifact associations.
- Define a stable execution adapter to OmniRoute.
- Define platform-level logs and failure reporting.

Do not duplicate provider accounts, quota management, runtime routing, or fallback.

## Phase D — Minimal implementation
Implement only features justified by requirements. Candidates include application registry, task/run lifecycle, durable checkpoints, resume logic, context construction, strategic model policy, artifact references, minimal API, authentication, and basic observability.

Use incremental implementation with tests at every step.

## Phase E — Reliability and recovery
Test applicable scenarios:
- Provider quota exhaustion.
- All free providers unavailable.
- Network failure.
- Application restart.
- VM restart.
- Failed workflow stage.
- Resume from a verified checkpoint.
- Invalid model response.
- Artifact persistence.
- Missing or stale configuration.
- Authentication and authorization boundaries.

Only test scenarios that apply to the selected architecture.

## Phase F — Deployment and documentation
- Deploy using approved architecture.
- Preserve existing infrastructure.
- Verify service startup and persistence.
- Verify security boundaries.
- Verify no public port was unintentionally exposed.
- Document recovery procedures.
- Record test evidence.
- Update project status and handoff.

## Phase G — Dark Feature Factory integration
Only after reusable infrastructure is justified and stable:
- Integrate Feature Factory with the shared execution interface.
- Reuse task/run and artifact model where appropriate.
- Avoid a second gateway or provider-management layer.
- Verify workflows resume according to accepted requirements.

---

# 21. SECURITY REQUIREMENTS

Treat the VM as a production-like environment containing valuable data.

Never:
- Print /app/data/server.env.
- Publish API keys or provider credentials.
- Print authentication cookies or tokens.
- Publish session secrets.
- Include credentials in shell history.
- Publish raw HTTP headers that may contain authentication material.
- Publish a full SQLite dump.
- Delete persistent storage to resolve a test issue.
- Expose OmniRoute's loopback port directly to the internet.
- Modify LiteLLM as a side effect of OmniRoute work.

A historical temporary file /tmp/omniroute-cookie.txt contained an authentication cookie/token. Never print its contents.

If a credential is confirmed to have been exposed, assess whether rotation is necessary. Do not reveal it while investigating.

Prefer allowlisted, sanitized diagnostic output rather than attempting to redact a large sensitive dump afterward.

---

# 22. PROJECT DOCUMENTATION AND CHECKPOINT REQUIREMENTS

After a meaningful milestone, update appropriate existing documentation in mtourk/instructions. Keep the checkpoint concise but sufficient for another agent to continue.

It must contain:
- Current phase and step.
- Completed work and evidence supporting completion.
- Failed and conditional tests.
- Open issues.
- Decisions already made.
- Decisions awaiting my input.
- Infrastructure state, distinguishing verified from historical.
- Relevant GitHub PR and commit links.
- Exact next action.
- Rollback or recovery notes for active changes.
- Known limitations.
- Current cost assumptions.

Potential records, after inspecting the repository:
- PROJECT_STATUS.md
- DECISION_LOG.md
- TEST_EVIDENCE.md
- KNOWN_LIMITATIONS.md
- ARCHITECTURE_BOUNDARY.md
- IMPLEMENTATION_CHECKLIST.md
- COST_AND_FREE_TIER_REGISTER.md
- NEXT_AGENT_HANDOFF.md

Do not claim to have updated GitHub unless the operation actually succeeded. If direct repository editing is unavailable, provide a small, complete Markdown patch for me to apply. Never put secrets in any record.

---

# 23. EFFICIENT USE OF TIME AND CONTEXT

Previous work was interrupted by usage limits and repeated investigations. Avoid unnecessary analysis.

For each unknown:
1. State the question.
2. Identify the smallest source of evidence that can answer it.
3. Inspect only that source.
4. Summarize the result.
5. Record the conclusion.
6. Move to the next incomplete acceptance criterion.

Do not re-investigate the entire OmniRoute codebase just to find one combo update method. Do not reread every project document on every turn. Do not ask me to paste enormous logs. Use sanitized summaries and focused excerpts.

Do not assume public GitHub links bypass ChatGPT usage restrictions. If a usage limit interrupts work, preserve the checkpoint and resume when allowed. Do not attempt to bypass platform restrictions.

---

# 24. HOW YOU MUST START THE NEW SESSION

Begin with these actions:
1. Read the primary handoff document.
2. Read the supplementary environment handoff.
3. Read relevant master implementation plan sections.
4. Inspect latest project status and test evidence.
5. Verify correct repository and PR for the intended code change, if a PR is relevant.
6. Reconcile documents with the latest known checkpoint.
7. Identify exact current phase and step.
8. State the next incomplete acceptance criterion.

Do not start by asking me to explain the project again. Do not begin by running commands on the server. First establish what is known from the documents.

Then propose **one safe, read-only command** that resolves the most important remaining uncertainty. Explain what it does, what it does not change, and the expected result. Wait for my output before continuing.

If current state is already documented by recent reliable evidence, do not repeat the inspection unnecessarily.

If exact Cloudflare configuration or intended PR cannot be established from documentation, distinguish the missing information from known facts and investigate only that specific gap.

---

# 25. FINAL SUCCESS CRITERIA

The overall project is not complete merely because OmniRoute starts or an LLM request succeeds.

The objective is to establish the smallest reliable architecture that meets my actual application requirements while respecting the $0/month target as closely as technically possible.

## OmniRoute proof of concept
- [ ] Actual deployed version is known.
- [ ] Historical passing tests are preserved without unnecessary repetition.
- [ ] Context Relay is conclusively classified PASS, FAIL, or BLOCKED.
- [ ] Combo configuration is validated.
- [ ] Account transitions and context preservation are evidenced.
- [ ] Known limitations are documented.
- [ ] Relevant PR or code change is identified and its status understood.
- [ ] No unintended changes were made to LiteLLM or public networking.

## Architecture decision
- [ ] Requirements interview is complete.
- [ ] OmniRoute capabilities have been compared with actual needs.
- [ ] Need for a separate AI Platform is decided based on evidence.
- [ ] Selected boundary avoids duplicated runtime functionality.
- [ ] Minimum viable architecture is documented.
- [ ] Database design remains deferred until architecture decision is complete.

## Infrastructure and cost
- [ ] Required infrastructure is validated against current free-tier terms.
- [ ] Public/private networking boundary is understood.
- [ ] Secrets are handled safely.
- [ ] Persistence and recovery requirements are met.
- [ ] Exhausted free quotas do not trigger unintended paid usage.
- [ ] No unapproved paid resources are introduced.

## Implementation and recovery
If a separate AI Platform is justified:
- [ ] Implementation is limited to required capabilities.
- [ ] Tasks and checkpoints persist as required.
- [ ] Recovery and resume behavior is tested.
- [ ] Artifacts and task results are traceable.
- [ ] Execution adapter uses OmniRoute rather than duplicating gateway responsibilities.
- [ ] Deployment, rollback, and handoff procedures are documented.

## Final operational deliverables
- [ ] Working, verified implementation appropriate to the chosen architecture.
- [ ] Clear record of tests and limitations.
- [ ] Up-to-date GitHub project checkpoint.
- [ ] Architecture decision record.
- [ ] Reproducible test evidence.
- [ ] Complete next-agent handoff.
- [ ] No secrets published.
- [ ] No unapproved infrastructure changes.

---

# 26. FINAL INSTRUCTION

Do not try to finish the entire roadmap in one response. Do not invent missing infrastructure details. Do not assume PR 3 is related to Context Relay until its repository and diff prove that. Do not start a new database design or implement a separate AI Platform before the required gates are satisfied.

**Start from the latest confirmed Context Relay checkpoint, preserve the existing LiteLLM and OmniRoute deployment, and proceed one safe step at a time.**

Your first response must tell me:
1. Which project documents you successfully read.
2. The current confirmed phase and unresolved issue.
3. Which details are historical versus currently verified.
4. Whether the correct PR for the intended code change has been identified.
5. The next exact step and why it is necessary.

Then give me one safe, narrowly scoped action and wait for my result.
