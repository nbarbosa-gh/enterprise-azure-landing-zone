# Architectural Decision Records

This file records significant design decisions made during the build of the Enterprise Azure Landing Zone. Each decision is captured as a numbered ADR (Architectural Decision Record). Update this file whenever a non-trivial design choice is made.

ADR template: Status / Context / Decision / Consequences.

---

## ADR-001: Resource Naming Convention

**Status:** Accepted — 2026-05-03
**Phase:** 1 (Bicep IaC Foundation)

### Context

Azure resource names need to be predictable, sortable, and self-describing across multiple environments (dev/prod), regions (West Europe / North Europe), and workloads. Without a fixed convention, the landing zone becomes unreadable and Bicep modules end up hardcoding ad-hoc names. The Microsoft Cloud Adoption Framework (CAF) recommends a consistent naming pattern; we adopt a CAF-aligned pattern adjusted for this project.

### Decision

All Azure resources will be named using the following format:

```
<resource-type>-<workload>-<environment>-<region>-<instance>
```

Token rules:

| Token           | Rule                                                                                                  | Example         |
| --------------- | ----------------------------------------------------------------------------------------------------- | --------------- |
| `resource-type` | CAF-standard abbreviation (see table below). Lowercase.                                              | `rg`, `kv`, `vnet` |
| `workload`      | Short workload identifier. Lowercase, no separators inside.                                           | `landingzone`   |
| `environment`   | One of: `dev`, `test`, `prod`.                                                                        | `dev`           |
| `region`        | Short region code. `weu` = West Europe, `neu` = North Europe.                                         | `weu`           |
| `instance`      | Three-digit zero-padded counter starting at `001`.                                                    | `001`           |

**Examples:**

- `rg-landingzone-dev-weu-001` — resource group
- `kv-landingzone-dev-weu-001` — Key Vault
- `vnet-hub-dev-weu-001` — hub virtual network
- `vnet-spoke1-dev-weu-001` — first spoke virtual network
- `law-landingzone-dev-weu-001` — Log Analytics workspace
- `aa-landingzone-dev-weu-001` — Automation Account
- `rsv-landingzone-dev-weu-001` — Recovery Services Vault

### Resource type abbreviations (CAF-aligned)

| Resource                       | Abbreviation |
| ------------------------------ | ------------ |
| Resource Group                 | `rg`         |
| Virtual Network                | `vnet`       |
| Subnet                         | `snet`       |
| Network Security Group         | `nsg`        |
| Application Security Group     | `asg`        |
| Route Table                    | `rt`         |
| Public IP                      | `pip`        |
| VPN Gateway                    | `vpng`       |
| Bastion Host                   | `bas`        |
| Load Balancer                  | `lb`         |
| Application Gateway            | `agw`        |
| Front Door                     | `afd`        |
| Private DNS Zone               | `pdnsz`      |
| Private Endpoint               | `pe`         |
| Network Interface              | `nic`        |
| Key Vault                      | `kv`         |
| Storage Account                | `st` (no separators, max 24 chars, lowercase alphanumeric) |
| Log Analytics Workspace        | `law`        |
| Automation Account             | `aa`         |
| Recovery Services Vault        | `rsv`        |
| Virtual Machine                | `vm`         |
| Virtual Machine Scale Set      | `vmss`       |
| App Service Plan               | `asp`        |
| App Service / Web App          | `app`        |
| Container Instance             | `ci`         |
| Container App                  | `ca`         |
| Managed Disk                   | `disk`       |
| Managed Identity (user-assigned) | `id`       |
| Action Group                   | `ag`         |
| Application Insights           | `appi`       |

### Storage account exception

Storage account names cannot contain hyphens, must be 3–24 characters, and must be globally unique. For storage accounts the convention collapses to:

```
st<workload><environment><region><instance>
```

Example: `stlandingzonedevweu001` (22 chars — fits).

