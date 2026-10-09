# Supplementary Environment Handoff — OmniRoute VM, Files, Database, Gemini Accounts, and Current Checkpoint

This is a **supplement to the main handoff document**, not a replacement for it. Read both documents and use them together.

The previous agent made an incorrect filesystem assumption when it tried to inspect:

`/app/.next/server/app/api/combos/[id]/route.js`

The command returned `EXPECTED_ROUTE_FILE_NOT_FOUND`. Therefore, that path is **not a verified location** for the route implementation. Do not repeat the same assumption, and do not restart the entire project investigation from scratch.

Your task is to retain the known environment context below, verify only what is necessary, and continue from the exact unresolved Context Relay checkpoint.

## 1. Existing project reference

Master implementation plan:

https://github.com/mtourk/instructions/blob/main/AI%20Platform%20%2B%20OmniRoute%20%E2%80%94%20Master%20Implementation%20Plan.md

The main handoff document contains the broader architecture, implementation phases, historical test results, decisions, limitations, and Definition of Done criteria. Consult it for the full project context.

The project’s constraints remain:

- Target cost: **$0/month**.
- Prefer genuinely free-tier services.
- Cloud-first architecture.
- No unnecessary permanent local services or local LLM hosting.
- Do not rebuild or repeat completed work without evidence that it is necessary.
- Do not begin database/ERD design before the outstanding OmniRoute Context Relay proof of concept is resolved and the architecture boundary is decided.

## 2. VM environment — known historical configuration

The existing VM has previously been identified as follows. These are known historical facts, not a guarantee that every value remains unchanged today.

| Property | Known value |
|---|---|
| Cloud provider | Oracle Cloud Infrastructure (OCI) |
| VM hostname | `litellm-gateway` |
| Operating system | Oracle Linux Server 10.2 |
| Architecture | ARM64 / `aarch64` |
| CPU | 2 vCPUs |
| RAM | 10 GiB |
| Swap | 4 GiB |
| Container runtime | Podman 5.8.2 |
| Docker | Previously reported absent |
| OmniRoute host port | `127.0.0.1:20128` |
| Historical private IP | `10.0.0.63/24` |
| Historical default gateway | `10.0.0.1` |

The IP address, available disk space, software versions, running containers, and current service state must be treated as historical until verified.

Do not assume that this VM is a clean installation. It already contains an existing OmniRoute deployment and historical LiteLLM/cloudflared resources. Preserve existing configuration and data.

### Important networking rule

OmniRoute has historically been published only on the loopback interface:

`127.0.0.1:20128`

Preserve that security boundary. Do not expose port `20128` directly to the internet or change its binding without explicit approval.

## 3. OmniRoute deployment and persistent configuration

### Container

- Container name: `omniroute`
- Previously deployed image: `docker.io/diegosouzapw/omniroute:3.8.51`
- Historical ARM64 image digest: `sha256:a1b425867ba4a5b250ca101382f58fc8d5adbfb3790a69464a8028768af93eba`
- Historical associated commit: `c1e30b7`

These identify the previously investigated deployment. Verify the actual running image before drawing conclusions about current application behavior or source-code locations.

### Persistent systemd/Podman Quadlet configuration

The previously recorded configuration file is:

`/home/opc/.config/containers/systemd/omniroute.container`

Historical configuration:

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

This is a historical configuration snapshot. Do not overwrite the current file with this copy. Inspect the existing file only if necessary, and never print environment variables or secrets while doing so.

The `opc` user was previously configured with systemd lingering enabled:

`sudo loginctl enable-linger opc`

Service persistence across reboots had previously been tested successfully.

## 4. Persistent data and SQLite database

### Known host directory

`/home/opc/omniroute/data`

### Corresponding container directory

`/app/data`

### SQLite database

`/app/data/storage.sqlite`

The host directory is mounted into the container, so the persistent database is expected to reside on the host at:

`/home/opc/omniroute/data/storage.sqlite`

