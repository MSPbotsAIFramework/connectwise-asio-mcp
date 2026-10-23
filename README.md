# connectwise-asio-mcp

**ConnectWise Asio** MCP server — exposes the ConnectWise Platform APIs (the "Current API" behind ConnectWise Asio, formerly SolarWinds MSP/N-central-adjacent RMM) as MCP tools.

> **Naming note:** app.mspbots.ai calls this integration **"ConnectWise Asio"** (`sys_integration.subject_code = CWASIO`). It is a distinct product from ConnectWise Command (Continuum), ConnectWise Manage (PSA), and ConnectWise Automate — each has its own MCP server in this fleet. The underlying API is officially named **"ConnectWise Platform APIs"** in its OpenAPI spec (`developer.connectwise.com` — "REST API" doc under Products > ConnectWise Platform > Platform Vendors > APIs and Callbacks).

<div align="center">

## MSPbots Connector

**The MCP Gateway built for MSPs: visible, controlled, and your data stays where it belongs.**

[**Get started at connect.mspbots.ai →**](https://connect.mspbots.ai)

</div>

---

MSPbots Connector gives your AI clients one MCP URL to reach your entire MSP stack (RMM, PSA, documentation, security, finance), and gives you control over who can call which tool and a record of everything that was called.

**Hosted service: [connect.mspbots.ai](https://connect.mspbots.ai)** · Free to use.

### Why Connector

![MSPbots Connector](https://stgoss.mspbots.ai/files/mcp/mspbots-mcp.png)

| | |
|---|---|
| **Visible** | Every tool call is logged: who, which vendor, when. Usage dashboard, CSV export, SIEM streaming. |
| **Controlled** | Role-based, per-tool allowlist. Deny by default. |
| **Protected** | Credentials encrypted per record. AI clients never see your vendor passwords or tokens. |
| **MSP-native** | Built for the MSP stack only: RMM, PSA, documentation, security, finance. |

### Quick start

1. **Sign up** at [connect.mspbots.ai](https://connect.mspbots.ai).
2. **Connect your vendors.** Pick from 150+ connectors and authorize each one on the vendor's own page, or bring your own API keys. No extra vendor accounts needed.
3. **Set access.** Create roles and decide which tools each role may call. Nothing is allowed until you allow it.
4. **Add your MCP URL to your AI client.** Copy the URL from your Connector dashboard into Claude, ChatGPT, Cursor, VS Code, Gemini, Codex, or any MCP-capable client.
5. **Ask your AI to do real work**, and watch every call appear in the audit log.

### Features

**Unified gateway**
- One MCP URL for all your AI clients
- 150+ connectors across the MSP stack
- Custom MCP servers supported, free
- Authorization happens on each vendor's own page; the AI never sees passwords or tokens
- Bring your own credentials: use the vendor API keys you already have
- Multiple accounts per app; connect once, reuse across AI clients
- Tool search: finds the right action for the task
- One sign-in; vendor credentials are injected behind the scenes and OAuth tokens refresh automatically

**Visibility**
- Audit log for every tool call (who, which vendor, when)
- Usage dashboard: call volume, top vendors, error rate, per-user activity
- Export audit logs to CSV, or stream to Loki, Graylog, or LogScale (SIEM)
- Optional recording of tool arguments and result summaries (off by default, enabled per organization)

**Control**
- Per-role, per-tool allowlist, deny by default
- owner / admin / member roles
- SCIM provisioning with Entra ID, Okta, JumpCloud, and Google Workspace

**Security**
- Credentials encrypted one record at a time (AES-256-GCM, per-record key)
- AI clients authenticate with OAuth 2.1 + PKCE
- Vendor responses are scanned for prompt-injection patterns and returned as structured data. This is one layer of defense, not a guarantee against every attack.
- Static egress IP for customer firewall and API allowlisting

### Pricing

**Free.**

Links: [connect.mspbots.ai](https://connect.mspbots.ai) · [mspbots.ai](https://mspbots.ai)

---

## Overview

This server implements the [Model Context Protocol](https://modelcontextprotocol.io/) (Streamable HTTP/SSE transport) and wraps the ConnectWise Platform APIs — **15 tools** across 4 categories (`devices`, `companies_contacts`, `patching`, `automation`), trimmed to the core MSP RMM workflow: find a company/site, find its devices, check device health and patch status (see Tool List below; originally a full-API build of 107 operations across 24 tag categories, then a 25-tool MSPbots-configured-endpoints trim, then this 15-tool core-workflow trim on 2026-08-26 — see Known Gaps for what was cut and why). It follows the MSPbots **Vendor MCP Service SOP**: stateless, no stored credentials, per-request header authentication.

The underlying API authenticates via **OAuth2 Client Credentials**. This server does not perform the token exchange itself — it receives an already-obtained bearer access token per request (matching how other OAuth-flow integrations, e.g. Microsoft Graph, are handled in this MSPbots MCP fleet: the gateway mints/refreshes the token from `client_id`/`client_secret`/`scope`, and only the resulting token reaches this server).

## Quick Start

### Docker (recommended)

```bash
docker compose up --build
```

The server starts on `http://localhost:8080`.

### Local (uv)

```bash
uv sync
python -m connectwise_asio_mcp
```

## Health Check

```bash
curl http://localhost:8080/health
# {"status": "ok"}
```

No token is required for the health endpoint.

## Authentication parameters (Authentication)

Every request to `/mcp` must include the following HTTP headers:

| Header | Type | Required | Default | Allowed values | Description | Example |
|---|---|---|---|---|---|---|
| `X-ConnectWise-Asio-Token` | string | Yes | None | None (free text) | OAuth2 bearer access token, obtained in advance and refreshed by the caller (Agent Platform / upstream gateway) via `POST /v1/token` (client_credentials grant, requires client_id + client_secret + scope). This service never touches client_id/client_secret and only forwards the already obtained token. | `X-ConnectWise-Asio-Token: <access_token>` |
| `X-ConnectWise-Asio-Base-Url` | string | Optional | `CONNECTWISE_ASIO_BASE_URL` environment variable (default `https://openapi.service.itsupport247.net`) | `https://openapi.service.itsupport247.net` (NA), `https://openapi.service.euplatform.connectwise.com` (EU), `https://openapi.service.auplatform.connectwise.com` (AU) | ConnectWise Asio has 3 independent servers by the region the account belongs to (the Client ID prefix 0e30=NA/0e31=EU/0e32=AU identifies the region); without this header the environment variable default is used, and a gateway serving tenants in multiple regions overrides it per request. | `X-ConnectWise-Asio-Base-Url: https://openapi.service.euplatform.connectwise.com` |

Missing `X-ConnectWise-Asio-Token` returns `401 Unauthorized`.

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `MCP_HTTP_PORT` | `8080` | Listening port |
| `MCP_HTTP_HOST` | `0.0.0.0` | Listening host |
| `CONNECTWISE_ASIO_BASE_URL` | `https://openapi.service.itsupport247.net` | Default regional server; override per-request with `X-ConnectWise-Asio-Base-Url` |

## MCP Endpoint

```
POST http://localhost:8080/mcp
```

Connect your MCP client with:
- Transport: `http` (Streamable HTTP / SSE)
- Headers: `X-ConnectWise-Asio-Token: <access_token>` (required), `X-ConnectWise-Asio-Base-Url: <base_url>` (optional)

## Tool List

**15 tools.** History: an original 104-tool full-API build (covering every ConnectWise Platform APIs operation) was trimmed on 2026-08-04 to 25 tools matching MSPbots' actually-configured 23 endpoints. On 2026-08-26, a further trim to these 15 tools extracted the core MSP RMM workflow — find a company/site, find its devices, check device health and patch status — after an intent-matching test pass found the removed tools added surface area (and real description/schema gaps) without matching real usage. If a removed tool/category is needed later, the vendor's OpenAPI spec (linked below) still documents its exact operations and it can be re-added the same way the kept tools were generated; git history has the removed tools' implementations verbatim.

**Removed in the 2026-08-26 trim** (10 tools): the entire Custom Fields category (6 tools — config/metadata management, not core monitoring), `connectwise_asio_create_company` (the only write tool; client onboarding isn't an agent task — this server is now fully read-only), `connectwise_asio_get_site` (redundant with `get_company_sites`), `connectwise_asio_get_endpoint_applications` (software-inventory audit, niche), `connectwise_asio_get_endpoint_system_state` (overlapped with the cpu/memory/disk/services tools and had an ambiguous own description).

**Known gap carried over from the original build**: MSPbots configured two endpoints — "ConnectWise Asio Suspension" (`GET /v2/alerting/suspensions`) and "ConnectWise Asio Suspension Details" (`GET /v2/alerting/suspensions/{ruleid}`) — that have **no corresponding tool** in this server at all (the original 104-tool build's `alerting` category only ever implemented create/update/delete/post-incident operations, never a GET-suspension read). This predates both trims. See Known Gaps below.

### Companies & Contacts (4)

| Tool | Description | Params |
|---|---|---|
| `connectwise_asio_get_companies` | List all companies (clients) for the partner. | none |
| `connectwise_asio_get_company` | Get a company by ID. | company_id |
| `connectwise_asio_get_company_sites` | List all sites for a company. | company_id |
| `connectwise_asio_get_sites` | List all sites for the partner (across all companies). | none |

### Devices & Endpoints (7)

| Tool | Description | Params |
|---|---|---|
| `connectwise_asio_list_endpoints` | List devices (endpoints) for a given category, scoped to specific resources — the entry point for device discovery. | category, resource_type, resources, limit, cursor, sort_by?, field?, filter? |
| `connectwise_asio_get_endpoint` | Get full details of a specific endpoint (device). Needs company_id/site_id/endpoint_id already known — use list_endpoints first. | company_id, site_id, endpoint_id, field? |
| `connectwise_asio_get_endpoint_services` | Get service (Windows service / daemon) details for an endpoint. Needs all 3 IDs already known. | company_id, site_id, endpoint_id, service_name?, field? |
| `connectwise_asio_get_endpoint_cpu_usage` | Get the latest CPU usage for an endpoint. | endpoint_id |
| `connectwise_asio_get_endpoint_disk_usage` | Get the latest disk usage for an endpoint. | endpoint_id |
| `connectwise_asio_get_endpoint_memory_usage` | Get the latest memory usage for an endpoint. | endpoint_id, minute? |
| `connectwise_asio_get_endpoint_heartbeat` | Get online/offline availability information for devices (bulk). | resource_type, resources, offline_lookback?, availability? |

### Patching (3)

| Tool | Description | Params |
|---|---|---|
| `connectwise_asio_get_endpoint_patches` | Get the list of OS patches for an endpoint. | endpoint_id |
| `connectwise_asio_get_os_patch_compliance_summary` | Get bulk OS patch compliance summary for a set of resources. | resource_type, resources, limit, cursor, field? |
| `connectwise_asio_get_os_patch_details` | Get bulk OS patch details for a set of endpoints. | resource_type, resources, field? |

### Automation (1)

| Tool | Description | Params |
|---|---|---|
| `connectwise_asio_get_automation_tasks` | Get all automation tasks for the partner. | none |


## Test Example

Get a company list, then its sites:

```json
{
  "method": "tools/call",
  "params": { "name": "connectwise_asio_get_companies", "arguments": {} }
}
```

Equivalent `curl` against the running server (streamable HTTP MCP endpoint):

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "X-ConnectWise-Asio-Token: <access_token>" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": { "name": "connectwise_asio_get_companies", "arguments": {} }
  }'
```

A parameterized call:

```json
{
  "method": "tools/call",
  "params": {
    "name": "connectwise_asio_get_endpoint_cpu_usage",
    "arguments": { "endpoint_id": "abc-123" }
  }
}
```

## API Reference

- Official docs (ConnectWise Developer Network — requires login): Products > ConnectWise Platform > Platform Vendors > APIs and Callbacks > REST API
- Base URLs: NA `https://openapi.service.itsupport247.net`, EU `https://openapi.service.euplatform.connectwise.com`, AU `https://openapi.service.auplatform.connectwise.com`

## Known Gaps / Not Yet Verified

- **Trimmed twice**: 104 tools (full API spec) → 25 tools on 2026-08-04
  (MSPbots' actually-configured endpoints) → **15 tools on 2026-08-26**
  (core RMM workflow only — see Tool List above for exactly what was cut
  each time and why). If a removed tool/category is needed later, the
  vendor's OpenAPI spec (linked below) still documents its exact
  operations and it can be re-added the same way the kept tools were
  generated; git history has every removed tool's implementation verbatim.
- **MSPbots' configured "Suspension"/"Suspension Details" endpoints have no
  corresponding tool** — this gap predates both trims (the original
  104-tool build's `alerting` category never implemented a GET-suspension
  read operation, only create/update/delete/post-incident). If suspension
  data is actually needed, this would need to be added from the vendor's
  OpenAPI spec.
- Not yet tested against a live ConnectWise Asio account — only
  protocol-level verification (health check, 401 on missing token,
  `tools/list` returning all 15 tools) plus an intent-matching test pass
  (real dispatch against a local server with a dummy token — confirms
  tool/argument selection, not live API behavior) has been done so far.
- This server is now fully read-only (both write tools — `create_company`,
  `update_company_custom_fields` — were removed in the 2026-08-26 trim).
- `connectwise_asio_get_os_patch_compliance_summary`'s `resource_type`
  accepting values other than `"endpoints"` (e.g. a company/site-level
  rollup) is unverified — the vendor's docs are behind a login wall. Its
  sibling `connectwise_asio_get_os_patch_details` is confirmed
  endpoints-only.
