# AI Platform + OmniRoute
## Master Architecture, PoC, Implementation & Execution Plan

---

# 0. PURPOSE OF THIS DOCUMENT

This document is the complete execution plan for building a generalized personal **AI Platform** around **OmniRoute**.

The goal is NOT simply to install OmniRoute.

The goal is to build a reusable AI infrastructure that can support multiple future AI applications while avoiding unnecessary local resource consumption and maximizing the use of free cloud services/free tiers.

The implementation must be performed sequentially.

The agent executing this plan must:

1. Understand the architecture before changing anything.
2. Never skip a phase.
3. Never design the AI Platform database/ERD before the OmniRoute PoC is completed.
4. Never duplicate functionality already adequately provided by OmniRoute.
5. Never expose or request secrets unnecessarily.
6. Prefer cloud-hosted services over local services whenever practical.
7. Keep costs as close to **$0/month** as realistically possible.
8. Use free tiers wherever possible.
9. Use the existing LiteLLM deployment only as the current baseline/reference until OmniRoute is proven.
10. Never destroy or modify the existing LiteLLM deployment during the PoC.
11. At the end of every step, explicitly report progress using the phase/step numbering.
12. Never declare a phase complete based only on successful installation. A phase is complete only when its explicit completion criteria have been satisfied.
13. Maintain a written record of important findings, decisions, failures, limitations, and measurements.

Example:

> We have completed Phase 2, Step 5.
>
> Step 6 is the final step of Phase 2.
>
> After Step 6 is completed and verified, we can start Phase 3, Step 1.

The agent must behave like a technical implementation partner, not like an autonomous script that blindly executes commands.

---

# 1. THE BIG PICTURE

The final architecture is intended to look conceptually like this:

```text
                         ┌────────────────────────────┐
                         │       AI PLATFORM          │
                         │                            │
                         │ Applications               │
                         │ Skills                     │
                         │ Tasks                      │
                         │ Workflows                  │
                         │ Runs                       │
                         │ Checkpoints                │
                         │ Context                    │
                         │ Memory                     │
                         │ Knowledge                  │
                         │ Artifacts                  │
                         │ Strategic Model Policy     │
                         └──────────────┬─────────────┘
                                        │
                                        ▼
                              ┌────────────────────┐
                              │     OmniRoute      │
                              │                    │
                              │ Provider catalog   │
                              │ Accounts           │
                              │ API keys/OAuth     │
                              │ Free providers     │
                              │ Quota awareness    │
                              │ Routing            │
                              │ Fallback           │
                              │ Circuit breakers   │
                              │ Cooldowns          │
                              │ Model lockout      │
                              │ Context relay      │
                              │ Compression        │
                              │ MCP                │
                              │ A2A                │
                              │ Runtime memory     │
                              │ Runtime skills     │
                              │ Telemetry          │
                              └─────────┬──────────┘
                                        │
             ┌──────────────────────────┼──────────────────────────┐
             ▼                          ▼                          ▼
        Gemini / Google            DeepSeek                    Groq
             │                          │                          │
             └──────────────────────────┼──────────────────────────┘
                                        │
                              Other free providers
                                        │
                                  300+ providers
```

The exact final infrastructure may change after the PoC.

The architecture above is the **target concept**, not a reason to prematurely implement anything.

---

# 2. WHY ARE WE BUILDING AN AI PLATFORM?

This is the most important architectural question.

## OmniRoute alone is NOT the AI Platform.

OmniRoute is primarily the **AI runtime/gateway layer**.

It solves problems such as:

- Which provider should receive a request?
- Which account should be used?
- Which model is available?
- Is a provider temporarily failing?
- Is a quota exhausted?
- Should another account be used?
- Should the request retry?
- Should a circuit breaker open?
- Can another model/provider be used?
- Can context be compressed?
- How should the request be exposed through an OpenAI-compatible API?
- MCP/A2A/runtime provider integration
- Provider/account management
- Runtime telemetry
- Runtime memory/skills capabilities

These are extremely valuable.

But these are NOT the complete problem we want to solve.

---

# 3. WHAT PROBLEM DOES THE AI PLATFORM SOLVE?

The AI Platform is the layer that understands:

> **What application is running, what task is being performed, what the user is trying to accomplish, what context is required, what previous work has happened, what should happen next, and what final artifact should be produced.**

OmniRoute generally answers:

> "How do I execute this AI request?"

The AI Platform answers:

> "Why am I making this AI request, what should I send, what should happen afterward, and how do I continue the overall job?"

That distinction is fundamental.

---

# 4. EXAMPLE: IMAGE APPLICATION

Suppose we eventually build:

```text
Image Editor AI
```

User uploads:

```text
photo.jpg
```

and asks:

> Remove the background and make it suitable for a professional profile picture.

The application needs to understand:

```text
Application
    ↓
Task
    ↓
Uploaded Artifact
    ↓
User Instruction
    ↓
Required Skill
    ↓
Model Capability
    ↓
AI Request
    ↓
Result
    ↓
Artifact
    ↓
Validation
    ↓
Final Output
```

OmniRoute can help execute the AI request.

But OmniRoute should not be responsible for the application's complete business workflow.

The AI Platform owns that.

---

# 5. EXAMPLE: DOCUMENT APPLICATION

Suppose we build:

```text
Document Intelligence
```

The user uploads:

```text
contract.pdf
```

and asks:

> Analyze this contract and identify financial risks.

The platform needs to know:

- which application is running
- which document was uploaded
- what task is being performed
- what skill is required
- what context should be extracted
- which previous analysis exists
- which model capabilities are required
- what output is expected
- where intermediate results are stored
- whether the task failed
- where it should resume
- what final artifact should be generated

Again:

**OmniRoute is the execution layer.**

**AI Platform is the intelligence/workflow layer.**

---

# 6. EXAMPLE: STOCK-MARKET RESEARCH APPLICATION

Another future application could be:

```text
Market Research AI
```

It may have:

```text
Research Skill
Financial Analysis Skill
Company Analysis Skill
Risk Analysis Skill
Report Generation Skill
```

A user could ask:

> Analyze company X and produce an investment research report.

The platform could execute:

```text
Task
 ↓
Research
 ↓
Gather sources
 ↓
Analyze financial data
 ↓
Run specialized analysis
 ↓
Cross-check
 ↓
Generate report
 ↓
Store report
```

OmniRoute executes individual AI calls.

The AI Platform manages the overall operation.

---

# 7. EXAMPLE: DARK FEATURE FACTORY

The original project was the "Dark Feature Factory."

It remains a valid application.

But it is no longer the foundation of the architecture.

Instead:

```text
AI Platform
     │
     ├── Feature Factory Application
     │
     ├── Document AI Application
     │
     ├── Image AI Application
     │
     ├── Research Application
     │
     └── Future Applications
```

The Feature Factory becomes **one consumer of the platform**.

This is a major architectural improvement because common infrastructure does not need to be rebuilt for every application.

---

# 8. CAN WE SKIP THE AI PLATFORM AND USE OMNIROUTE ONLY?

Technically:

## Yes.

If the objective were simply:

> "I want an OpenAI-compatible gateway that routes requests between free LLM providers."

Then OmniRoute alone could be sufficient.

There would be no need for a custom AI Platform.

But that is NOT the objective.

The objective is to build a reusable AI ecosystem.

Therefore:

```text
OmniRoute alone
=
AI Runtime / Gateway
```

while:

```text
AI Platform + OmniRoute
=
Reusable AI application infrastructure
```

---

# 9. IS THE AI PLATFORM ABSOLUTELY MANDATORY?

No.

It is not technically mandatory.

It becomes necessary because of the intended scope.

If the project remains:

```text
LLM Gateway
```

then OmniRoute is enough.

If the project becomes:

```text
Multiple AI applications
+
skills
+
workflows
+
durable tasks
+
memory
+
context
+
artifacts
+
checkpoint/resume
+
application-level orchestration
```

then a dedicated platform layer becomes justified.

The important rule is:

> Do not build the AI Platform simply because it sounds architecturally sophisticated.

Build only the parts that solve problems that OmniRoute does not solve adequately.

---

# 10. CORE ARCHITECTURAL PRINCIPLE

We must follow this rule throughout the project:

> **Build around OmniRoute, not against OmniRoute.**

Before implementing any feature, ask:

```text
Does OmniRoute already provide this?
        │
        ├── YES
        │    ↓
        │  Use OmniRoute.
        │
        └── NO
             ↓
        Is it required by the AI Platform?
             │
             ├── NO → Don't build it.
             │
             └── YES → Build it.
```