If the resulting name exceeds 24 chars, abbreviate `workload` (e.g., `lz` instead of `landingzone`) and document the abbreviation in this ADR.

### Consequences

- Every Bicep module must derive resource names from input parameters using this format. No hardcoded names.
- A `naming.bicep` helper module or string-interpolation pattern in `main.bicep` will enforce this.
- Renaming an existing resource requires deletion + redeploy (Azure does not allow rename). Pick names carefully; the `instance` suffix exists so a v2 of a resource (`-002`) can coexist during cutover.
- Globally-unique resources (Key Vault, Storage Account, public DNS) may collide with names elsewhere in Azure. The `instance` suffix and unique workload prefix mitigate this; if a collision happens we'll append a 4-character hash from `uniqueString(resourceGroup().id)`.

### Rejected alternatives

- **PascalCase or camelCase names** (e.g., `KvLandingZoneDevWeu001`) — rejected because some Azure resource types disallow uppercase (storage accounts) and inconsistency across types is worse than uniform lowercase.
- **CAF naming module from public Bicep registry** (`br/public:avm/...`) — rejected for this lab to maximise learning. Production environments should use it.

---

## ADR-002: Mandatory Resource Tagging

**Status:** Accepted — 2026-05-03
**Phase:** 1 (Bicep IaC Foundation), enforced via Azure Policy in Phase 2

### Context

Tags drive cost reporting, ownership tracking, automation scope, and policy decisions. Without enforced tags, cost allocation and cleanup become impossible. Azure Policy can enforce tag presence and inheritance from the resource group, but only if every resource is tagged from creation.

### Decision

Every resource deployed in this landing zone must carry the following five tags. All Bicep modules accept a `tags` object parameter and apply it to every resource they create.

| Tag           | Required values / format                                                  | Purpose                                                |
| ------------- | ------------------------------------------------------------------------- | ------------------------------------------------------ |
| `environment` | `dev` \| `test` \| `prod`                                                 | Filter resources for cost and lifecycle automation.    |
| `workload`    | Free-form short string matching the workload token in the resource name.  | Group all resources for one application/service.       |
| `costCenter` | Free-form string. For this lab: `lab-personal`.                            | Cost allocation; in production maps to a finance code. |
| `owner`       | Email address of the responsible person.                                  | Accountability and contact for incidents.              |
| `deployedBy` | Always `bicep` for IaC-deployed resources; `manual` is forbidden.          | Distinguishes IaC from drift; supports cleanup audits. |

In Phase 2 a custom Azure Policy initiative will:
- Deny resource creation if any of the five tags are missing.
- Inherit tags from the parent resource group where the resource itself didn't specify them.

### Consequences

- The base Bicep parameter file (`dev.parameters.json` / `prod.parameters.json`) defines the tag object once, and `main.bicep` passes it to every module.
- Resources that do not support tags (e.g., subnets, some classic resources) are exempt; document each exemption here when discovered.
- Tag values are case-sensitive in Azure. Standardise on lowercase to avoid policy false-negatives.

---

## ADR-003: Subscription-scoped deployment for `main.bicep`

**Status:** Proposed — 2026-05-03
**Phase:** 1

### Context

The landing zone needs to create resource groups, then deploy resources into them. A resource-group-scoped Bicep template cannot create a resource group; only subscription-scoped or higher can. Since we cannot use real Management Groups under a single subscription, we simulate hierarchy with multiple resource groups deployed from a subscription-scoped `main.bicep`.

### Decision

`bicep/main.bicep` uses `targetScope = 'subscription'`. It creates the resource groups and then invokes child modules at `scope: rg-name` for resources that live inside those groups.

### Consequences

- Deployments are run with `az deployment sub create --location westeurope ...` rather than `az deployment group create`.
- Requires `Owner` or `Contributor` + `User Access Administrator` on the subscription (Phase 2 will add RBAC role assignments).
- What-If output at subscription scope is more verbose; use `--what-if-result-format FullResourcePayloads` only when debugging.

---