This mapping was previously established from the container configuration. Confirm the actual current mount and database file before modifying anything.

### Known database information

The database has previously contained, among other things:

- A `combos` table.
- A `context_handoffs` table.
- Provider/account configuration and operational state.
- Persistent data associated with OmniRoute’s model routing and context-handling functionality.

A previous read-only integrity check returned `ok`.

The previously inspected `combos` table had these columns:

- `id`
- `name`
- `data`
- `sort_order`
- `created_at`
- `updated_at`
- `system_message`
- `tool_filter_regex`
- `context_cache_protection`

The `data` column stores JSON configuration, including combo models, their connection assignments, strategy, and configuration.

The `context_handoffs` table was previously inspected for records containing fields such as:

- `session_id`
- `combo_name`
- `from_account`
- `summary`
- `key_decisions`
- `task_progress`
- `active_entities`
- `message_count`
- `model`
- `last_model`
- `warning_threshold_pct`
- `generated_at`
- `expires_at`

These fields were identified from historical source-code investigation. Confirm the current schema before relying on them for queries or modifications.

### Database safety requirements

- Prefer the application's supported API or UI for configuration changes.
- Use read-only SQLite queries when inspecting stored configuration.
- Do not modify the SQLite database directly until the official configuration/update mechanism has been investigated and direct database editing has been explicitly approved.
- Do not delete, recreate, reset, or migrate the database merely to solve the current Context Relay issue.
- Do not dump the entire database or publish its contents.
- Never inspect or print `/app/data/server.env`.
- Never expose credentials, tokens, cookies, or sensitive provider configuration in diagnostic output.

## 5. Gemini account identifiers

Two Gemini connections have been identified in the existing OmniRoute configuration.

**Account 1**

`ae9ef7ed-fd4b-4773-bb14-8b67d889ac79`

**Account 2**

`2c27504c-e8e6-4189-b7f1-3950acb5e88c`

These are connection IDs, not API keys. Do not confuse them with provider credentials.

An earlier mistake used an incorrect identifier containing `4188` instead of the correct `4189`. That incorrect ID generated `notFound` audit entries. Historical investigation indicated that this mistake did not delete the account, and SQLite integrity remained `ok`.

Use the exact IDs above when investigating connection assignments. If current state needs confirmation, query only the relevant sanitized account fields; never print the actual Gemini API keys.

## 6. Existing Context Relay test combo

A test combo was created with the following configuration:

| Property | Value |
|---|---|
| Name | `context-relay-gemini-test` |
| Combo ID | `d5a0e969-06d4-4f45-8140-059e827e13b5` |
| Strategy | `context-relay` |
| Model | `gemini-3.1-flash-lite` |

Its previously observed configuration contained two model entries.

### First model entry

- Model: `gemini-3.1-flash-lite`
- `connectionId`: `null`
- Weight: `0`
- Generated model-entry ID incorrectly included the old `4188` account-ID fragment.

### Second model entry

- Model: `gemini-3.1-flash-lite`
- `connectionId`: `ae9ef7ed-fd4b-4773-bb14-8b67d889ac79`
- Weight: `0`

The combo therefore had a defective first connection assignment in the observed snapshot. The second model entry was explicitly pinned to Account 1.

The combo's `config` was observed as `{}`, and its version was `2`.

A repair note in the returned combo JSON reported that a missing connection pin had been cleared. This is additional evidence that the combo configuration needs to be checked and corrected through a supported mechanism before the handoff test can be considered valid.

### API behavior previously observed

A request to:

`GET /api/combos/context-relay-gemini-test`

returned `COMBO_007`.

The collection endpoint:

`GET /api/combos`

was able to list the combo.

Do not assume that the combo name is a valid parameter for the individual-resource endpoint. Verify the route's actual identifier requirements before calling it.

## 7. Application filesystem and source-code locations

**Do not assume the application uses a conventional Next.js output layout.**

The previously attempted path:

`/app/.next/server/app/api/combos/[id]/route.js`

was not found. The command returned:

`EXPECTED_ROUTE_FILE_NOT_FOUND`

This does not prove that the route does not exist. It only proves that the specific expected file was absent at the tested path.

The historical container startup command was:

`node dev/run-standalone.mjs`

The application has previously been investigated through bundled JavaScript chunks, including:

- `chunks/1642.js` — a previously inspected bundle containing combo schema validation, identified as module `385179`.
- `chunks/56268.js` — a previously inspected bundle containing context-handoff logic, identified as module `786319`.

These are historical bundle paths and module identifiers. Verify their current existence before relying on them.

### Relevant API routes previously identified

The following route names were found in earlier source/bundle investigations:

```text
/api/combos
/api/combos/[id]
/api/combos/[id]/route
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

These are route identifiers discovered during earlier investigation, not verified filesystem paths.

### Required approach for finding the actual implementation

1. Start from the existing container's actual filesystem and startup configuration.
2. Identify the application directory and the actual deployed application layout.
3. Use a short, targeted filesystem listing to locate the existing combo route implementation or bundled route registration.
4. Search only the relevant files for combo-update methods, `connectionId` validation, and request-body handling.
5. If the implementation exists only in compiled bundles, inspect the relevant existing bundle rather than assuming `.next/server/app/...`.
6. If the expected implementation genuinely cannot be found, report the exact paths checked and the evidence. Do not silently invent another location.

Do not run another broad recursive search across the whole container unless the targeted investigation demonstrates it is necessary. Avoid giant grep outputs and excessive diagnostic logs.

## 8. Relevant Context Relay implementation findings

Earlier source-code investigation indicated:

- Combo schema validation accepts the `context-relay` strategy.
- Model entries support fields such as `kind`, `provider`, `providerId`, `model`, `connectionId`, `allowedConnectionIds`, `tags`, and `prompt`.
- Combo configuration supports options including `handoffThreshold`, `handoffModel`, `handoffProviders`, `maxMessagesForSummary`, `relayMode`, and `disableSessionStickiness`.
- The internal context-handoff module writes/upserts records into `context_handoffs`.
- The routing implementation checks handoff state and injects relevant context into chat requests.
- Session IDs have historically been accepted from headers including `x-session-id`, `x-omniroute-session-id`, and `x_session_id`.
- A previous `/api/settings` response showed `disableSessionStickiness: false` and `sessionAffinityTtlMs: 0`.
- `/api/context-relay` was not identified as a route in the previous investigation.
- `/api/v1/relay/chat/completions` was identified as a separate relay-token endpoint; do not confuse it with the internal Context Relay combo strategy.

These are investigation findings, not proof that the currently running deployment behaves exactly as described. Use them to narrow the next investigation rather than repeating the entire source review.

## 9. Current test status and unresolved evidence

A previous request used the session ID:

`context-relay-test-session-001`

It requested the codeword:

`ORBIT-7319`

The response used `max_tokens: 1`, returned empty content, and ended with `finish_reason: length`.

That request is **not evidence of successful context preservation**.

A historical call log showed execution against Account 2, but the combo had an invalid first model connection assignment. Consequently, that call does not establish that the intended two-account handoff works.

A previous query against `context_handoffs` returned no rows after one request. Since a handoff may only be recorded when an account transition occurs, this result alone does not prove either success or failure.

The Context Relay PoC remains **UNPROVEN**.

### What constitutes a valid proof

The PoC must demonstrate all of the following:

1. The combo has two correctly assigned Gemini connections, using the exact connection IDs above.
2. The first request actually executes against the intended first account.
3. A controlled transition to the second account occurs through the intended Context Relay mechanism.
4. The handoff is evidenced by the appropriate application response, logs, or database record.
5. A subsequent request in the same session can recall the unique codeword and meaningful task context from the earlier request.
6. The test results are captured in a concise, sanitized evidence record.

Use enough output tokens for meaningful responses. Do not repeat a test unnecessarily if it consumes limited free-tier quota.

If a valid transition cannot be triggered safely or the relevant free-tier quota is exhausted, mark the test **BLOCKED** with the exact reason. Do not mark it PASS based on an attempted request alone.

## 10. Other previously existing containers and services

Historical container inventory included:

- `omniroute`
- A LiteLLM container, previously associated with port `127.0.0.1:4000`
- A `cloudflared` container

At one historical checkpoint, the LiteLLM and cloudflared containers were stopped. Their existence and current state must be verified before making assumptions.

**Do not remove, recreate, modify, or redeploy the existing LiteLLM installation.** Its configuration and deployment are to be preserved unless a specific change is justified and explicitly approved.

Do not assume that Cloudflare Tunnel is currently running or that public DNS/routing is correctly configured merely because the services existed previously.

## 11. Resource and operational notes

Historical disk observations were:

- Root volume: approximately 25 GiB.
- Root filesystem free space: approximately 14 GiB at the time of inspection.
- `/var/oled`: approximately 20 GiB free at the time of inspection.

These figures are snapshots, not current measurements. Check current free space only if needed for the next action.

A previous Podman investigation encountered a lock warning. Do not delete lock files, repair Podman storage, prune containers, or remove images unless there is concrete evidence of a problem and the proposed operation is approved.

A Python HTTP compatibility test against the OpenAI-compatible endpoint had previously passed. Installing the Python OpenAI SDK is not required merely to repeat that test.

## 12. Security and diagnostic-output policy

Treat this VM as an existing production-like environment containing valuable configuration and persistent data.

- Never print API keys, environment files, authentication cookies, OAuth tokens, or session credentials.
- A file previously used at `/tmp/omniroute-cookie.txt` contained an authentication cookie/token. Never print its contents. Treat previously exposed credentials as potentially compromised and recommend safe rotation if appropriate.
- Do not publish raw diagnostic output from the VM to public GitHub.
- When producing evidence, use field allowlists and redact identifiers or personal data that are not necessary for the test.
- Never include credentials in commands that will be recorded in shell history or logs.
- Prefer read-only checks before configuration changes.
- Explain whether a command is read-only, changes state, restarts a service, or consumes provider quota.
- Do not restart the service or edit configuration just to discover information that can be obtained safely through a read-only check.

## 13. Instructions for continuing the work

Do not restart the project analysis from scratch.

First, use this document and the main handoff document to establish the current checkpoint. Revalidate only the minimum facts necessary to avoid acting on stale historical information.

Then continue with the unresolved Context Relay PoC in this order:

1. Locate the actual combo route implementation or supported configuration mechanism using the deployed application's real filesystem layout.
2. Determine how combo model connection assignments can be updated and validated safely.
3. Correct the test combo through the supported application mechanism.
4. Verify the stored configuration using a narrow, read-only query or sanitized API response.
5. Run a controlled context-handoff test only after the combo is valid.
6. Capture concise evidence and update the project status and test evidence records.
7. If the PoC passes, continue to the architecture-boundary decision described in the main handoff document. If it fails, isolate the failure and present evidence before proposing a patch.

Follow the user's working preference: **one safe command at a time**. Explain the purpose and expected result, wait for the command output, and verify the result before proceeding. Maintain explicit phase/step numbering and enforce each phase's Definition of Done.

Do not combine multiple unverified actions into a single shell command. Do not expose secrets. Do not produce long speculative analyses when a small, targeted check will resolve the uncertainty.

### Required first response

Before taking action, provide:

- A concise summary of the known environment and current checkpoint.
- The exact next step and why it is needed.
- One safe, narrowly scoped command to begin locating the real combo route/configuration mechanism, based on the actual deployed application layout rather than the missing `.next/server/app/api/combos/[id]/route.js` assumption.

Then wait for the user's output before issuing the next command.

The goal is to continue from established evidence—not to repeat the original investigation.