This prevents unnecessary duplication.

---

# 11. WHAT OMNIROUTE SHOULD OWN

The following should remain primarily OmniRoute responsibilities.

## Provider Management

- provider catalog
- provider integrations
- provider authentication
- API keys
- OAuth
- account/connection management

## Runtime Model Availability

- available models
- provider/model mappings
- model availability

## Runtime Routing

- routing
- load balancing
- account rotation
- fallback
- retry
- cooldown
- circuit breakers
- model lockout

## Quota/Provider Runtime Intelligence

- quota awareness
- quota-related failures
- reset hints
- account availability
- headroom where available

## Runtime Resilience

- failures
- backoff
- Retry-After
- circuit breaker
- anti-thundering-herd behavior

## Context Runtime Optimization

- context relay
- token compression where appropriate

## Runtime Protocols

- OpenAI-compatible API
- MCP
- A2A

## Runtime Skills/Memory

Use where they fit.

Do not automatically duplicate them.

---

# 12. WHAT THE AI PLATFORM SHOULD OWN

The AI Platform should own the things that represent the **application's intent and durable business process**.

## Application Registry

Defines:

```text
What applications exist?
```

Examples:

```text
Feature Factory
Document Analyzer
Image Editor
Research Assistant
```

---

# 13. SKILL REGISTRY

Defines reusable capabilities.

Examples:

```text
Code Analysis
Document Summarization
Financial Research
Image Editing
Requirements Analysis
Architecture Review
Report Generation
```

The platform should understand:

```text
Application
   ↓
Skill
   ↓
Task
```

A skill is not merely a prompt.

It may define:

- required capabilities
- preferred model classes
- context requirements
- tools
- input requirements
- output requirements
- validation
- fallback behavior
- execution constraints

---

# 14. TASK ENGINE

A task represents actual work.

Example:

```text
Analyze this PDF
```

or:

```text
Implement Feature X
```

or:

```text
Generate an architecture document
```

A task needs:

- task ID
- application
- skill
- input
- status
- context
- execution state
- output
- errors
- timestamps

---

# 15. WORKFLOW ENGINE

A workflow represents multiple tasks.

Example:

```text
Feature Request
   ↓
Requirements Analysis
   ↓
Architecture
   ↓
Implementation
   ↓
Testing
   ↓
Review
   ↓
Documentation
```

Each step may use a different model.

The workflow should not care which provider executes an individual LLM request.

It asks the model layer for an appropriate execution.

OmniRoute handles the runtime.

---

# 16. DURABLE RUNS

This is one of the most important missing layers.

A run represents a concrete execution of a task/workflow.

Example:

```text
Run #1234
Status: RUNNING
```

It should record enough state to understand:

```text
What happened?
What has completed?
What failed?
What is next?
```

---

# 17. CHECKPOINT / RESUME

This is particularly important for the Feature Factory.

Imagine:

```text
Step 1 complete
Step 2 complete
Step 3 complete
Step 4 currently running
```

Then:

- provider quota is exhausted
- VM restarts
- request fails
- application crashes
- model becomes unavailable

We do NOT want to restart the entire operation.

Instead:

```text
Checkpoint
   ↓
Failure
   ↓
New execution
   ↓
Load checkpoint
   ↓
Continue from Step 4
```

This is a platform responsibility.

---

# 18. CONTEXT MANAGEMENT

Context should be treated as a first-class concept.

The platform may need to combine:

```text
System instructions
+
Application instructions
+
Skill instructions
+
Task input
+
Previous results
+
Relevant memory
+
Relevant knowledge
+
Artifacts
+
Workflow state
```

Then produce an appropriate context package for OmniRoute.

The platform must avoid blindly sending everything to every model.

---

# 19. MEMORY

Memory is not one thing.

We should eventually distinguish:

```text
Conversation Memory
Task Memory
Application Memory
User Memory
Long-Term Knowledge
Execution History
```

OmniRoute may already provide runtime memory.

The AI Platform should only add higher-level semantics where required.

For example:

> "This Feature Factory project has already decided that the database is Supabase."

That is application/project knowledge.

It should not necessarily be represented as a generic gateway memory record.

---

# 20. KNOWLEDGE

Knowledge is information the application can retrieve.

Examples:

```text
Git repositories
Documentation
PDFs
Specifications
Company documents
Research material
Database records
Web research
```

The AI Platform decides:

```text
Which knowledge source is relevant?
When should it be retrieved?
How should it be incorporated?
```

The underlying storage/search technology can be selected later.

---

# 21. ARTIFACT MANAGEMENT

Artifacts are outputs or files generated during execution.

Examples:

```text
PDF
DOCX
Markdown
JSON
Image
Source code
Architecture diagram
Test report
```

The platform should know:

```text
Artifact
 ↓
belongs to
 ↓
Task / Run / Application
```

It should also support versions.

---

# 22. STRATEGIC MODEL POLICY

This is different from runtime routing.

The user explicitly wants global model ranking.

Example:

```text
1. Gemini 3.5 Flash
2. DeepSeek V4
3. Grok 4.7
4. Gemini 2.5
5. DeepSeek V3
```

This is NOT:

```text
Google
  ├── Gemini 3.5
  └── Gemini 2.5

DeepSeek
  ├── DeepSeek V4
  └── DeepSeek V3
```

The platform should be able to express:

```text
Global Strategic Model Ranking
```

independent of provider.

The strategic policy says:

> "For this task, these models are preferred in this order."

OmniRoute then determines whether a candidate can actually execute.

---

# 23. STRATEGIC RANKING VS RUNTIME AVAILABILITY

This distinction is critical.

Suppose:

```text
1. Gemini 3.5 Flash
2. DeepSeek V4
3. Grok 4.7
```

Gemini may currently have:

```text
quota exhausted
```

The platform should NOT permanently change:

```text
1 → 2
```

Instead:

```text
Strategic ranking
1. Gemini
2. DeepSeek
3. Grok

Runtime filtering
Gemini = temporarily unavailable

Effective candidates
1. DeepSeek
2. Grok
```

When Gemini becomes available again:

```text
Gemini returns to #1.
```

This prevents runtime conditions from corrupting strategic preferences.

---

# 24. COST PRINCIPLE

The project has an explicit financial objective:

> **Avoid paid AI infrastructure wherever realistically possible.**

Priority:

```text
Free tier
   ↓
Free managed service
   ↓
Oracle Cloud Always Free
   ↓
Cloudflare free services
   ↓
Other legitimate free tiers
   ↓
Only if necessary: paid service
```

Paid services must not be introduced simply because they are convenient.

If a paid component becomes necessary, the agent must explicitly explain:

1. Why it is needed.
2. What free alternatives were considered.
3. Approximate cost.
4. Whether the component can be removed later.

---

# 25. LOCAL RESOURCE PRINCIPLE

The user explicitly wants to avoid running infrastructure locally.

The laptop should primarily be:

```text
Development Client
+
Browser
+
IDE
+
Terminal
```

It should NOT become the permanent hosting environment.

Avoid unnecessarily running:

```text
PostgreSQL locally
Redis locally
LiteLLM locally
OmniRoute locally
Vector DB locally
LLM locally
Docker stacks locally
```

unless required temporarily for development/testing.

The preferred architecture is:

```text
Laptop
   │
   │ HTTPS
   ▼
Cloud-hosted infrastructure
```

---

# 26. CLOUD-FIRST PRINCIPLE

Infrastructure should preferably run online.

Possible components include:

```text
Oracle Cloud VM
Cloudflare
Supabase
Other free PostgreSQL providers
Other managed databases
Object storage free tiers
Cloudflare services
GitHub
Free AI providers
```

The exact services must be chosen based on the requirements discovered during implementation.

Do NOT assume Supabase must remain.

Do NOT assume the current Oracle VM must remain.

Do NOT assume SQLite must remain.

Do NOT assume PostgreSQL must be used.

The architecture must determine the infrastructure.

---

# 27. CURRENT INFRASTRUCTURE

At the beginning of this project, the current infrastructure is:

```text
Laptop
   │
 HTTPS
   ▼
llm.mtourk.com
   │
Cloudflare Tunnel
   │
   ▼
Oracle Cloud VM
   │
   └── LiteLLM :4000
            │
            ▼
      Supabase PostgreSQL
```

LiteLLM currently works.

The existing Gemini configuration has two deployments under:

```text
gemini-free
```

with:

```text
gemini/gemini-3.5-flash
```

and deployment ordering/failover has already been tested successfully.

