# Azure Healthcare Security Landing Zone

> Enterprise Cloud Security Architecture, Governance & Compliance Engineering

A healthcare-focused Azure landing zone designed to demonstrate how security,
governance, private connectivity, encryption, monitoring, and compliance
readiness can be engineered into a cloud environment from the beginning.

This project goes beyond deploying Azure resources. Security controls were
tested, failures were investigated, governance scope was corrected, centralized
logging was validated with KQL, and the completed environment was assessed
against relevant SOC 2 Trust Services Criteria and HIPAA Security Rule safeguards.

---

## Project Objectives

The landing zone was designed around five objectives:

- Establish scalable Azure governance using Management Groups and Azure Policy.
- Enforce least privilege and separation of security/workload responsibilities.
- Isolate healthcare workloads using hub-spoke networking and tiered segmentation.
- Protect sensitive data using Private Link, Key Vault, and customer-managed encryption.
- Produce defensible security evidence through centralized logging, KQL validation,
  control testing, and a formal SOC 2/HIPAA readiness assessment.

## Architecture

The environment follows a hub-spoke model with a segmented Development workload.

<img width="1536" height="1024" alt="ChatGPT Image Sep 22, 2026, 10_41_31 AM" src="https://github.com/user-attachments/assets/c236df24-dc08-48ef-8c06-cf390d128951" />


### Governance Layer

**Landing-Zones Management Group**
- Development
- Production
- Azure Policy inheritance
- RBAC inheritance

### Development Workload

**Spoke VNet — `vnet-spoke-dev-eastus2`**

| Tier | Subnet | Purpose |
|---|---|---|
| Web | `snet-web` | Internet-facing application tier |
| Application | `snet-app` | Internal application services |
| Data | `snet-data` | Protected data services / Private Endpoints |

Traffic is restricted using dedicated NSGs.

`Internet → Web → Application → Data`

Lateral traffic that is not explicitly required is denied.

---

## Security Architecture

### Network Segmentation

The Development spoke implements tier-based segmentation.

**Web**
- HTTPS/443 permitted from Internet
- Other unsolicited traffic denied

**Application**
- HTTPS/443 permitted from Web subnet
- Other VNet inbound traffic explicitly denied

**Data**
- SQL/1433 permitted from Application subnet
- Other VNet inbound traffic explicitly denied

Azure Private Link is used for sensitive platform services, reducing exposure
to public service endpoints.

### Private Data Access

The Storage Account and Key Vault use Private Endpoints with Azure Private DNS.

The Storage private endpoint resolves internally to:

`10.1.3.7`

Public network access to the Storage Account is disabled.

### Encryption & Key Management

Sensitive storage uses a customer-managed key architecture:

`Storage Account → Managed Identity → Key Vault → CMK`

Implemented controls include:

- Customer-managed RSA key
- Azure RBAC authorization
- User-assigned managed identity
- Key Vault purge protection
- Key Vault soft delete
- Automatic use of latest key version
- TLS 1.2
- Secure transfer required
- Anonymous Blob access disabled

---

## Governance Engineering

Azure Policy is assigned at the Landing-Zones Management Group so controls
inherit across workload environments.

Implemented policies include:

### Allowed Azure Regions

Deployments are restricted to:

- East US
- East US 2

Non-approved locations are denied.

### Mandatory Environment Tag

Workload resources must contain an `Environment` tag.

The control was validated by attempting an untagged deployment and confirming
Azure returned `RequestDisallowedByPolicy`.

### A Real Governance Failure

The original tag policy was too broad.

It interfered with Azure-managed Private Link dependencies such as Private DNS
resources. A custom `Indexed` policy was created to enforce workload tagging
without incorrectly evaluating unsupported resource types.

A second issue was later discovered: the custom assignment had accidentally
been applied at subscription scope. This caused unrelated security-lab
resources to appear noncompliant.

The assignment was recreated at the intended `Landing-Zones` Management Group
scope and the subscription-level assignment was removed.

**Engineering lesson:** governance controls must be tested not only for
enforcement, but also for scope, inheritance, compatibility, and unintended impact.

---

## Centralized Security Monitoring

Security telemetry is centralized in:

`law-security-dev-eastus2`

Sources include:

- Azure Subscription Activity Logs
- Key Vault diagnostic logs
- Storage diagnostic logs

Configuration alone was not treated as sufficient evidence.

An administrative Azure Policy change was generated and subsequently queried
from Log Analytics using KQL.

```kusto
AzureActivity
| where TimeGenerated > ago(6h)
| project
    TimeGenerated,
    OperationNameValue,
    ActivityStatusValue,
    Caller,
    ResourceGroup,
    ResourceId
| order by TimeGenerated desc
