# Enterprise Azure Landing Zone

A hands-on, end-to-end Azure landing zone built to demonstrate enterprise-grade patterns across IaC, identity, networking, hybrid AD, automation, backup, and observability.

## Project scope

Single Azure subscription. Primary region **West Europe**, secondary (DR) region **North Europe**. Management Group hierarchy is simulated with resource groups. All infrastructure is deployed via Bicep — no portal-driven resource creation.

## Phases

1. Bicep IaC Foundation
2. Identity & Governance (RBAC, Policy, Cost Management, Entra ID)
3. Storage & Data Services
4. Networking (Hub-and-Spoke, Load Balancer, App Gateway WAF, DNS, Network Watcher)
5. Compute Workloads (VMs, VMSS, App Service, ACI, Container Apps)
6. Hybrid Identity (on-prem AD DS, Entra Connect, Conditional Access)
7. Azure Automation & Runbooks
8. Backup & Site Recovery
9. Observability, Monitoring & Defender for Cloud

## Repository layout

```
.
├── bicep/                    # Infrastructure as Code (Bicep)
│   ├── main.bicep            # Subscription-scoped entry point
│   ├── modules/              # Reusable resource modules
│   └── parameters/           # Per-environment parameter files
├── runbooks/                 # PowerShell runbooks for Azure Automation
├── policy/                   # Custom RBAC roles and policy definitions
├── docs/                     # Architecture decisions, diagrams, runbooks
│   └── decisions.md          # ADR log — read this first
└── README.md
```

## Conventions

See `docs/decisions.md` for naming and tagging standards (ADR-001, ADR-002).

## How to deploy

Phase 1 deployment commands will be added once `main.bicep` is authored.

```bash
# Preview changes
az deployment sub what-if \
  --location westeurope \
  --template-file bicep/main.bicep \
  --parameters bicep/parameters/dev.parameters.json

# Apply
az deployment sub create \
  --location westeurope \
  --template-file bicep/main.bicep \
  --parameters bicep/parameters/dev.parameters.json
```

## Status

Phase 1 — in progress.