This existing deployment must be treated as the **baseline/reference system**.

---

# 28. IMPORTANT: DO NOT DESTROY THE CURRENT LITELLM DEPLOYMENT

The OmniRoute PoC must be performed alongside it.

Initially:

```text
Current VM

LiteLLM
   │
   └── Existing production/test environment

OmniRoute
   │
   └── Separate PoC environment
```

Do not modify the existing LiteLLM deployment unless there is an explicit decision to migrate.

---

# 29. PHASE 1 — RESET THE MENTAL MODEL

## Step 1.1 — Understand the objective

Agent must understand:

```text
We are not building a LiteLLM replacement just for the sake of replacement.

We are evaluating whether OmniRoute is a better runtime foundation
for a larger AI Platform.
```

## Step 1.2 — Understand the AI Platform boundary

Agent must understand:

```text
AI Platform
    =
application intelligence + orchestration + durable state

OmniRoute
    =
AI runtime + provider + routing infrastructure
```

## Step 1.3 — Freeze premature database design

Do NOT create:

```text
ERD
Database schema
Prisma schema
SQL migrations
```

yet.

The OmniRoute capability audit must come first.

### Completion Criteria — Phase 1

Phase 1 is complete only when ALL of the following are true:

- [ ] The agent can explain in its own words why OmniRoute is the runtime/gateway layer.
- [ ] The agent can explain in its own words why the AI Platform is the application/orchestration layer.
- [ ] The agent understands that Feature Factory is an application, not the platform itself.
- [ ] The agent understands that PostgreSQL, SQLite, Supabase, Oracle VM, and Cloudflare are implementation choices, not architectural requirements.
- [ ] The agent understands the `$0/month`-as-close-as-practical cost objective.
- [ ] The agent understands that permanent local infrastructure is intentionally avoided.
- [ ] The agent confirms that no AI Platform ERD/database schema will be designed before the OmniRoute PoC.
- [ ] No production infrastructure has been modified.

### Phase 1 Required Deliverable

Create a short:

```text
Phase 1 Architecture Understanding Record
```

containing:

1. AI Platform responsibility.
2. OmniRoute responsibility.
3. Why both layers exist.
4. What is explicitly out of scope at this stage.

### Phase 1 Gate

**DO NOT start Phase 2 until all Phase 1 completion criteria are checked.**

---

# 30. PHASE 2 — OMNIROUTE DEEP AUDIT

This phase is research and verification.

No major implementation yet.

## Step 2.1 — Inspect OmniRoute architecture

Study:

- repository
- current release
- architecture
- runtime
- configuration
- persistence
- provider system
- routing
- Auto Combo
- quota
- account management
- resilience
- memory
- skills
- MCP
- A2A
- telemetry
- APIs

Use the actual current release being evaluated.

---

# 31. STEP 2.2 — Build a CAPABILITY MATRIX

Create a detailed table:

| Capability | OmniRoute | Required by AI Platform | Build ourselves? |
|---|---|---|---|
| Provider registry | Yes | Yes | No |
| API keys | Yes | Yes | No |
| OAuth | Yes | Maybe | No |
| Account rotation | Yes | Yes | No |
| Quota awareness | Yes | Yes | No |
| Routing | Yes | Yes | No |
| Fallback | Yes | Yes | No |
| Circuit breaker | Yes | Yes | No |
| Compression | Yes | Yes | No |
| MCP | Yes | Yes | No |
| A2A | Yes | Yes | No |
| Runtime memory | Yes | Partial | Only abstraction |
| Runtime skills | Yes | Partial | Domain layer |
| Application registry | No | Yes | Yes |
| Skill registry | Partial | Yes | Yes |
| Task engine | Partial | Yes | Yes |
| Workflow engine | Partial | Yes | Yes |
| Durable runs | Partial | Yes | Yes |
| Checkpoints | Insufficient | Yes | Yes |
| Application context | Partial | Yes | Yes |
| Application knowledge | Partial | Yes | Yes |
| Artifact lifecycle | Partial | Yes | Yes |
| Strategic model ranking | Partial | Yes | Yes |
| Provider runtime health | Yes | Yes | No |

This matrix must be based on actual current OmniRoute capabilities.

---

# 32. STEP 2.3 — IDENTIFY DUPLICATION

For every proposed AI Platform feature ask:

```text
Does OmniRoute already provide it?

If yes:
    Can the AI Platform consume it through a stable API?

If yes:
    Do not duplicate it.

If no:
    Determine whether it belongs in the AI Platform.
```

---

# 33. STEP 2.4 — STUDY OMNIROUTE PERSISTENCE

Determine exactly:

- database type
- tables/files
- state
- configuration
- secrets
- runtime requirements
- backup requirements
- concurrency requirements
- restart behavior
- migration behavior
- multi-instance behavior

Do not decide the database technology yet.

---

# 34. STEP 2.5 — ARM64 COMPATIBILITY ANALYSIS

The target environment may be Oracle Cloud ARM64.

Verify:

```text
OmniRoute
Node.js
native dependencies
SQLite
better-sqlite3
Docker/Podman
system libraries
```

Do not assume ARM64 works merely because the project has ARM-related code.

### Completion Criteria — Phase 2

Phase 2 is complete only when ALL of the following exist:

- [ ] Current OmniRoute version/release identified.
- [ ] Official/current architecture documentation reviewed.
- [ ] Provider system documented.
- [ ] Account/credential system documented.
- [ ] Routing/Auto Combo behavior documented.
- [ ] Quota behavior and limitations documented.
- [ ] Fallback/retry/circuit-breaker behavior documented.
- [ ] Context relay/compression behavior documented.
- [ ] MCP and A2A capabilities documented.
- [ ] Memory and Skills capabilities documented.
- [ ] API interfaces relevant to the AI Platform documented.
- [ ] Persistence mechanism documented.
- [ ] Backup/recovery implications documented.
- [ ] Multi-instance/HA implications documented.
- [ ] ARM64 risks identified.
- [ ] Native dependency risks identified.
- [ ] Capability Matrix completed.
- [ ] Every proposed AI Platform feature classified as:
  - OmniRoute provides it.
  - OmniRoute partially provides it.
  - AI Platform must provide it.
  - Not currently required.
- [ ] No unresolved critical architecture question is hidden inside the audit.

### Phase 2 Required Deliverables

The agent must produce:

```text
1. OmniRoute Architecture Report
2. OmniRoute Capability Matrix
3. Duplication Analysis
4. Persistence Analysis
5. ARM64 Compatibility/Risk Report
6. List of Open Questions
```

### Phase 2 Gate

The agent must explicitly answer:

> "What do we need to build ourselves that OmniRoute does not already adequately provide?"

If the answer is unclear, **Phase 2 is NOT complete.**

---

# 35. PHASE 3 — OMNIROUTE PoC

## CRITICAL REQUIREMENT

Before any AI Platform ERD/database design:

> **Run OmniRoute as a PoC on Oracle ARM64 beside the existing LiteLLM deployment.**

Do not replace LiteLLM yet.

Do not modify LiteLLM.

Do not migrate the current database.

Do not delete anything.

---

# 36. STEP 3.1 — CREATE ISOLATED OMNIROUTE ENVIRONMENT

Prefer:

```text
Separate directory
Separate configuration
Separate port
Separate container/service
Separate hostname if required
```

Example conceptual structure:

```text
Oracle VM

LiteLLM
  :4000

OmniRoute PoC
  :XXXX
```

The actual port must be selected after checking the current VM.

---

# 37. STEP 3.2 — VERIFY BASIC STARTUP

Test:

```text
Application starts
Health endpoint works
API responds
UI works if applicable
Persistence works
Restart works
```

Record:

```text
CPU
RAM
startup time
disk usage
errors
```

---

# 38. STEP 3.3 — TEST 1: GEMINI ACCOUNT ROTATION

Configure multiple Gemini accounts/keys.

Verify:

```text
Account A
 ↓
request
 ↓
Account A exhausted/unavailable
 ↓
Account B
```

The test must prove actual runtime behavior.

---

# 39. STEP 3.4 — TEST 2: MODEL-LEVEL FAILOVER

Example:

```text
Preferred Model
      ↓
failure
      ↓
Second Model
```

Verify:

- timeout
- provider error
- model unavailable
- fallback
- retry

---

# 40. STEP 3.5 — TEST 3: QUOTA EXHAUSTION

Simulate or trigger a quota limitation safely.

Verify:

```text
Quota failure
 ↓
OmniRoute detects failure
 ↓
Current account/model avoided
 ↓
Alternative candidate selected
```

