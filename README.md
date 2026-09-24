# Azure Healthcare Security Landing Zone

### Azure Cloud Security Architecture | Governance | Zero Trust | SOC 2 & HIPAA Readiness

This project demonstrates the design, implementation, and validation of a security-focused Microsoft Azure landing zone modeled for a healthcare environment.

The goal was not simply to deploy Azure resources, but to build security controls around identity, networking, governance, encryption, monitoring, and data protection, then validate that those controls actually worked.

During the project, I tested policy enforcement, investigated governance failures, corrected scope and compatibility issues, generated administrative activity, and used KQL to validate centralized security logging.

> **Project Focus:** Cloud Security Engineering | Azure Governance | Zero Trust | Security Monitoring | Compliance Readiness

---

## Architecture

<img width="1536" height="1024" alt="ChatGPT Image Sep 22, 2026, 10_41_31 AM" src="https://github.com/user-attachments/assets/c236df24-dc08-48ef-8c06-cf390d128951" />
The environment uses a hub-spoke network architecture with a segmented Development workload.

The spoke network was divided into Web, Application, and Data tiers to model a three-tier healthcare workload.

| Tier | Subnet | Security Purpose |
|---|---|---|
| Web | `snet-web` | Entry tier for web workloads |
| Application | `snet-app` | Internal application services |
| Data | `snet-data` | Protected data services and Private Endpoints |

Traffic between tiers is restricted using dedicated **Network Security Groups (NSGs)**.

### Intended Application Flow

```text
Web
 │ HTTPS 443
 ▼
Application
 │ SQL 1433
 ▼
Data
```

Sensitive Azure PaaS services use **Private Link and Private DNS** rather than public service access.

---

## Security Controls Implemented

### Azure Governance

A Management Group hierarchy was created to provide centralized governance across workload environments.

```text
Landing-Zones
├── Development
└── Production
```

Azure Policy was assigned at the **Landing-Zones Management Group** so governance controls could inherit across environments.

Implemented controls included:

- Approved Azure deployment regions
- Mandatory `Environment` tagging
- Policy inheritance through Management Groups
- RBAC separation between workload and security responsibilities

Deployments were restricted to:

- `East US`
- `East US 2`

Resources deployed outside approved regions were denied.

---

## Governance Testing & Remediation

One of the most important outcomes of the project came from a **real policy failure**.

The original mandatory-tagging policy was too broad and interfered with Azure-managed dependencies created by Private Link, including Private DNS resources.

Rather than removing the security control, I redesigned it using an **Indexed custom policy** so tagging would continue to be enforced against supported workload resources without preventing Azure-managed dependencies from being created.

A second issue was discovered during compliance review.

The custom policy assignment had accidentally been applied at the **subscription scope** instead of the intended **Landing-Zones Management Group**. This caused resources belonging to unrelated security labs within the subscription to appear noncompliant.

The assignment was recreated at the correct Management Group scope and the overly broad subscription-level assignment was removed.

### Engineering Lesson

> Security controls must be validated for enforcement, scope, inheritance, compatibility, and unintended operational impact.

---

## Network Security

The Development spoke uses **tier-based network segmentation**.

### Web Tier

- Dedicated Network Security Group
- HTTPS/443 permitted for the modeled web workload
- Segmented from internal application and data tiers

### Application Tier

- HTTPS/443 permitted from the Web subnet
- Other VNet inbound traffic explicitly denied
- Dedicated Network Security Group

### Data Tier

- SQL/1433 permitted from the Application subnet
- Other VNet inbound traffic explicitly denied
- Private Endpoints deployed for sensitive Azure services

### Segmented Traffic Model

```text
Internet
   │
   ▼
Web Tier
snet-web
   │
   │ HTTPS 443
   ▼
Application Tier
snet-app
   │
   │ SQL 1433
   ▼
Data Tier
snet-data
```

The architecture models a controlled three-tier workload rather than allowing unrestricted east-west communication between subnets.

---

## Private Connectivity

Public exposure of sensitive data services was reduced using **Azure Private Link**.

Implemented controls included:

- Storage Account Private Endpoint
- Key Vault Private Endpoint
- Azure Private DNS
- Public network access disabled for Storage
- Private Endpoint placement within the protected Data subnet

The Storage Account Private Endpoint resolved internally to:

```text
10.1.3.7
```

This demonstrated private service access without requiring the Storage Account to remain publicly reachable.

---

## Encryption & Key Management

Sensitive Storage data was protected using a **Customer-Managed Key (CMK)**.

