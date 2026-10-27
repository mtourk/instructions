# Oracle VM Deep Handoff

**Purpose:** VM-specific supplement for the OmniRoute project. Read
alongside: -
https://github.com/mtourk/instructions/blob/main/OMNIROUTE_AI_PLATFORM_MASTER_HANDOFF.md -
https://github.com/mtourk/instructions/blob/main/Supplementary%20Environment%20Handoff.md -
https://github.com/mtourk/instructions/blob/main/MASTER%20HANDOFF%20PROMPT.md

**Prepared:** 27 October 2026. The latest detailed VM snapshot in the
available project notes was reported on 11 October 2026. This is
historical evidence, not live access to the VM.

## 1. Security: secrets are intentionally not embedded

This document is intended for a **public GitHub repository**. It records
known infrastructure facts and where sensitive configuration may be
found, but it must **not** contain actual passwords, provider keys,
authentication tokens/cookies, private keys, OCI credentials, database
passwords, or Cloudflare Tunnel credentials. Keep actual values in a
private password/secret manager or protected configuration. Do not ask
the owner to paste credentials into chat.

Never print or publish environment files, authentication cookie files,
unfiltered container inspection output, shell history, raw logs
containing authorization headers, or complete database dumps. If an
actual credential was exposed, assess revocation/rotation through the
owning provider.

Known sensitive locations: - `/app/data/server.env` --- previously noted
as sensitive; never print its contents. - `/tmp/omniroute-cookie.txt`
--- previously used for an authentication cookie/token; never print its
contents. Assess safe rotation if it may have been exposed. - Podman
container environment, Quadlet files, Cloudflare configuration, shell
history, and application/database configuration may contain sensitive
values. Inspect only specific non-secret fields. - Gemini connection IDs
listed below are identifiers, not provider keys.

## 2. VM identity and resources --- historical values

  -----------------------------------------------------------------------
  Field                   Last known value        Qualification
  ----------------------- ----------------------- -----------------------
  Cloud                   Oracle Cloud            Historical
                          Infrastructure (OCI)    

  Hostname / prompt       `litellm-gateway`       Historical

  Linux user              `opc`                   Seen in shell prompt
                                                  and paths

  Operating system        Oracle Linux Server     Reported 11 Oct 2026
                          10.2                    

  Architecture            ARM64 / `aarch64`       Reported 11 Oct 2026

  CPU                     2 vCPUs                 Historical

  RAM                     10 GiB                  Historical

  Swap                    4 GiB                   Historical

  Container runtime       Podman 5.8.2            Historical

  Docker / Compose        Reported unavailable    Historical; may have
                                                  changed

  Private IP              `10.0.0.63/24`          Historical

  Default gateway         `10.0.0.1`              Historical

  Root filesystem         25 GiB; about 6.3 GiB   Snapshot from 11 Oct
                          free (75% used)         

  Other path              `/var/oled` previously  Exact mount/current
                          reported with about 20  state unknown
                          GiB free                
  -----------------------------------------------------------------------

Do not confuse this existing VM's reported **10 GiB RAM** with an
earlier proposed OCI configuration of 2 OCPU / 12 GiB RAM. The proposed
configuration was not proof of the live machine's specs.

## 3. Podman containers --- last known snapshot from 11 October 2026

  --------------------------------------------------------------------------------------------
  Container               State then              Image / details
  ----------------------- ----------------------- --------------------------------------------
  `omniroute`             Running                 `docker.io/diegosouzapw/omniroute:3.8.51`;
                                                  bound to `127.0.0.1:20128`; pasta network

  `litellm`               Exited                  `berriai/litellm:v1.103.0`; historically
                                                  associated with `127.0.0.1:4000`

  `cloudflared`           Exited, code 2          `cloudflared:latest`; host network; restart
                                                  policy previously reported as
                                                  `unless-stopped`
  --------------------------------------------------------------------------------------------

These are historical states; verify current status before action. One
inventory reported OmniRoute restart=no, while the historical Quadlet
specified `Restart=always`. Inspect current configuration if this
discrepancy matters.

**Preserve existing services and data.** Do not remove/recreate
containers, prune images or volumes, reset Podman storage, delete lock
files, update OmniRoute, or modify LiteLLM as part of the Context Relay
investigation. Do not restart services just to gather information.

## 4. OmniRoute deployment and paths

Historical details: - Container: `omniroute` - Image:
`docker.io/diegosouzapw/omniroute:3.8.51` - Historical ARM64 image
digest:
`sha256:a1b425867ba4a5b250ca101382f58fc8d5adbfb3790a69464a8028768af93eba` -
Historical associated commit: `c1e30b7` - Container working directory:
`/app` - Startup entry: `/app/dev/run-standalone.mjs` - Historical
launch: `node dev/run-standalone.mjs`, invoked via
`/app/check-permissions.sh`

Historical Quadlet path:
`/home/opc/.config/containers/systemd/omniroute.container`

Historical Quadlet snapshot (**do not overwrite the current file with
this copy**):