Also document limitations.

Do not assume OmniRoute quota information is perfect.

Record:

```text
source
freshness
confidence
behavior
```

---

# 41. STEP 3.6 — TEST 4: GLOBAL MODEL PRIORITY

Configure conceptual ranking:

```text
1. Gemini 3.5 Flash
2. DeepSeek V4
3. Grok 4.7
4. Gemini 2.5
5. DeepSeek V3
```

Verify whether OmniRoute can represent this directly.

If not:

```text
AI Platform
    ↓
Strategic Model Policy
    ↓
OmniRoute
```

must be used.

---

# 42. STEP 3.7 — TEST 5: CONTEXT RELAY

Send a multi-turn/context-dependent request.

Verify:

```text
Request 1
 ↓
Request 2
 ↓
Model/provider changes
 ↓
Context remains usable
```

Measure token usage.

---

# 43. STEP 3.8 — TEST 6: ARM64 STABILITY

Run repeated requests.

Check:

```text
native dependency errors
segmentation faults
SQLite errors
memory leaks
CPU spikes
random crashes
restart failures
```

This is mandatory because ARM64 compatibility has historically been a concern for SQLite/native Node dependencies in this project.

---

# 44. STEP 3.9 — TEST 7: RESOURCE USAGE

Measure:

```text
idle RAM
idle CPU
normal workload RAM
normal workload CPU
peak RAM
peak CPU
disk usage
```

The objective is to determine whether OmniRoute is practical on the selected Oracle VM.

---

# 45. STEP 3.10 — TEST 8: RESTART / STATE RECOVERY

Perform:

```text
normal restart
unexpected process termination if safely testable
container restart
VM reboot if appropriate
```

Then verify:

```text
provider configuration
accounts
routing configuration
state
memory
skills
application availability
```

---

# 46. STEP 3.11 — TEST 9: CONCURRENT REQUESTS

Run multiple requests simultaneously.

Test:

```text
5
10
20
```

or an appropriate safe load for the VM.

Measure:

```text
latency
failures
memory
CPU
routing behavior
```

Do not overload the free-tier providers.

---

# 47. STEP 3.12 — TEST 10: OPENAI-COMPATIBLE API

Verify that the future AI Platform can communicate with OmniRoute using a stable API abstraction.

Target:

```text
AI Platform
      │
      │ OpenAI-compatible API
      ▼
OmniRoute
```

This is important because it reduces coupling.

---

# 48. STEP 3.13 — PO C SCORECARD

Create a final scorecard:

```text
Provider management
Account rotation
Quota awareness
Routing
Fallback
Resilience
Global model policy compatibility
Context relay
Compression
MCP
A2A
Memory
Skills
API
ARM64 stability
Resource consumption
Persistence
Restart recovery
Concurrent workload
```

Each result must be:

```text
PASS
PASS WITH LIMITATION
FAIL
NOT TESTABLE
```

Do not hide limitations.

### Completion Criteria — Phase 3

Phase 3 is complete only when ALL of the following are true:

### Isolation

- [ ] OmniRoute runs separately from LiteLLM.
- [ ] LiteLLM remained operational during the PoC.
- [ ] No production LiteLLM configuration was modified.
- [ ] No LiteLLM database migration was performed.
- [ ] OmniRoute has its own configuration/state.

### Basic Operation

- [ ] OmniRoute starts successfully.
- [ ] Health check succeeds.
- [ ] API request succeeds.
- [ ] Required provider configuration works.
- [ ] Persistence survives restart.

### Mandatory Tests

- [ ] Test 1 — Gemini account rotation completed.
- [ ] Test 2 — model-level failover completed.
- [ ] Test 3 — quota exhaustion behavior tested.
- [ ] Test 4 — global model priority behavior tested.
- [ ] Test 5 — context relay tested.
- [ ] Test 6 — ARM64 stability tested.
- [ ] Test 7 — resource consumption measured.
- [ ] Test 8 — restart/state recovery tested.
- [ ] Test 9 — concurrent requests tested.
- [ ] Test 10 — OpenAI-compatible API tested.

### Evidence

Every test must have:

```text
Test ID
Date/time
Configuration used
Expected result
Observed result
PASS / FAIL / LIMITATION
Relevant logs/metrics
Conclusion
```

A test without evidence does not count as completed.

### Resource Requirements

The agent must record at minimum:

```text
Idle RAM
Idle CPU
Normal workload RAM
Normal workload CPU
Peak RAM
Peak CPU
Disk usage
```

### Phase 3 Gate

Phase 3 cannot be marked complete if:

- any mandatory test was skipped;
- any test was marked PASS without evidence;
- ARM64 stability is unknown;
- restart recovery is unknown;
- resource usage is unknown;
- OmniRoute interfered with LiteLLM;
- the agent cannot explain how the AI Platform will communicate with OmniRoute.

---

# 49. PHASE 4 — FINAL OMNIROUTE DECISION

Only after the PoC:

## Option A — OmniRoute becomes the runtime

If it passes:

```text
AI Platform
    ↓
OmniRoute
    ↓
Providers
```

LiteLLM can later be retired.

---

## Option B — OmniRoute requires a supporting architecture

If OmniRoute works but some capability needs external infrastructure:

```text
AI Platform
    ↓
OmniRoute
    ↓
Cloud services
```

Examples:

```text
Managed database
Object storage
Vector database
Cloudflare
```

---

## Option C — OmniRoute is not suitable

If critical requirements fail:

```text
Return to LiteLLM
```

or evaluate a hybrid architecture.

The decision must be evidence-based.

### Completion Criteria — Phase 4

Phase 4 is complete only when:

- [ ] The Phase 3 PoC report exists.
- [ ] Every mandatory test has a final status.
- [ ] All critical failures are documented.
- [ ] All important limitations are documented.
- [ ] ARM64 suitability has a clear conclusion.
- [ ] Resource consumption has a clear conclusion.
- [ ] Persistence/recovery has a clear conclusion.
- [ ] Runtime routing capabilities have a clear conclusion.
- [ ] The AI Platform ↔ OmniRoute integration boundary has a clear conclusion.
- [ ] The team has explicitly selected one of:
  - OmniRoute accepted.
  - OmniRoute accepted with limitations.
  - OmniRoute rejected.
- [ ] If accepted with limitations, every limitation has an identified mitigation or explicit acceptance.
- [ ] No database/ERD design has been started prematurely.

### Required Deliverable

```text
OmniRoute Final Decision Report
```

containing:

1. Decision.
2. Evidence.
3. Advantages.
4. Limitations.
5. Risks.
6. Required supporting infrastructure.
7. Migration implications.
8. Final recommendation for the next phase.

### Phase 4 Gate

No AI Platform ERD is allowed until this report exists.

---

# 50. DATABASE DECISION

Only after Phase 3 and Phase 4.

Possible options:

```text
OmniRoute embedded SQLite
```

or:

```text
Managed PostgreSQL
```

or:

```text
Cloudflare-related storage
```

or:

```text
Another free-tier managed database
```

or:

```text
New Oracle VM
```

The selection criteria:

```text
Compatibility
Reliability
Free tier
Storage
Backup
Performance
Operational complexity
ARM64 compatibility
Restart recovery
Future scalability
```

### Completion Criteria — Database Decision

This decision is complete only when:

- [ ] OmniRoute's actual persistence requirements are known.
- [ ] Expected storage requirements are estimated.
- [ ] Expected concurrency is estimated.
- [ ] Backup/recovery requirements are known.
- [ ] At least one free/near-zero-cost option has been evaluated.
- [ ] At least one alternative has been considered where practical.
- [ ] Compatibility has been verified.
- [ ] Estimated monthly cost is documented.
- [ ] Operational complexity is documented.
- [ ] Migration/replacement path is documented.
- [ ] No paid service is selected without explicit justification.

### Required Deliverable

```text
Infrastructure Persistence Decision
```

with:

```text
Chosen option
Why
Alternatives
Cost
Risks
Migration path
Rollback path
```

---

# 51. VM DECISION

The existing VM is NOT sacred.

If a clean environment is better:

```text
Delete/rebuild VM
```

is acceptable.

A new Oracle VM may be created if:

- a clean OS is preferable
- OmniRoute has specific requirements
- current configuration is messy
- resources should be isolated
- deployment automation should be cleaner

The goal is:

> Correct infrastructure, not preservation of existing infrastructure.

### Completion Criteria — VM Decision