```text
Azure Storage
      │
      ▼
User-Assigned Managed Identity
      │
      ▼
Azure Key Vault
      │
      ▼
Customer-Managed Encryption Key
```

Implemented controls included:

- Customer-managed RSA encryption key
- User-assigned managed identity
- Azure RBAC authorization
- Key Vault soft delete
- Key Vault purge protection
- Automatic use of the latest key version
- TLS 1.2
- Secure transfer required
- Anonymous Blob access disabled

The managed identity was granted only the permissions required for Storage encryption operations against the Key Vault.

This architecture avoids embedding encryption credentials directly within workloads.

---

## Data Protection & Recovery

Azure Storage was configured with additional data-protection controls:

- Blob soft delete — **7 days**
- Container soft delete — **7 days**
- Blob versioning
- Change Feed logging
- Secure transfer required
- Anonymous Blob access disabled

These controls provide recovery and investigation capabilities for accidental or malicious data modification and deletion.

For a production healthcare environment, backup retention, RPO/RTO requirements, and point-in-time recovery would be determined through formal business impact and recovery requirements.

---

## Centralized Security Monitoring

Security telemetry was centralized in an Azure Log Analytics workspace.

```text
law-security-dev-eastus2
```

Log sources included:

- Azure Subscription Activity Logs
- Key Vault diagnostic logs
- Storage diagnostic logs

I did not treat configuration alone as proof that logging worked.

Administrative activity was deliberately generated and then queried from the centralized workspace to validate **end-to-end telemetry ingestion**.

### Logging Validation

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
```

The query confirmed that administrative activity was successfully ingested and contained:

- Initiating identity
- Operation performed
- Affected resource
- Timestamp
- Operation result

This provided an auditable trail for security investigations and compliance monitoring.

---

## Security Control Change Detection

After validating ingestion, I developed a **KQL security detection** to identify administrative changes to critical Azure security controls.

```kusto
AzureActivity
| where TimeGenerated > ago(24h)
| where OperationNameValue has_any (
    "POLICYASSIGNMENTS",
    "ROLEASSIGNMENTS",
    "NETWORKSECURITYGROUPS",
    "VAULTS"
)
| where OperationNameValue has_any (
    "WRITE",
    "DELETE"
)
| project
    TimeGenerated,
    OperationNameValue,
    ActivityStatusValue,
    Caller,
    ResourceGroup,
    ResourceId,
    CallerIpAddress
| order by TimeGenerated desc
```

The detection was validated against an **Azure Policy assignment deletion** and successfully captured the:

- Initiating identity
- Administrative operation
- Affected Azure resource
- Timestamp
- Source IP address
- Operation result

This provides a starting point for detecting unauthorized or unexpected changes to critical cloud security controls.

---

## Cloud Security Posture Management

**Microsoft Defender for Cloud Foundational CSPM** was enabled for the Azure subscription.

The project used CSPM concepts to evaluate Azure configuration posture and identify areas requiring additional controls in a production environment.

The free Foundational CSPM capability was used for the cost-controlled lab environment rather than enabling unnecessary paid workload protection plans.

> A lack of reported recommendations was not treated as proof that the environment contained zero vulnerabilities. Assessment coverage and workload visibility must always be considered when interpreting CSPM results.

---

## SOC 2 & HIPAA Security Readiness Assessment

After completing the technical build, I performed a security readiness review against relevant:

- **SOC 2 Trust Services Criteria**
- **HIPAA Security Rule safeguards**

The assessment mapped implemented technical controls to security objectives and documented:

- Implemented controls
- Validation evidence
- Identified security gaps
- Residual risks
- Recommended production improvements

> **Important:** This was a technical security readiness assessment performed as part of a portfolio lab. It is not a SOC 2 attestation, HIPAA certification, legal opinion, or independent compliance determination.

---

## Key Production Improvements Identified

The assessment identified several controls that would require additional maturity in a production healthcare environment:

- Just-in-time privileged access using PIM
- Periodic privileged access reviews
- Continuous vulnerability scanning
- Risk-based vulnerability prioritization
- Formal vulnerability remediation SLAs
- Centralized ingress and egress inspection
- Azure Firewall or equivalent network security controls
- Formal backup and recovery requirements
- Incident response escalation procedures
- Continuous compliance monitoring
- Stronger restrictions around Storage Shared Key access

These were documented as **known gaps and production-hardening recommendations** rather than enabling expensive services simply to make the lab appear complete.

---

## Evidence-Driven Validation

A major objective of this project was proving that security controls worked rather than only showing configuration screens.

| Security Control | Validation Performed |
|---|---|
| Azure Policy | Unauthorized deployment denied |
| Policy Scope | Incorrect subscription scope discovered and corrected |
| Regional Governance | Deployment restricted to approved Azure regions |
| Network Segmentation | Web/App/Data NSG rules validated |
| Private Link | Storage resolved to private IP |
| CMK Encryption | Storage successfully configured with Key Vault-managed key |
| Managed Identity | Identity granted scoped Key Vault encryption permissions |
| Activity Logging | Administrative change located in centralized Log Analytics |
| KQL Detection | Policy deletion successfully detected |
| Data Recovery | Soft delete and versioning controls verified |
| CSPM | Defender for Cloud Foundational monitoring enabled |

---

## Security Engineering Lessons Learned

### Controls Can Create Operational Problems

A security control that prevents required platform dependencies from functioning should be redesigned rather than blindly enforced.

The tagging policy issue demonstrated the importance of understanding how Azure-managed resources interact with governance controls.

### Scope Matters

A correctly written Azure Policy applied at the wrong scope can still create incorrect compliance results or unintended operational impact.

### Configuration Is Not Validation

Enabling diagnostic settings does not prove telemetry is reaching the monitoring platform.

I generated administrative activity and queried the centralized workspace to validate the complete logging path.

### Private Connectivity Introduces Dependencies

Private Endpoints depend on correct:

- DNS configuration
- VNet connectivity
- Subnet configuration
- Network security rules
- Private DNS zone links

### Compliance Requires Evidence

Security controls should be tied to evidence demonstrating that they exist and operate as intended.

---

## Incident Investigation Workflow

The centralized telemetry and KQL detections were designed around a basic cloud incident investigation workflow:

```text
Detect
  ↓
