# CI MCP Server

An MCP (Model Context Protocol) server for SAP Cloud Integration (CPI), powered by [odata-mcp-proxy](https://www.npmjs.com/package/odata-mcp-proxy). It exposes CPI OData APIs as MCP tools, allowing AI assistants like Claude to manage your integration landscape through natural language.

The entire server is defined through a single JSON config file -- no custom code required.

> **Credits.** This repository is based on [lemaiwo/ci-mcp-server](https://github.com/lemaiwo/ci-mcp-server) (MIT). The server definition and tool configuration are the original author's work. The sections marked **[Field notes]** below are my own additions: a worked deployment from the SAP Basis side, a prerequisites checklist, usage prompts for different audiences, and observed limitations.

## How It Works

This project uses the `odata-mcp-proxy` npm package, which maps OData/REST services to MCP tools based on a configuration file. You provide a config describing your APIs and entity sets, and the proxy generates the corresponding MCP tools automatically.

```
AI Assistant (Claude, Cursor, etc.)
        |
        | MCP Protocol (HTTP or stdio)
        v
  odata-mcp-proxy
        |
        | REST + OAuth2 (via BTP Destination Service)
        v
  SAP Cloud Integration OData API
```

Think of it like the [SAP Application Router](https://www.npmjs.com/package/@sap/approuter) -- a ready-made runtime you configure, not code you write.

## Exposed CPI APIs

The config file (`ci-api-config.json`) exposes the SAP Cloud Integration OData API, organized into the following categories:

### Integration Content

| Tool | Operations | Description |
|------|-----------|-------------|
| `IntegrationPackages` | list, get, create, update, delete | Logical containers that group iFlows, value mappings, and other design-time artifacts |
| `IntegrationDesigntimeArtifacts` | list, get, create, update, delete | iFlow design-time definitions (editable integration logic before deployment) |
| `IntegrationRuntimeArtifacts` | list, get | Deployed integration artifacts (deployment status, version, and errors) |
| `ValueMappingDesigntimeArtifacts` | list, get, create, update, delete | Lookup tables that translate codes/identifiers between sender and receiver systems |
| `MessageMappingDesigntimeArtifacts` | list, get, create, update, delete | Graphical structure-to-structure transformations between message formats |
| `ScriptCollectionDesigntimeArtifacts` | list, get, create, update, delete | Reusable Groovy or JavaScript libraries shared across iFlows |
| `CustomTagConfigurations` | list, get, create, update, delete | Tenant-level labels for categorizing and filtering integration packages |
| `BuildAndDeployStatus` | list, get | Track whether an iFlow deployment is queued, running, or finished |

### Message Processing Logs

| Tool | Operations | Description |
|------|-----------|-------------|
| `MessageProcessingLogs` | list, get | Execution history for iFlows, used to debug failed messages or monitor processing |
| `IdMapFromId2s` | list | ID mapping entries for exactly-once processing (source-to-target ID mappings) |
| `IdempotentRepositoryEntries` | list | Duplicate-check records ensuring a message is processed only once |

### Message Stores

| Tool | Operations | Description |
|------|-----------|-------------|
| `DataStoreEntries` | list, get, delete | Key-value records persisted by iFlows for cross-message data sharing |
| `Variables` | list, get | Runtime variables persisted between iFlow executions (timestamps, counters, delta tokens) |
| `NumberRanges` | list, get | Auto-incrementing counters for generating unique sequence numbers |
| `MessageStoreEntries` | list, get | Full messages persisted via the Persist step for later retrieval or retry |
| `JmsBrokers` | list, get | Messaging broker instances provisioned on the tenant |
| `JmsResources` | list | Individual JMS message queues with depth, capacity, and consumer status |

### Log Files

| Tool | Operations | Description |
|------|-----------|-------------|
| `LogFiles` | list, get | Tenant-level runtime logs (HTTP, default trace, audit) for troubleshooting |
| `LogFileArchives` | list, get | Compressed historical log bundles available for download |

### Security Content

| Tool | Operations | Description |
|------|-----------|-------------|
| `KeystoreEntries` | list, get, delete | SSL/TLS certificates, key pairs, and trusted CA certificates |
| `CertificateResources` | list, get | Full X.509 certificate chains for verifying trust paths |
| `SSHKeyResources` | list, get | Public/private key pairs for SFTP adapter connectivity |
| `UserCredentials` | list, get, create, update, delete | Stored username/password pairs for basic-auth connections |
| `OAuth2ClientCredentials` | list, get, create, update, delete | Client ID/secret pairs and token endpoints for OAuth2 connections |
| `SecureParameters` | list, get, create, update, delete | Encrypted key-value entries for sensitive configuration values |
| `CertificateUserMappings` | list, get, create, update, delete | Rules mapping inbound client certificates to CPI user roles |
| `AccessPolicies` | list, get, create, update, delete | Fine-grained authorization rules for integration artifacts |

### Partner Directory

| Tool | Operations | Description |
|------|-----------|-------------|
| `Partners` | list, get, create, update, delete | Trading partner entries driving dynamic iFlow routing |
| `StringParameters` | list, get, create, update, delete | Partner-specific text configuration values (endpoints, format codes) |
| `BinaryParameters` | list, get, create, update, delete | Partner-specific file-based configuration (XSLT, certificates, mappings) |
| `AlternativePartners` | list, get, create, update, delete | Additional partner identifiers (DUNS, GLN) mapping to a primary partner |
| `AuthorizedUsers` | list, get, create, update, delete | Users permitted to send messages on behalf of a specific partner |

All `_list` tools support OData query parameters: `$filter`, `$select`, `$expand`, `$orderby`, `$top`, `$skip`.

## Prerequisites

- **Node.js** 18+ (20+ recommended)
- **SAP BTP account** with a Cloud Foundry environment
- **SAP Cloud Integration** tenant (part of SAP Integration Suite)
- **BTP Destination** configured for the CPI OData API with OAuth2 authentication
- **Cloud Foundry CLI** (`cf`) and **MBT Build Tool** (`mbt`) for deployment

## Project Structure

```
ci-mcp-server/
├── package.json              # Start script + odata-mcp-proxy dependency
├── ci-api-config.json        # API configuration (defines all MCP tools)
├── mta.yaml                  # BTP Cloud Foundry deployment descriptor
├── xs-security.json          # XSUAA OAuth2 configuration
├── default-env.json          # Local dev credentials (gitignored)
└── LICENSE
```

## Getting Started

### 1. Install dependencies

```bash
npm install
```

### 2. Configure BTP destination

Create a BTP Destination pointing to the CPI OData API:

| Destination | URL |
|-------------|-----|
| `CPI_DESTINATION` | `https://<tenant>.it-cpi0<xx>.cfapps.<region>.hana.ondemand.com` |

The destination should use OAuth2 client credentials authentication with the CPI service key credentials.

### 3. Local development

Create a `default-env.json` with your BTP service bindings (XSUAA, Destination, Connectivity) to run locally:

```bash
npm start
```

This runs `odata-mcp-proxy --config ci-api-config.json`.

### 4. Deploy to BTP

```bash
npm run build:btp     # Build MTA archive
npm run deploy:btp    # Deploy to Cloud Foundry
```

The MTA deployment provisions three service instances:
- **Destination** (lite) -- resolves the CPI API endpoint and manages OAuth2 tokens
- **Connectivity** (lite) -- enables secure backend connectivity
- **XSUAA** (application) -- handles OAuth2 authentication with role-based access control

## Security

The XSUAA configuration (`xs-security.json`) defines three role templates:

| Role | Scopes | Description |
|------|--------|-------------|
| `MCPViewer` | read | Read-only access to CPI data |
| `MCPEditor` | read, write | Read and modify CPI data |
| `MCPAdmin` | read, write, admin | Full administrative access |

OAuth2 redirect URIs are pre-configured for Claude.ai, Cursor, Microsoft Teams, and local development.

## [Field notes] Deployment walkthrough (Basis perspective)

Written by an SAP Basis administrator, not a developer. This is the path that worked, including a setup where **the MCP app and the CPI tenant live in different BTP subaccounts and different regions**. That is fine: the app reaches CPI over the internet through a Destination.

### Prerequisites checklist

| # | Item | Where it comes from |
|---|------|---------------------|
| 1 | Node.js 18+ | nodejs.org |
| 2 | Cloud Foundry CLI (`cf`) and the **MultiApps plugin** (`cf install-plugin multiapps -r CF-Community`) | Needed for `cf deploy` |
| 3 | MBT build tool (`npm i -g mbt`) | On Windows it also needs **GNU make** (e.g. `winget install ezwinports.make`) |
| 4 | BTP subaccount with Cloud Foundry enabled, a space, and entitlements for **Destination**, **Connectivity** and **XSUAA** | BTP cockpit, in the *target* subaccount |
| 5 | A CPI tenant (SAP Integration Suite) | Its runtime URL |
| 6 | A CPI **service key**: service *SAP Process Integration Runtime*, plan `api` | Gives `url`, `clientid`, `clientsecret`, `tokenurl` |
| 7 | Roles on that service **instance** (set at instance creation, **not** shown in the key) | See "Roles" below |
| 8 | A Destination named exactly `CPI_DESTINATION` in the target subaccount | See "Destination" below |
| 9 | A role collection (**MCP Viewer / Editor / Administrator**) assigned to each user | BTP cockpit, in the target subaccount |

### Roles on the CPI service instance
Roles are defined in the instance parameters, not in the service key. To cover all the tool groups in this config you need roughly: `WorkspacePackages*` (packages and artifacts), `WorkspaceArtifactsDeploy` and `MonitoringArtifactsDeploy` (deployments), `MonitoringDataRead` (message logs, runtime), `DataStoresAndQueues*` and `MessagePayloadsRead` (message stores), `CredentialsEdit` and `SecurityMaterial*` (security content), `AccessPolicies*`, and `AuthGroup_TenantPartnerDirectoryConfigurator` (Partner Directory). A tool returning HTTP 403 usually means one role is missing.

### Destination `CPI_DESTINATION`
Create it in the **target** subaccount (Connectivity > Destinations):

| Field | Value |
|-------|-------|
| URL | `url` from the service key. Do **not** append `/api/v1`: the server adds that itself |
| Authentication | `OAuth2ClientCredentials` |
| Client ID / Secret | `clientid` / `clientsecret` from the key |
| Token Service URL | `tokenurl` from the key, **exactly as given** (it already ends in `/oauth/token`) |
| Type / Proxy | `HTTP` / `Internet` |

Use **Check Connection** afterwards. Never commit these values: they belong only in the BTP Destination.

### Build, validate, deploy
```bash
npm install
npm start                        # local check: http://localhost:4004/health and /mcp
mbt build                        # produces mta_archives/ci-mcp-server_1.0.0.mtar
cf login -a <target CF API endpoint>
cf deploy mta_archives/ci-mcp-server_1.0.0.mtar
```
The local run proves that the server starts and exposes its tools (187 with this config). It cannot reach CPI, because the Destination service only exists on BTP. Live calls are validated after deployment.

### Connecting Claude
In Claude, add a custom connector with the URL `https://<your-app-route>/mcp`. Use **Sign in now** and **Register automatically** (DCR). The server's own XSUAA login handles authentication, so the CPI credentials are never entered in Claude.

## [Field notes] Using it: example prompts by audience

Ask for **small, bounded questions** ("last 24 hours", "top 5"). Wide queries return very large responses.

| Audience | Prompt |
|----------|--------|
| **Management** | *Summarise the last 7 days in plain language: messages succeeded and failed, the 5 interfaces with the most failures, and which business process each affects (orders, payments, payroll, inventory, pricing). Flag anything not currently running. Under one page, no jargon.* |
| **Key users** | *Check whether the interface for [orders / bank statement / price file] ran today and succeeded. If it failed, explain in simple words and say who to contact.* |
| **Functional team** | *For the last 24 hours, list failed messages for [area]. Show time, sender and the error in plain English. Group repeats and say whether it looks like a data, connection or configuration problem.* |
| **Basis** | *List every deployed iFlow that is not in STARTED state, with the reason, ranked by business importance. Also list blocked or filling queues.* |
| **Observability** | *Which iFlows leave no custom header (business key) on their messages? Rank by business criticality and explain what we could not trace if a message failed.* |
| **Housekeeping** | *Which iFlows are deployed but never executed in 30 days? Which certificates or credentials expire in the next 60 days?* |
| **Weekly priority** | *Which 3 interfaces, if fixed, would remove the most failures and business risk? Justify with failure counts, business process and traceability.* |

### What this found in practice (anonymised)
- An interface in deployment **ERROR** state produces **no failed messages**, so message monitoring looks green while the interface is actually down. Check runtime artifact status, not only message logs.
- Two chained iFlows (router and follow-up step) failed together at the same clock time every night, which pointed to a scheduled upstream job: counted per chain, that is one incident, not two.
- Several business-critical iFlows (order retrieval, bank statements, inventory) wrote **no custom headers**, so a failed run could not be traced to a business document.

## [Field notes] Governance and limitations

- **Least privilege.** The config exposes create, update and delete tools as well as read. Give business users **MCP Viewer** only.
- **`$select` was rejected** by the server in testing (`$select is not supported`), although the tool descriptions mention it. `$top`, `$filter` and `$orderby` worked. Because of this, list calls return full records.
- **Large tenants.** High-frequency polling flows can dominate time-window queries. Exclude them with `$filter` or shorten the window.
- **Design-level questions** (retry handling, exception subprocesses) need the inside of an iFlow, which the runtime tools do not reliably show. Treat those as out of scope.
- **Results are samples** unless a query is explicitly complete. Always state the time window.
- **Keep customer data out of public material.** Real tenant output contains partner names and personal e-mail addresses. Anonymise before sharing.

## License

MIT