- [ ] Required CPU/RAM/disk are known.
- [ ] ARM64 compatibility is known.
- [ ] Current VM suitability is documented.
- [ ] New VM requirements are documented if required.
- [ ] Migration impact is known.
- [ ] Downtime impact is known.
- [ ] Backup requirements are known.
- [ ] Network/Cloudflare requirements are known.
- [ ] Final VM strategy is selected.
- [ ] No destructive action has been performed without explicit approval.

---

# 52. PHASE 5 — DEFINE THE FINAL SYSTEM BOUNDARY

After PoC:

Create a definitive boundary document.

Example:

```text
AI PLATFORM OWNS
----------------
Applications
Skills
Tasks
Workflows
Runs
Checkpoints
Context
Memory abstraction
Knowledge
Artifacts
Strategic model policy

OMNIROUTE OWNS
--------------
Providers
Accounts
Credentials
Runtime models
Quota
Routing
Fallback
Circuit breakers
Cooldowns
Compression
MCP
A2A
Runtime telemetry
Runtime health
```

Anything unclear must be resolved before ERD.

### Completion Criteria — Phase 5

- [ ] Every major capability has exactly one primary owner.
- [ ] Shared capabilities have a clearly defined interface.
- [ ] No critical responsibility is duplicated ambiguously.
- [ ] No required capability has an unknown owner.
- [ ] OmniRoute is accessed through supported external interfaces.
- [ ] AI Platform does not depend directly on OmniRoute internal database tables.
- [ ] Strategic model policy ownership is explicitly assigned to AI Platform.
- [ ] Runtime provider/model availability ownership is explicitly assigned to OmniRoute.
- [ ] Memory ownership is explicitly separated into runtime vs application semantics.
- [ ] Artifact ownership is explicitly assigned.
- [ ] Workflow/checkpoint ownership is explicitly assigned.
- [ ] Cost/quota responsibilities are explicitly assigned.
- [ ] Gateway replacement remains technically possible through an adapter boundary.

### Required Deliverable

```text
Final System Boundary Specification
```

This document becomes the mandatory input for the ERD.

### Phase 5 Gate

If any capability has:

```text
Owner = unclear
```

Phase 5 is NOT complete.

---

# 53. PHASE 6 — AI PLATFORM DOMAIN MODEL

Now — and only now — design the AI Platform domain.

Start with concepts, not SQL.

Identify:

```text
Application
Skill
Task
Workflow
Workflow Step
Run
Checkpoint
Context
Memory
Knowledge Source
Artifact
Model Policy
Execution Request
Execution Result
```

Determine relationships.

### Completion Criteria — Phase 6

- [ ] Every core domain entity has a defined purpose.
- [ ] Every entity has a clear owner.
- [ ] Relationships between entities are documented.
- [ ] Lifecycle of each major entity is documented.
- [ ] Task lifecycle is defined.
- [ ] Workflow lifecycle is defined.
- [ ] Run lifecycle is defined.
- [ ] Checkpoint semantics are defined.
- [ ] Artifact ownership is defined.
- [ ] Model Policy semantics are defined.
- [ ] Context semantics are defined.
- [ ] Memory semantics are defined.
- [ ] Knowledge semantics are defined.
- [ ] No entity exists merely because it "might be useful later."
- [ ] No entity duplicates OmniRoute unnecessarily.

### Required Deliverable

```text
AI Platform Domain Model
```

with entity definitions and relationships.

---

# 54. PHASE 7 — AI PLATFORM DATABASE / ERD

Now create the ERD.

Important:

The database must NOT recreate OmniRoute's internal database.

Instead:

```text
AI Platform DB
       │
       │ references
       ▼
OmniRoute API
```

The AI Platform may store:

```text
omniroute_model_name
omniroute_connection_reference
omniroute_request_id
```

or other stable identifiers where required.

It should not directly depend on OmniRoute's internal SQLite tables.

### Completion Criteria — Phase 7

- [ ] ERD covers every required Phase 6 domain entity.
- [ ] Every relationship has been reviewed.
- [ ] Primary keys are defined.
- [ ] Foreign keys are defined where appropriate.
- [ ] Status/lifecycle fields are defined.
- [ ] Idempotency requirements are addressed.
- [ ] Checkpoint persistence is represented.
- [ ] Artifact relationships are represented.
- [ ] Model policy relationships are represented.
- [ ] OmniRoute references are represented without coupling to internal OmniRoute tables.
- [ ] Audit timestamps are defined.
- [ ] Soft deletion/versioning requirements are addressed where necessary.
- [ ] Indexing strategy is documented.
- [ ] Expected growth is considered.
- [ ] No unnecessary duplicate gateway tables exist.

### Required Deliverables

```text
1. ERD
2. Entity Dictionary
3. Relationship Rules
4. Indexing Strategy
5. Migration Strategy
```

### Phase 7 Gate

The database schema cannot be implemented until the ERD has been reviewed against the Final System Boundary Specification.

---

# 55. PHASE 8 — APPLICATION REGISTRY

Implement:

```text
Applications
```

Examples:

```text
feature-factory
document-ai
image-ai
research-ai
```

Each application can define:

```text
skills
workflows
policies
configuration
```

### Completion Criteria — Phase 8

- [ ] Applications can be created.
- [ ] Applications can be identified uniquely.
- [ ] Application configuration is persisted.
- [ ] Application-to-skill relationship works.
- [ ] Application-to-workflow relationship works.
- [ ] Application isolation is enforced.
- [ ] At least one real application can be registered.
- [ ] Application metadata survives restart.
- [ ] API/service tests pass.

### Phase 8 Gate

At least one real application must successfully execute through the registry before proceeding.

---

# 56. PHASE 9 — SKILL SYSTEM

Implement reusable skills.

Example:

```text
requirements-analysis
code-review
document-analysis
financial-analysis
image-editing
report-generation
```

A skill should be composable.

Applications can reuse skills.

### Completion Criteria — Phase 9

- [ ] Skills can be registered.
- [ ] Skills can be versioned where required.
- [ ] Skills can define required capabilities.
- [ ] Skills can be associated with applications.
- [ ] Skills can be associated with tasks/workflows.
- [ ] Skill configuration is persisted.
- [ ] At least one skill is reused by two different workflows or applications.
- [ ] Skill execution is observable.
- [ ] Skill behavior does not hard-code a specific provider.

### Phase 9 Gate

At least one reusable skill must successfully serve more than one execution context.

---

# 57. PHASE 10 — TASK SYSTEM

Implement:

```text
Task
```

with lifecycle:

```text
CREATED
QUEUED
RUNNING
WAITING
FAILED
PAUSED
COMPLETED
CANCELLED
```

The exact state machine must be designed during implementation.

### Completion Criteria — Phase 10

- [ ] Task creation works.
- [ ] Task execution works.
- [ ] Task status transitions are enforced.
- [ ] Invalid transitions are rejected.
- [ ] Task input is persisted.
- [ ] Task output is persisted or referenced.
- [ ] Task errors are persisted.
- [ ] Task timestamps are persisted.
- [ ] Task can be retried safely.
- [ ] Task can be cancelled safely.
- [ ] Duplicate execution is handled or prevented.
- [ ] Task execution can call OmniRoute through the adapter.

### Phase 10 Gate

A complete task must successfully travel:

```text
CREATED
→ QUEUED
→ RUNNING
→ COMPLETED
```

through the actual platform.

---

# 58. PHASE 11 — WORKFLOW ENGINE

Implement workflow execution.

Example:

```text
Workflow
   │
   ├── Step 1
   ├── Step 2
   ├── Step 3
   ├── Step 4
   └── Step 5
```

Each step should be independently observable.

### Completion Criteria — Phase 11

- [ ] Workflow definitions can be created.
- [ ] Workflow steps can be defined.
- [ ] Step ordering/dependencies work.
- [ ] Workflow execution works.
- [ ] Step status is observable.
- [ ] Step output can feed later steps.
- [ ] Failed steps are represented correctly.
- [ ] Workflow completion is represented correctly.
- [ ] At least one workflow contains 3+ steps.
- [ ] At least one step uses OmniRoute.
- [ ] Workflow state survives service restart.

### Phase 11 Gate

A multi-step workflow must complete successfully end-to-end.

---

# 59. PHASE 12 — DURABLE RUNS + CHECKPOINTS

Implement:

```text
Run
Checkpoint
Execution Event
```

The goal is:

```text
Failure
 ↓
Restart
 ↓
Load state
 ↓
Resume
```

rather than:

```text
Failure
 ↓
Start everything again
```

### Completion Criteria — Phase 12