Triage
  ↓
Investigate
  ↓
Contain
  ↓
Eradicate
  ↓
Recover
  ↓
Lessons Learned
```

For example, an unexpected Azure Policy or RBAC modification would initiate investigation of:

1. The identity that performed the action
2. Source IP and expected location
3. Entra ID authentication activity
4. MFA and Conditional Access results
5. Other privileged activity performed by the identity
6. Additional resources affected during the same timeframe

This connects cloud configuration monitoring with practical incident response.

---

## Technology Used

### Cloud & Identity

- Microsoft Azure
- Microsoft Entra ID
- Azure RBAC
- Managed Identities

### Network Security

- Azure Virtual Network
- Hub-Spoke Architecture
- VNet Peering
- Network Security Groups
- Azure Private Link
- Azure Private DNS

### Data Security

- Azure Storage
- Azure Key Vault
- Customer-Managed Keys
- TLS 1.2
- Blob Versioning
- Soft Delete

### Governance

- Azure Management Groups
- Azure Policy
- Custom Policy Definitions
- Resource Tagging

### Monitoring & Detection

- Azure Monitor
- Log Analytics
- Microsoft Defender for Cloud
- Kusto Query Language (KQL)

### Security & Compliance

- Zero Trust principles
- SOC 2 Trust Services Criteria
- HIPAA Security Rule safeguards

---

## Project Outcome

The completed environment demonstrated an end-to-end Azure security architecture covering:

```text
Governance
    ↓
Identity & Access
    ↓
Network Segmentation
    ↓
Private Connectivity
    ↓
Encryption & Key Management
    ↓
Centralized Monitoring
    ↓
Detection Engineering
    ↓
Control Validation
    ↓
Compliance Readiness
```

The environment was **decommissioned after validation and evidence collection** to avoid unnecessary cloud costs.

The next iteration of this work will focus on taking these cloud security concepts and implementing them through **Infrastructure as Code, CI/CD, DevSecOps security controls, and automated cloud deployment**.

---

## Repository Contents

```text
Azure-Healthcare-Security-Landing-Zone/
│
├── README.md
│
├── docs/
│   ├── architecture/
│   └── SOC2-HIPAA-Security-Readiness-Assessment.pdf
│
├── evidence/
│   ├── governance/
│   ├── identity/
│   ├── networking/
│   ├── data-protection/
│   └── monitoring/
│
├── policy/
│   └── require-environment-tag.json
│
└── detections/
    └── security-control-change-detection.kql
```

---

## Author

**Harrison Knapp**

Cloud Security | Azure Security | Detection Engineering

---

## Disclaimer

This project was created as a personal cloud security engineering lab using synthetic resources and data.

The SOC 2 and HIPAA portions represent a technical security readiness exercise and do not constitute a SOC 2 attestation, HIPAA certification, legal opinion, or independent compliance assessment.