``` ini
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

The `opc` user was previously configured for systemd lingering with
`sudo loginctl enable-linger opc`; persistence across reboots had
previously been tested. Current state remains unverified.

### Persistent storage

  ------------------------------------------------------------------------------------------------
  Purpose                 Container path               Expected host path
  ----------------------- ---------------------------- -------------------------------------------
  Data directory          `/app/data`                  `/home/opc/omniroute/data`

  SQLite database         `/app/data/storage.sqlite`   `/home/opc/omniroute/data/storage.sqlite`

  Sensitive               `/app/data/server.env`       Expected
  environment/config                                   `/home/opc/omniroute/data/server.env` if
  noted earlier                                        the historical mount remains
  ------------------------------------------------------------------------------------------------

Confirm the current mount before relying on these paths. The database
previously contained `combos` and `context_handoffs`; a previous
integrity check returned `ok`. Never dump the whole database or edit it
directly without an explicit, justified plan. Prefer the supported
application API/UI for configuration changes.

Historical `combos` columns: `id`, `name`, `data`, `sort_order`,
`created_at`, `updated_at`, `system_message`, `tool_filter_regex`,
`context_cache_protection`.

Historical `context_handoffs` fields included: `session_id`,
`combo_name`, `from_account`, `summary`, `key_decisions`,
`task_progress`, `active_entities`, `message_count`, `model`,
`last_model`, `warning_threshold_pct`, `generated_at`, `expires_at`.
These are historical; inspect the current schema before querying.

## 5. Source layout and API investigation

The expected path `/app/.next/server/app/api/combos/[id]/route.js` was
not found in an earlier check. Do not repeat that assumption and do not
infer the route is missing.

Historical actual path:
`/app/.build/next/server/app/api/combos/[id]/route.js`

Other historical paths: - Catch-all:
`/app/.build/next/server/app/api/[...omnirouteApiCatchAll]/route.js` -
Trace manifest:
`/app/.build/next/server/app/api/combos/[id]/route.js.nft.json` -
Bundle: `/app/.build/next/server/chunks/1642.js` - Context-handoff
bundle previously identified as `chunks/56268.js` (confirm exact current
path) - Modules `401642` and `385179` were found in `1642.js`; module
`385179` related to combo schema/weights. - Module `786319` was
associated with context-handoff logic in prior investigation.

Historical route names included `/api/combos`, `/api/combos/[id]`,
`/api/combos/test`, `/api/context/combos`, `/api/context/combos/[id]`,
`/api/context/combos/[id]/assignments`, `/api/context/combos/default`,
and `/api/v1/combos`. These are route identifiers, not guarantees about
current files. Earlier combo route exports included GET, PATCH, PUT,
DELETE.

Use targeted, read-only investigation of the deployed layout. Avoid
broad recursive searches and giant grep output. Prefer a supported
API/UI for changes; never guess an update endpoint.

## 6. Context Relay test identifiers and unresolved status

Gemini saved-connection IDs (not provider keys): - Account 1:
`ae9ef7ed-fd4b-4773-bb14-8b67d889ac79` - Account 2:
`2c27504c-e8e6-4189-b7f1-3950acb5e88c`

An earlier typo used `4188` instead of the correct `4189` in Account 2's
ID and generated `notFound` audit entries. Do not repeat it.

Historical test combo: - Name: `context-relay-gemini-test` - ID:
`d5a0e969-06d4-4f45-8140-059e827e13b5` - Strategy: `context-relay` -
Model: `gemini-3.1-flash-lite` - Historical config version: 2

The previously observed config was defective: one model entry had
`connectionId: null`, weight 0, and a generated ID containing the old
4188 fragment; another entry was pinned to Account 1, weight 0; config
was `{}`. GET `/api/combos/context-relay-gemini-test` returned
`COMBO_007`, while the collection endpoint listed the combo. Do not
assume the name is accepted as the item identifier.

Historical session: `context-relay-test-session-001`; codeword:
`ORBIT-7319`; expected later response: `FIRST-PASS`. A request with
`max_tokens: 1` returned empty content with `finish_reason: length`. A
call log showed Account 2, but the combo was defective; a previous query
found no handoff rows. This does **not** prove success or failure.
Context Relay remains **UNPROVEN**.

A valid test must show correct two-account assignments, first-account
execution, a controlled transition to the second account, evidence of
handoff, and recall of the codeword/task context in the same session.
Use enough output tokens, check quota before testing, and save only
sanitized evidence. If unavailable, mark BLOCKED rather than PASS.

## 7. Network, ports, and Cloudflare

Historical listener report: - `22`: SSH. - `127.0.0.1:20128`: OmniRoute,
loopback only. - `127.0.0.1:4000`: historically associated with
LiteLLM. - `4330`, `44321`, and RPC `111` were also seen listening;
purpose/current exposure not established.

A prior report found no nginx/Caddy/Traefik/cloudflared/Ollama binaries
and no matching running systemd services; Docker Compose was
unavailable. This is historical and does not rule out containers or
later changes.

Public hostname historically documented: `llm.mtourk.com`. Cloudflare
DNS/proxy and Tunnel were part of the intended architecture,
historically routing to LiteLLM. Current DNS, tunnel UUID, ingress,
origin, access policies, and service status have not been verified. The
cloudflared container was exited with code 2 at one checkpoint; cause
unknown.

Do not expose port `20128` to the internet. Do not change
`llm.mtourk.com` routing or point it at OmniRoute without inspecting the
existing setup and obtaining approval. Never publish tunnel credentials.

## 8. Secret register --- labels/locations only

  -----------------------------------------------------------------------------
  Type                    Possible location             Handling
  ----------------------- ----------------------------- -----------------------
  Gemini provider         Saved OmniRoute connections   Never print raw values;
  credentials             and/or protected app config   inspect only sanitized
                                                        connection metadata

  OmniRoute cookie/token  Previously                    Never print; assess
                          `/tmp/omniroute-cookie.txt`   revocation if exposed

  Application secrets     Previously                    Never print, upload, or
                          `/app/data/server.env`        copy to GitHub

  Cloudflare Tunnel       Container environment,        Inspect storage method
  credentials             mounted config/credential     only; no unfiltered
                          file, or dashboard            environment/inspect
                                                        output

  LiteLLM master/provider Container environment,        Preserve deployment;
  credentials             config, mounted files, or     redact values
                          database                      

  OCI credentials / SSH   OCI account and               Not recorded here;
  private key             administrator's protected     never paste into chat
                          files                         

  Supabase credentials    External secret storage or    Exact current VM
                          app config                    location unknown

  GitHub credentials      Connected integration or      Never commit tokens
                          protected credential manager  
  -----------------------------------------------------------------------------

Do not create a public env file, password list, credential dump, raw
Podman inspect output, or shell-history export. A private password
manager should hold secret values; this public file should contain only
labels, purpose, owner, rotation metadata, and secure retrieval
instructions.

## 9. Historical tests --- don't repeat without a reason

Previously reported passing checks: ARM64 startup, SQLite
migrations/integrity, health endpoint, Gemini inference, account
selection, HTTP 429 quota fallback Account 2→Account 1, cooldown when
both accounts exhausted, systemd persistence, 10 concurrent requests,
streaming, and direct OpenAI-compatible HTTP endpoint.

A 20-concurrent-request test produced 17 successes and 3 cooldown
failures when both accounts were cooling down. Temporary
invalid-credential HTTP 400 did not trigger fallback in tested v3.8.51.
These are historical/version-specific observations; don't build a
duplicate fallback engine or repeat completed tests without a concrete
reason.

## 10. Read-only inventory plan

Follow the owner's strict workflow: **one command at a time**, explain
purpose and whether it changes state, wait for output, then interpret it
before proceeding. Do not batch commands into one giant script.

Suggested first command only if the facts are not already freshly
verified:

``` bash
printf '%s\\n' '--- identity ---'
hostname
id
printf '%s\\n' '--- OS / architecture ---'
grep -E '^(NAME|VERSION|PRETTY_NAME)=' /etc/os-release
uname -m
printf '%s\\n' '--- CPU / memory / swap ---'
nproc
free -h
printf '%s\\n' '--- root filesystem ---'
df -h /
```

This is read-only and does not intentionally print credentials. Wait for
output before choosing another command.

When needed, use narrow checks for Podman container
name/image/state/ports, current Quadlet non-sensitive directives,
listener addresses, mounts, or SQLite schema. Avoid unfiltered
`podman inspect`, `env`, `printenv`, reading `server.env`, broad logs,
and full database dumps.

## 11. Rules before state-changing operations

Before any change, explain phase/step, objective, affected resources,
impact, whether it restarts services/changes routing/modifies persistent
data/consumes quota, and rollback. Obtain approval for consequential
changes. One action at a time.

Never restart the VM, change firewall rules, expose ports, edit
configuration, update images, delete files, alter Cloudflare routing, or
modify SQLite just to discover information.

## 12. Next checkpoint

Continue from the unresolved OmniRoute Context Relay proof of concept,
not from a fresh infrastructure build: 1. Read the main handoff and
supplementary environment handoff. 2. Revalidate only necessary live
facts. 3. Inspect the test combo and supported update mechanism
read-only. 4. Correct the disposable test combo only via the supported
mechanism and after explaining impact. 5. Read back sanitized config. 6.
Run controlled handoff test only when config is valid and quota is
available. 7. Classify PASS / FAIL / BLOCKED honestly and preserve
sanitized evidence. 8. Interview the owner and decide whether OmniRoute
alone meets requirements before designing a separate AI Platform
database.

The owner previously said not to spend more time on the historical
response `{"version":"4","deployed_at":"2026-10-04T20:35:00+02:00"}` and
`{"error":"Not found"}` unless it directly blocks acceptance criteria.

**Bottom line:** this file records known paths, service history,
connection identifiers, and safe secret handling. Current state must be
reverified before any consequential operation.