- [ ] Run state is persisted.
- [ ] Checkpoints are persisted.
- [ ] Execution events are persisted.
- [ ] Workflow progress can be reconstructed.
- [ ] A running workflow can create a checkpoint.
- [ ] Service restart does not lose completed work.
- [ ] Execution resumes from the correct checkpoint.
- [ ] Already-completed steps are not repeated unnecessarily.
- [ ] Failed steps can be retried.
- [ ] Resume behavior is tested after an actual interruption.

### Phase 12 Gate

A workflow must be intentionally interrupted and successfully resumed without restarting completed work.

This is a **mandatory acceptance test**.

---

# 60. PHASE 13 — CONTEXT ENGINE

Implement a context abstraction.

Potential inputs:

```text
Task input
Application instructions
Skill instructions
Previous step outputs
Memory
Knowledge
Artifacts
User information
Execution state
```

Context should be assembled intentionally.

### Completion Criteria — Phase 13

- [ ] Context sources are explicitly defined.
- [ ] Context can be assembled from multiple sources.
- [ ] Context ordering/priority is defined.
- [ ] Previous workflow results can be included.
- [ ] Relevant artifacts can be included.
- [ ] Memory can be included where appropriate.
- [ ] Context can be passed to OmniRoute.
- [ ] Context is not duplicated unnecessarily.
- [ ] Context size/token behavior is observable.
- [ ] At least one multi-step task successfully uses accumulated context.

### Phase 13 Gate

A multi-step task must successfully use information produced by an earlier step.

---

# 61. PHASE 14 — STRATEGIC MODEL POLICY

Implement:

```text
Model Policy
```

Example:

```text
Policy: coding
1. Gemini 3.5 Flash
2. DeepSeek V4
3. Grok 4.7
4. Gemini 2.5
5. DeepSeek V3
```

Another policy might be:

```text
Policy: document-analysis
1. Gemini
2. ...
```

The model policy is application/skill aware.

It is NOT provider-first.

### Completion Criteria — Phase 14

- [ ] Global model ranking can be represented.
- [ ] Ranking is independent of provider grouping.
- [ ] Different skills can have different policies.
- [ ] Runtime availability does not permanently mutate strategic ranking.
- [ ] Temporarily unavailable models are filtered at execution time.
- [ ] A model returning to availability can regain its strategic priority.
- [ ] Policy evaluation is observable.
- [ ] Policy can be changed without rewriting application code.
- [ ] Policy is passed correctly to the execution adapter.

### Phase 14 Gate

Demonstrate:

```text
Strategic rank:
Gemini #1

Gemini unavailable:
DeepSeek selected

Gemini available again:
Gemini selected again
```

without changing the stored strategic ranking.

---

# 62. PHASE 15 — OMNIROUTE EXECUTION ADAPTER

Create a clean internal abstraction:

```text
AI Platform
      │
      ▼
Model Execution Adapter
      │
      ▼
OmniRoute
```

The AI Platform should not spread OmniRoute-specific implementation details throughout its codebase.

Instead:

```text
AIExecutionService
```

or equivalent abstraction.

This makes future gateway replacement possible.

### Completion Criteria — Phase 15

- [ ] AI Platform does not directly call OmniRoute from arbitrary modules.
- [ ] A single defined adapter/interface exists.
- [ ] Adapter supports required request types.
- [ ] Adapter handles errors.
- [ ] Adapter handles timeouts.
- [ ] Adapter returns normalized execution results.
- [ ] OmniRoute-specific details are isolated.
- [ ] Tests can mock the adapter.
- [ ] Replacing OmniRoute would require changes primarily inside the adapter layer.

### Phase 15 Gate

At least one complete application workflow must use only the platform's execution abstraction rather than directly calling OmniRoute.

---

# 63. PHASE 16 — MEMORY ABSTRACTION

Define the platform-level memory contract.

The implementation may use:

```text
OmniRoute memory
```

or:

```text
external database
```

or:

```text
vector store
```

depending on the use case.

The application should not need to know the storage technology.

### Completion Criteria — Phase 16

- [ ] Memory types are defined.
- [ ] Memory ownership is defined.
- [ ] Memory read interface exists.
- [ ] Memory write interface exists.
- [ ] Application code is storage-agnostic.
- [ ] At least one memory-backed workflow works.
- [ ] Memory survives restart where persistence is required.
- [ ] Memory does not accidentally mix unrelated applications/users/tasks.
- [ ] Memory deletion/retention behavior is defined.

### Phase 16 Gate

A test must prove that a later execution can retrieve intentionally stored memory through the platform abstraction.

---

# 64. PHASE 17 — KNOWLEDGE / RAG

Implement only when an actual application requires it.

Do NOT build an enormous RAG platform prematurely.

Start with:

```text
Knowledge Source
 ↓
Retrieve
 ↓
Context
 ↓
AI Execution
```

Then expand if needed.

### Completion Criteria — Phase 17

This phase is complete only when at least one real use case requires knowledge retrieval.

Then:

- [ ] Knowledge source can be registered.
- [ ] Content can be indexed/stored.
- [ ] Relevant content can be retrieved.
- [ ] Retrieved knowledge can enter context.
- [ ] Source attribution/provenance is preserved where required.
- [ ] Retrieval is observable.
- [ ] Storage remains cloud-first.
- [ ] No unnecessary local vector database is introduced.

If no real application currently requires RAG:

> Phase 17 may be explicitly marked **DEFERRED — NOT CURRENTLY REQUIRED**.

It must not be implemented merely for architectural completeness.

---

# 65. PHASE 18 — ARTIFACT MANAGEMENT

Implement cloud-first artifact storage.

Possible providers:

```text
Cloudflare R2
Supabase Storage
Oracle Object Storage
Other free-tier object storage
```

Select based on actual free-tier limits and project requirements.

The laptop should not be the permanent artifact server.

### Completion Criteria — Phase 18

- [ ] Artifact upload works.
- [ ] Artifact metadata is stored.
- [ ] Artifact ownership is defined.
- [ ] Artifact can be associated with a Task/Run/Application.
- [ ] Artifact retrieval works.
- [ ] Artifact versioning works where required.
- [ ] Artifact deletion/retention policy exists.
- [ ] Storage is cloud-hosted.
- [ ] Laptop is not required for permanent storage.
- [ ] Free-tier limits are documented.
- [ ] Monthly cost is documented.

### Phase 18 Gate

At least one complete workflow must:

```text
receive artifact
→ process it
→ generate artifact
→ store artifact
→ retrieve artifact
```

successfully.

---

# 66. PHASE 19 — OBSERVABILITY

The platform must distinguish:

```text
AI Platform telemetry
```

from:

```text
OmniRoute telemetry
```

OmniRoute owns runtime telemetry.

AI Platform owns:

```text
Application
Task
Workflow
Run
Step
Checkpoint
Artifact
```

observability.

### Completion Criteria — Phase 19

- [ ] Application executions are traceable.
- [ ] Task executions are traceable.
- [ ] Workflow steps are traceable.
- [ ] Run IDs can correlate events.
- [ ] OmniRoute request IDs can be correlated where available.
- [ ] Failures are visible.
- [ ] Duration is measurable.
- [ ] Token/usage data can be associated where available.
- [ ] Resource consumption can be observed.
- [ ] Logs do not expose secrets.

### Phase 19 Gate

Given a failed run, the agent must be able to reconstruct:

```text
Application
→ Task
→ Workflow
→ Step
→ Model execution
→ Failure
```

from observability data.

---

# 67. PHASE 20 — SECURITY

Implement:

```text
HTTPS
authentication
authorization
secret management
least privilege
API key protection
audit logging
```

Never store provider secrets unnecessarily in the AI Platform DB if OmniRoute already manages them.

### Completion Criteria — Phase 20

- [ ] HTTPS is enforced for public access.
- [ ] Authentication is implemented.
- [ ] Authorization boundaries are defined.
- [ ] Provider secrets are not exposed in logs.
- [ ] Provider secrets are not duplicated unnecessarily.
- [ ] Database credentials are protected.
- [ ] API keys are protected.
- [ ] Least privilege is applied where practical.
- [ ] Audit events exist for security-sensitive actions.
- [ ] Public network exposure is minimized.
- [ ] A basic security review has been performed.

### Phase 20 Gate

No known critical secret exposure or unrestricted public administrative interface may remain.

---

# 68. PHASE 21 — COST CONTROL

Create a cost dashboard/policy.

Track:

```text
Provider
Model
Requests
Tokens
Free-tier usage
Failures
Quota
Estimated cost
```

The primary target is:

```text
$0
```

or as close as practical.

No paid provider should be silently introduced.

### Completion Criteria — Phase 21

- [ ] Every infrastructure component has an identified cost.
- [ ] Every AI provider has a cost/free-tier status.
- [ ] Free-tier limits are documented.
- [ ] Unexpected paid usage can be detected.
- [ ] Monthly cost can be estimated.
- [ ] Paid services require explicit approval.
- [ ] There is a documented fallback when a free service reaches its limit.
- [ ] Local hosting is not being used merely to avoid a free cloud service unless there is a clear reason.

### Phase 21 Gate

The agent must be able to answer:

> "What will this system cost per month under normal expected usage?"

with an evidence-based estimate.

---

# 69. PHASE 22 — FEATURE FACTORY APPLICATION

Only after the platform foundation exists:

Build Feature Factory as:

```text
AI Platform Application
```

It can use:

```text
Skills
Tasks
Workflows
Context
Memory
Knowledge
Artifacts
Model Policies
OmniRoute
```

This prevents Feature Factory from becoming a monolithic system.

### Completion Criteria — Phase 22

- [ ] Feature Factory is registered as an AI Platform application.
- [ ] It uses platform Tasks.
- [ ] It uses platform Workflows.
- [ ] It uses platform Skills.
- [ ] It uses platform Model Policies.
- [ ] It uses the OmniRoute adapter rather than direct gateway coupling.
- [ ] It produces durable Runs.
- [ ] It creates checkpoints.
- [ ] It can resume interrupted work.
- [ ] It produces persistent artifacts.
- [ ] Its execution can be observed end-to-end.

### Phase 22 Gate

A representative Feature Factory workflow must complete successfully and survive an intentional interruption/resume test.

---

# 70. PHASE 23 — DOCUMENT AI APPLICATION

Create a second application to validate that the platform is actually reusable.

Example:

```text
Upload PDF
 ↓
Extract
 ↓
Analyze
 ↓
Summarize
 ↓
Generate report
```

This is an architectural validation exercise.

If adding a second application requires rewriting the core platform, the abstraction is wrong.

### Completion Criteria — Phase 23

- [ ] Document AI is registered independently from Feature Factory.
- [ ] It reuses existing platform capabilities.
- [ ] It can accept a document artifact.
- [ ] It can execute a multi-step workflow.
- [ ] It uses skills.
- [ ] It uses model policy.
- [ ] It stores output artifacts.
- [ ] It can retrieve previous task/run state.
- [ ] It does not require application-specific duplication of core platform infrastructure.

### Phase 23 Gate

A complete document analysis workflow must run without modifying the platform core architecture specifically for Document AI.

---

# 71. PHASE 24 — IMAGE AI APPLICATION

Potentially:

```text
Upload image
 ↓
Understand request
 ↓
Select capability/model
 ↓
Edit/generate
 ↓
Store artifact
```

Again:

```text
Application
 ↓
Platform
 ↓
OmniRoute/runtime
```

### Completion Criteria — Phase 24

Only implement if a suitable image-capable runtime/provider is available.

If implemented:

- [ ] Image application is independently registered.
- [ ] Image artifact ingestion works.
- [ ] Required model capability is detected.
- [ ] Model policy can select an appropriate runtime.
- [ ] Image processing executes through the platform.
- [ ] Result is stored as an artifact.
- [ ] The workflow is observable.

If no suitable free/near-zero-cost image capability is available:

> Mark Phase 24 **DEFERRED — PROVIDER/CAPABILITY NOT CURRENTLY AVAILABLE**.

Do not force implementation.

---

# 72. PHASE 25 — PLATFORM HARDENING

After multiple applications work:

Test:

```text
failure recovery
quota exhaustion
provider failure
VM restart
database restart
network failure
partial workflow completion
duplicate execution
concurrent runs
large context
large files
artifact recovery
```

### Completion Criteria — Phase 25

- [ ] Provider failure has been tested.
- [ ] Quota exhaustion has been tested.
- [ ] Network interruption has been tested.
- [ ] Application restart has been tested.
- [ ] Database/storage restart has been tested where applicable.
- [ ] Partial workflow recovery has been tested.
- [ ] Duplicate execution behavior has been tested.
- [ ] Concurrent workflow execution has been tested.
- [ ] Large context behavior has been tested.
- [ ] Large artifact behavior has been tested.
- [ ] Artifact recovery has been tested.
- [ ] No known critical data-loss scenario remains unaddressed.

### Phase 25 Gate

All critical recovery scenarios must either:

```text
PASS
```

or have a documented, consciously accepted limitation.

---

# 73. PHASE 26 — FINAL INFRASTRUCTURE OPTIMIZATION

Only now optimize:

```text
VM size
database
storage
Cloudflare
backup
monitoring
domains
containers
resource limits
```

Do not optimize infrastructure prematurely.

### Completion Criteria — Phase 26

- [ ] CPU allocation is appropriate.
- [ ] RAM allocation is appropriate.
- [ ] Disk allocation is appropriate.
- [ ] Database/storage is appropriate.
- [ ] Cloudflare configuration is optimized.
- [ ] Backups exist where required.
- [ ] Monitoring is appropriate.
- [ ] Unused services are removed.
- [ ] Unused containers are removed.
- [ ] Unused ports are closed.
- [ ] Monthly cost is re-evaluated.
- [ ] Local machine is not carrying unnecessary permanent infrastructure.
- [ ] Final architecture documentation matches actual deployed infrastructure.

### Final Phase Gate

The deployed system and the architecture documentation must describe the same system.

No undocumented critical infrastructure may remain.

---

# 74. FINAL TARGET ARCHITECTURE

The target should eventually resemble:

```text
                       USER
                         │
                         ▼
                 Cloudflare / HTTPS
                         │
                         ▼
                 AI PLATFORM API
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
    Applications      Skills        Workflows
          │              │              │
          └──────────────┼──────────────┘
                         │
                         ▼
                  Task / Run Engine
                         │
              ┌──────────┼──────────┐
              │          │          │
              ▼          ▼          ▼
           Context     Memory    Knowledge
              │          │          │
              └──────────┼──────────┘
                         │
                         ▼
                 Model Policy
                         │
                         ▼
              Execution Adapter
                         │
                         ▼
                    OmniRoute
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
    Gemini           DeepSeek           Groq
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
                  Other providers
```

---

# 75. IMPORTANT ARCHITECTURAL RULES

## Rule 1

Do not duplicate OmniRoute functionality without a documented reason.

## Rule 2

Do not directly access OmniRoute internal database tables from the AI Platform.

## Rule 3

Use OmniRoute's external APIs/interfaces.

## Rule 4

Do not create the AI Platform ERD before completing the OmniRoute PoC.

## Rule 5

Do not assume PostgreSQL.

## Rule 6

Do not assume SQLite.

## Rule 7

Do not assume Supabase.

## Rule 8

Do not assume the existing Oracle VM must remain.

## Rule 9

Do not run permanent local infrastructure on the laptop unless necessary.

## Rule 10

Prefer free cloud services.

## Rule 11

Never silently introduce paid infrastructure.

## Rule 12

Strategic model ranking belongs to the AI Platform.

## Rule 13

Runtime provider/model availability belongs to OmniRoute.

## Rule 14

The AI Platform must be gateway-agnostic at the architectural boundary.

## Rule 15

Every major decision must be backed by an actual test, source, or explicit requirement.

## Rule 16

A phase is not complete because the software "appears to work." Its explicit completion criteria must be satisfied.

## Rule 17

A failed test is valuable evidence. Do not hide, bypass, or redefine a failed test merely to allow the project to continue.

## Rule 18

A deferred feature is acceptable when its real-world requirement does not yet exist. Do not implement infrastructure solely to make the checklist look complete.

---

# 76. HOW THE AGENT MUST REPORT PROGRESS

After every completed step, report:

```text
Progress:
Phase X — Step Y completed.

What we completed:
- ...

What we verified:
- ...

Important findings:
- ...

Completion criteria:
- X/Y criteria satisfied.

Next:
Phase X — Step Y+1.

Why:
- ...
```

When reaching the final step of a phase:

```text
Phase X — Step Y completed.

This was the final step of Phase X.

Phase X completion criteria:
- [x] ...
- [x] ...
- [x] ...

Phase X is now COMPLETE.

We can now start:
Phase X+1 — Step 1.

We will NOT proceed until the completion conditions of the current phase are satisfied.
```

If a criterion fails:

```text
Phase X — Step Y completed.

Phase completion status:
BLOCKED.

Failed criterion:
- ...

Reason:
- ...

Required action:
- ...

We will NOT start Phase X+1 until this is resolved.
```

---

# 77. STOP CONDITIONS

The agent must stop and ask for a decision when:

```text
Architecture has two materially different valid options.
```

or:

```text
A paid service may be required.
```

or:

```text
A destructive migration is proposed.
```

or:

```text
Current evidence is insufficient.
```

or:

```text
OmniRoute behavior contradicts expectations.
```

or:

```text
A security-sensitive decision is required.
```

or:

```text
A mandatory phase completion criterion cannot be satisfied.
```

Do not guess.

---

# 78. FIRST ACTION WHEN STARTING WITH A NEW AGENT

The agent must NOT immediately install anything.

The first action is:

## Phase 1 — Step 1

Read and understand this document.

Then inspect the current environment.

The agent should establish:

```text
Current VM
Current OS
Architecture
CPU
RAM
Disk
Podman/Docker
Current LiteLLM
Current Cloudflare configuration
Current domain
Current network exposure
```

Secrets must not be printed.

---

# 79. SECOND ACTION

Perform the OmniRoute deep capability audit.

Do not design the AI Platform database yet.

The agent must produce:

```text
OmniRoute Capability Matrix
+
Architecture Findings
+
Limitations
+
ARM64 Risks
+
Persistence Options
+
Free-tier Requirements
```

---

# 80. THIRD ACTION

Prepare the isolated OmniRoute PoC.

The existing LiteLLM deployment remains untouched.

Target:

```text
LiteLLM
    +
OmniRoute PoC
```

running simultaneously.

---

# 81. FOURTH ACTION

Execute the ten mandatory tests:

```text
1. Gemini account rotation
2. Model-level failover
3. Quota exhaustion
4. Global model priority
5. Context relay
6. ARM64 stability
7. RAM/CPU usage
8. Restart/state recovery
9. Concurrent requests
10. OpenAI-compatible API
```

---

# 82. FIFTH ACTION

Produce a final PoC report.

Only then decide:

```text
OmniRoute = accepted
```

or:

```text
OmniRoute = accepted with limitations
```

or:

```text
OmniRoute = rejected
```

---

# 83. SIXTH ACTION

If accepted:

Determine the final infrastructure:

```text
Existing Oracle VM?
New Oracle VM?
SQLite?
PostgreSQL?
Cloudflare?
Supabase?
Other free managed service?
```

based on evidence.

---

# 84. SEVENTH ACTION

Freeze the boundary:

```text
What OmniRoute owns
+
What AI Platform owns
```

Then, and only then:

```text
Design AI Platform domain
Design ERD
Design database
```

---

# 85. FINAL OBJECTIVE

The final result is NOT:

> "We installed OmniRoute."

The final result is:

> **A cloud-hosted, near-zero-cost, reusable AI Platform that can support multiple AI applications and use OmniRoute as its AI runtime/gateway, while maintaining durable application workflows, context, memory, knowledge, artifacts, checkpoint/resume, and strategic model-selection policies.**

The laptop remains primarily a development client.

Infrastructure runs online.

Free tiers are preferred.

Local LLM hosting is not a requirement.

Paid services are avoided whenever practical.

The platform is designed so that:

```text
Today:
Feature Factory

Tomorrow:
Document AI

Later:
Image AI

Later:
Research AI

Later:
Other AI applications
```

without rebuilding the underlying AI infrastructure every time.

---

# 86. THE EXECUTION ORDER — MASTER CHECKLIST

```text
PHASE 1
□ Understand objective
□ Understand AI Platform vs OmniRoute
□ Freeze ERD/database design
□ Satisfy Phase 1 completion criteria
□ Produce Architecture Understanding Record

PHASE 2
□ Deep OmniRoute audit
□ Capability matrix
□ Duplication analysis
□ Persistence analysis
□ ARM64 analysis
□ Satisfy Phase 2 completion criteria
□ Produce OmniRoute Architecture/Audit package

PHASE 3 — MANDATORY PoC
□ Isolated OmniRoute deployment
□ Basic startup
□ Gemini account rotation
□ Model failover
□ Quota exhaustion
□ Global model priority
□ Context relay
□ ARM64 stability
□ Resource usage
□ Restart/state recovery
□ Concurrent requests
□ OpenAI API
□ Collect evidence for every test
□ Satisfy Phase 3 completion criteria

PHASE 4
□ PoC scorecard
□ Final OmniRoute decision
□ Satisfy Phase 4 completion criteria
□ Produce Final Decision Report

PHASE 5
□ Database decision
□ VM decision
□ Cloud services decision
□ Final architecture boundary
□ Satisfy Phase 5 completion criteria

PHASE 6
□ AI Platform domain model
□ Satisfy Phase 6 completion criteria

PHASE 7
□ AI Platform ERD
□ Database schema
□ Satisfy Phase 7 completion criteria

PHASE 8
□ Application registry
□ Satisfy Phase 8 completion criteria

PHASE 9
□ Skill system
□ Satisfy Phase 9 completion criteria

PHASE 10
□ Task engine
□ Satisfy Phase 10 completion criteria

PHASE 11
□ Workflow engine
□ Satisfy Phase 11 completion criteria

PHASE 12
□ Durable runs
□ Checkpoints
□ Resume
□ Satisfy Phase 12 completion criteria

PHASE 13
□ Context engine
□ Satisfy Phase 13 completion criteria

PHASE 14
□ Strategic model policies
□ Satisfy Phase 14 completion criteria

PHASE 15
□ OmniRoute execution adapter
□ Satisfy Phase 15 completion criteria

PHASE 16
□ Memory abstraction
□ Satisfy Phase 16 completion criteria

PHASE 17
□ Knowledge/RAG if actually required
□ Satisfy Phase 17 completion criteria or explicitly defer

PHASE 18
□ Artifact management
□ Satisfy Phase 18 completion criteria

PHASE 19
□ Observability
□ Satisfy Phase 19 completion criteria

PHASE 20
□ Security
□ Satisfy Phase 20 completion criteria

PHASE 21
□ Cost controls
□ Satisfy Phase 21 completion criteria

PHASE 22
□ Feature Factory
□ Satisfy Phase 22 completion criteria

PHASE 23
□ Document AI
□ Satisfy Phase 23 completion criteria

PHASE 24
□ Image AI if capability/cost allows
□ Satisfy Phase 24 completion criteria or explicitly defer

PHASE 25
□ Hardening
□ Satisfy Phase 25 completion criteria

PHASE 26
□ Infrastructure optimization
□ Satisfy Phase 26 completion criteria
□ Final architecture matches actual deployment
```

---

# 87. THE MOST IMPORTANT PRINCIPLE

Do not think:

```text
"We need to build everything."
```

Think:

```text
"What is the smallest thing we need to build ourselves
after taking maximum advantage of OmniRoute and free cloud services?"
```

The AI Platform should contain **only the capabilities that are necessary to turn OmniRoute from an LLM gateway into the runtime foundation of a reusable AI application ecosystem.**

That is the architectural objective.

---

# 88. FINAL DEFINITION OF PROJECT SUCCESS

The project is considered successful only when ALL of the following are true:

- [ ] OmniRoute has been objectively evaluated rather than assumed.
- [ ] OmniRoute's ARM64 viability is proven.
- [ ] The final runtime architecture is based on PoC evidence.
- [ ] The AI Platform boundary is clearly separated from OmniRoute.
- [ ] No unnecessary OmniRoute functionality has been duplicated.
- [ ] The AI Platform has durable application/task/workflow state.
- [ ] Checkpoint/resume works.
- [ ] Strategic model ranking works independently from provider ranking.
- [ ] OmniRoute is accessed through a clean adapter.
- [ ] At least two different AI applications can use the same platform foundation.
- [ ] The system can operate cloud-first.
- [ ] The laptop does not need to host permanent infrastructure.
- [ ] Free tiers are used wherever practical.
- [ ] Monthly cost is known and controlled.
- [ ] No paid service is silently introduced.
- [ ] Recovery behavior has been tested.
- [ ] Security boundaries have been tested.
- [ ] The actual deployed infrastructure matches the documented architecture.

The final question is not:

> "Does OmniRoute work?"

The final question is:

> **"Did we build a reusable, cloud-hosted, near-zero-cost AI Platform in which OmniRoute performs the runtime/gateway responsibilities while the platform itself manages applications, skills, tasks, workflows, context, durable execution, memory, knowledge, artifacts, and strategic AI behavior?"**

If the answer is yes, the architecture has achieved its original objective.