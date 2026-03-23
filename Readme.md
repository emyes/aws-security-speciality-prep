# AWS Security Specialty (SCS-C03) — COMPLETE CHEAT SHEET

## 📊 EXAM OVERVIEW
- **Exam Code:** SCS-C03
- **Duration:** 170 minutes
- **Questions:** 65
- **Passing Score:** 750/1000
- **Domain Distribution:** IAM (25%) | Infrastructure (26%) | Logging (20%) | Data Protection (22%) | Incident Response (7%)

---

# 📋 IN-SCOPE AWS SERVICES & FEATURES

> **Important:** Security affects ALL AWS services. Services not listed here may be out of scope overall, but their **security aspects remain in scope**. Example: S3 replication setup = out of scope; S3 bucket policy configuration = **in scope**.

## Analytics
| Service | Security Relevance |
|---------|--------------------|
| Amazon Athena | Query CloudTrail/VPC Flow Logs in S3; enforce fine-grained access via IAM |
| Amazon OpenSearch Service | Access policies, encryption at rest/transit, fine-grained access control |

## Application Integration
| Service | Security Relevance |
|---------|--------------------|
| Amazon SNS | Topic policies, message encryption, cross-account delivery |
| AWS Step Functions | IAM execution roles, state machine security, secrets handling |

## Compute
| Service | Security Relevance |
|---------|--------------------|
| Amazon API Gateway | Resource policies, authorizers, WAF integration, mutual TLS |
| Amazon EC2 | SGs, NACLs, IMDSv2, IAM instance profiles, EC2 Image Builder hardening, EC2 Instance Connect |
| Amazon EKS | IRSA, RBAC, network policies, secrets encryption, pod security |
| Amazon EMR | Kerberos, encryption at rest/transit, Lake Formation integration |
| AWS Lambda | Execution roles, VPC isolation, environment variable encryption, resource policies |
| Amazon Data Lifecycle Manager | Automate EBS snapshot lifecycle; enforce backup compliance |

## Developer Tools
| Service | Security Relevance |
|---------|--------------------|
| AWS Fault Injection Service | IAM permissions for chaos experiments; blast-radius control |

## Internet of Things

### AWS IoT Core
**What it is:** Managed cloud service that lets IoT devices securely connect, communicate, and exchange messages with cloud applications over MQTT, HTTPS, and WebSockets.

#### 🔐 Device Authentication
- **X.509 Certificates** — Primary method. Each device holds a unique certificate. AWS IoT acts as the CA or you can bring your own CA (BYOCA). Mutual TLS (mTLS) authenticates both device and broker at connection time.
- **Custom Authorizers** — For devices that cannot use X.509 (e.g., legacy HTTP devices), a Lambda-backed custom authorizer validates tokens or HTTP headers and returns an IAM-style policy.
- **Amazon Cognito Identity Pools** — For mobile/browser clients connecting to IoT through an authenticated app user identity.
- **IAM roles** — Used by backend applications and services calling IoT control-plane APIs (not by devices themselves).

#### 📜 IoT Policies
- Attached to **certificates** (not IAM identities). JSON format similar to IAM policies.
- Control actions: `iot:Connect`, `iot:Publish`, `iot:Subscribe`, `iot:Receive`.
- **Least privilege:** specify exact topic ARNs a device may publish/subscribe to — never use `*`.
- **Policy variables** like `${iot:Connection.Thing.ThingName}` enable dynamic per-device topic isolation so one device cannot impersonate or access another's topic.

```json
{
  "Effect": "Allow",
  "Action": "iot:Publish",
  "Resource": "arn:aws:iot:us-east-1:123456789012:topic/devices/${iot:Connection.Thing.ThingName}/telemetry"
}
```

#### 🌐 Transport Security
- All MQTT, HTTPS, and WSS connections require **TLS 1.2+** — plaintext connections are not accepted.
- Broker endpoint: `<account-prefix>.iot.<region>.amazonaws.com` on port **8883** (MQTT) or **443** (HTTPS/WSS).

#### 🛡️ AWS IoT Device Defender *(Runtime Security Monitoring)*
- **Audit mode:** Scans IoT configurations for policy weaknesses — overly permissive policies, revoked certificates still active, certificates shared across multiple devices, logging not enabled.
- **Detect mode:** Behavioral anomaly detection. Flags devices deviating from a learned baseline — unusual message volume, new source IPs, unexpected open ports, abnormal connection frequency.
- **Mitigation Actions:** Automatically quarantine a compromised device by updating its certificate, adding a DENY policy, or moving it to an isolated Thing Group.

#### 🔒 AWS IoT Greengrass *(Edge Security)*
- Extends AWS IoT to edge devices, running Lambda functions and ML models locally.
- Edge-to-cloud communication uses TLS and X.509 certificates — same trust model as IoT Core.
- **Secrets Manager integration:** Rotate secrets centrally in AWS and automatically push updated secrets to edge devices without manual intervention.
- **IAM execution role:** The Greengrass core device assumes an IAM role that defines what AWS services it can interact with (S3, CloudWatch, etc.).

#### 📋 Logging & Auditing
| Log Type | Where | What It Captures |
|----------|-------|-----------------|
| IoT Core logging | CloudWatch Logs | MQTT connections, publish/subscribe actions, rule engine executions |
| Control-plane APIs | CloudTrail | CreateThing, AttachPolicy, RegisterCertificate, DeleteCertificate |
| Device Defender findings | CloudWatch + Security Hub | Audit violations, detected anomalies |

#### ⚡ Exam Key Points
| Scenario | Answer |
|----------|--------|
| Device auth without certificates | Custom Authorizer (Lambda-backed) |
| Prevent one device accessing another's topic | IoT policy with `${iot:Connection.Thing.ThingName}` variable |
| Detect compromised IoT device behavior | IoT Device Defender → Detect mode |
| Audit for misconfigured IoT policies | IoT Device Defender → Audit mode |
| Push rotated secrets to edge devices | Secrets Manager + Greengrass |
| Are all device comms encrypted? | Yes — TLS enforced, cannot be disabled |
| Quarantine misbehaving device automatically | Device Defender Mitigation Actions |

---

## Machine Learning

> **Exam note:** ML services are increasingly in scope for the SCS-C03. Focus on **data isolation, IAM access control to models, VPC deployment, preventing prompt injection, and avoiding data leakage from AI outputs**.

---

### Amazon Bedrock
**What it is:** Fully managed service for accessing and customizing foundation models (FMs) from Amazon (Titan, Nova) and third-party providers (Anthropic Claude, Meta Llama, Cohere, Mistral) via a single API — no infrastructure to manage.

#### 🔐 Access Control
- **IAM policies** govern which principals can invoke which models.
  - Key action: `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`
  - Scope to specific model ARN (e.g., `arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3`) — never grant `bedrock:*`.
- **Resource-based policies** on custom/fine-tuned models restrict which accounts or roles can use them.
- **AWS IAM Identity Center** federates corporate users to Bedrock-powered applications without managing separate credentials.

#### 🌐 Network Isolation
- Use a **VPC Interface Endpoint** for Bedrock so all model API calls stay on the AWS private network — no traffic over the public internet.
- Combine with **endpoint policies** to restrict which models can be invoked from the VPC.

#### 🛡️ Data Privacy & Isolation
- AWS does **not** use customer prompts or completions to train or improve base foundation models.
- **Fine-tuning training data** is isolated per customer, stored encrypted in your own S3 bucket, processed in an isolated environment, and deleted after the job completes.
- **Encryption:** All model customization job data is encrypted at rest with KMS (customer CMK strongly recommended for regulated workloads).

#### 📜 Bedrock Guardrails *(AI Safety & Compliance Layer)*
Guardrails are applied at the API layer **before prompts reach the model** and **before responses reach the user** — the model never even sees blocked content.

| Guardrail Feature | Purpose |
|-------------------|---------|
| **Content filters** | Block harmful content (hate, violence, sexual) with configurable thresholds |
| **PII detection & redaction** | Detect and mask PII (names, emails, SSNs, phone numbers) in both inputs and outputs |
| **Denied topics** | Define subject areas the model must refuse to discuss (e.g., competitor pricing, legal advice) |
| **Word filters** | Block specific terms or profanity |
| **Grounding checks** | Detect hallucinations by comparing responses to source documents (RAG use cases) |

#### 📋 Logging & Auditing
- **CloudTrail:** Logs all Bedrock API calls — InvokeModel, CreateModelCustomizationJob, DeleteGuardrail, etc.
- **Model invocation logging:** Optionally log full prompt + response payloads to **S3 or CloudWatch Logs**. *Disabled by default* — explicitly enable for compliance/audit requirements.
- **CloudWatch Metrics:** Token usage, latency, throttle events, guardrail trigger counts per guardrail.

#### ⚡ Exam Key Points
| Scenario | Answer |
|----------|--------|
| Restrict team A to only Claude, team B to only Titan | IAM policy scoped to specific model ARN per principal |
| Keep all model API calls off the internet | Bedrock VPC Interface Endpoint |
| Log every prompt and response for compliance audit | Enable model invocation logging → S3/CloudWatch |
| Filter harmful or inappropriate AI responses | Bedrock Guardrails — content filtering |
| Stop PII leaking from AI responses | Bedrock Guardrails — PII detection + redaction |
| Fine-tune model with proprietary data securely | Customer S3 + CMK encryption + IAM execution role scoped to that bucket |

---

### Amazon CodeGuru Security
**What it is:** AI-powered **Static Application Security Testing (SAST)** tool that scans source code repositories to find security vulnerabilities before they reach production.

#### 🔍 What It Detects (OWASP Top 10 Coverage)
| Vulnerability | How It's Detected |
|--------------|-------------------|
| **Hardcoded secrets** | API keys, passwords, tokens, connection strings embedded in code |
| **SQL Injection** | Unsanitized user input passed directly to DB queries |
| **SSRF** | Unsafe URL construction from user-controlled input |
| **XSS** | Unescaped output rendered in HTML/JS contexts |
| **Insecure cryptography** | MD5/SHA-1/DES usage, predictable random generators |
| **Insecure deserialization** | Unsafe object deserialization patterns |
| **Path traversal** | Directory traversal via unsanitized file paths |
| **Command injection** | User input passed to OS shell commands |

#### 🔗 Integrations
- **Source repositories:** GitHub, GitHub Enterprise, GitLab, Bitbucket, AWS CodeCommit, S3 (ZIP upload).
- **CI/CD Pipeline:** Integrates with **CodePipeline** — configure a gate to block deployments when Critical or High findings exist.
- **IDE:** Plugin for VS Code and JetBrains shows findings inline as you code.
- **Security Hub:** Findings published in ASFF format for centralized security posture management.

#### 📊 Findings
- Severity: **Critical → High → Medium → Low → Informational**.
- Each finding includes: file path, line number, CWE ID mapping, description, and suggested code fix.
- **Suppression:** Mark false positives as suppressed; suppression activity is tracked in an audit trail.

#### 🔐 Security of the Service Itself
- Code is transmitted over TLS; CodeGuru does **not retain** your source code after scanning completes.
- IAM actions: `codeguru-security:CreateScan`, `codeguru-security:GetFindings`, `codeguru-security:ListFindings`.

#### ⚡ Exam Key Points
| Scenario | Answer |
|----------|--------|
| Scan codebase for hardcoded secrets before deployment | CodeGuru Security |
| Block CI/CD pipeline if critical vulnerabilities found | CodeGuru Security + CodePipeline approval gate |
| OWASP Top 10 coverage in source code | CodeGuru Security (SAST) |
| Runtime threat detection (not code scanning) | Amazon Inspector (EC2/Lambda/container runtime) or GuardDuty |
| Difference from Inspector | CodeGuru = source code; Inspector = deployed runtime artifacts (packages, AMIs) |

---

### Amazon Q Business
**What it is:** Generative AI enterprise assistant — employees ask natural language questions and receive answers grounded in your organisation's internal data (documents, wikis, ticketing systems, databases, etc.).

#### 🔐 Identity & Access
- **Authentication is mandatory via IAM Identity Center (SSO)** — anonymous or unauthenticated access is not supported. All users authenticate through your corporate IdP (Okta, Azure AD, etc.) federated through IAM Identity Center.
- **User-level document permissions:** Q Business respects the **native ACLs of each data source**. If a user doesn't have access to a document in SharePoint or Confluence, Q Business will not surface content from that document in answers — even if the document is indexed.
- **Admin controls:** Define which data sources are available, which groups can access which applications, and which topics the assistant can or cannot discuss.

#### 📂 Data Source Security
- Each data source connector (S3, Confluence, SharePoint, Salesforce, Jira, etc.) uses a **dedicated IAM role** with least-privilege read-only access to the required data source.
- Data source credentials (OAuth tokens, API keys) are stored and auto-rotated in **Secrets Manager**.
- All indexed content is **encrypted at rest with KMS** — customer CMK is supported for regulated environments.
- Data is stored in an isolated, per-customer index with no cross-customer data access.

#### 🛡️ Guardrails & Topic Controls
| Control Type | Description |
|-------------|-------------|
| **Blocked topics** | Define topics the assistant must refuse to answer (legal advice, competitor comparisons, HR policies) |
| **Response scope** | Restrict answers to only indexed internal data — prevents the model from answering using general internet knowledge |
| **Global controls** | Apply rules across all conversations in the application |
| **Topic-level controls** | Fine-grained rules per subject area |

#### 📋 Logging & Auditing
- **CloudTrail:** Logs all Q Business API calls (CreateApplication, UpdateRetriever, PutFeedback, BatchDeleteDocument).
- **Conversation logs:** Optionally configure to write conversation history to **CloudWatch Logs** for compliance review and audit.

#### ⚡ Exam Key Points
| Scenario | Answer |
|----------|--------|
| Ensure users only see Q responses based on docs they're authorised for | Q Business + IAM Identity Center + data source ACL sync |
| Secure the credentials used by Q to read SharePoint | Secrets Manager (auto-rotated) |
| Prevent Q Business from discussing HR disciplinary matters | Q Business Admin Controls → blocked topics |
| Audit who queried the AI assistant | CloudTrail + CloudWatch conversation logs |
| Q Business vs Bedrock | Q Business = enterprise Q&A grounded in *your internal data*; Bedrock = general FM API for custom AI apps |

---

### Amazon Q Developer
**What it is:** AI coding assistant integrated into IDEs (VS Code, JetBrains, Visual Studio) and the AWS Console that suggests code, explains existing code, generates tests, and scans for security issues.

#### 🔐 Access & Identity
- **Enterprise tier (recommended for organisations):** Authenticate via **IAM Identity Center**. AWS does not use enterprise customer code to train models. Governed by your organization's SSO policies.
- **Individual/Free tier (Builder ID):** Public identity provider. Code may be used to improve the model — not suitable for proprietary or sensitive codebases.
- Enforce enterprise tier via IAM Identity Center to ensure code privacy compliance.

#### 🔍 Security Scanning (`/review` command)
- On-demand vulnerability scan of local or repository code.
- Detects the same vulnerability classes as CodeGuru Security: hardcoded credentials, injection issues, insecure crypto, OWASP Top 10.
- Findings include file path, line, severity, and a plain-English explanation with suggested code fix.
- **Automated scans:** Can be integrated into a project to scan on every save or commit.

#### 🛡️ Code Privacy
- All code transmitted to AWS is encrypted over TLS.
- **Enterprise tier:** Code is not stored, logged, or used for model training post-response.
- **Individual tier:** Code submissions may be retained — avoid using with proprietary or sensitive IP.

#### ⚡ Exam Key Points
| Scenario | Answer |
|----------|--------|
| Developer tool that finds security vulnerabilities in the IDE | Amazon Q Developer (`/review` security scan) |
| Prevent developer code from being used to train AWS models | Use Q Developer Enterprise tier with IAM Identity Center |
| Q Developer vs CodeGuru Security | Q Developer = interactive IDE assistant + inline scan; CodeGuru = dedicated SAST tool for pipeline integration |
| Q Developer vs Q Business | Q Developer = coding assistant for developers; Q Business = enterprise Q&A for all employees |

---

### Amazon SageMaker AI
**What it is:** Comprehensive end-to-end ML platform for building, training, tuning, and deploying machine learning models at scale. Security is critical because models process sensitive training data and serve production traffic.

#### 🌐 Network Isolation
- **VPC mode:** Training jobs, processing jobs, and notebook instances run inside a **private VPC** — restricts internet access and AWS service traffic to defined routes only.
- **`EnableNetworkIsolation=true`:** Completely blocks all outbound network traffic from the container (no internet, no AWS services) — use for untrusted training scripts or third-party model images.
- **VPC endpoints:** Access S3 (via Gateway endpoint), ECR, CloudWatch, STS, and SageMaker APIs **privately** without internet routing.
- **SageMaker Studio VPC-only mode:** Users access Studio via VPN, Direct Connect, or bastion — no public internet access to the Studio domain.
- **Inter-container traffic encryption:** Enable TLS encryption for communication between containers in **distributed training jobs** to prevent in-transit data capture on the cluster network.

#### 🔐 IAM & Access Control
- **Execution roles:** Every notebook instance, training job, processing job, and endpoint is associated with a **dedicated IAM execution role**. Apply least privilege — grant only the specific S3 bucket, ECR repo, and KMS key actually needed.
- **SageMaker Studio user profiles:** Each user gets an isolated execution role. Multiple users can share a Studio domain but have separate permission boundaries.
- **Presigned URLs for notebook access:** Access to notebook instances is granted via short-lived presigned URLs — generate and revoke them; all access logged in CloudTrail.
- **Model registry cross-account access:** Use resource-based policies to allow other accounts to deploy approved models from a central registry.

#### 🔒 Encryption Summary
| Asset | Encryption |
|-------|------------|
| Model artifacts stored in S3 | SSE-KMS with CMK |
| EBS volumes (notebook/training instances) | KMS CMK |
| Inter-container training traffic | TLS (must be explicitly enabled) |
| Endpoint HTTPS inference traffic | TLS (enforced by default) |
| Feature Store online store | KMS CMK |
| Feature Store offline store (S3) | SSE-KMS with CMK |

#### 🛡️ Secrets & Credentials Best Practices
- **Never hardcode credentials** in notebook cells or training scripts — use the execution role's IAM permissions instead.
- For external system credentials (database passwords, API keys): store in **Secrets Manager** and retrieve at runtime via the SDK (`boto3.client('secretsmanager').get_secret_value(...)`).
- Avoid storing secrets in environment variables on the training job definition — they appear in CloudTrail logs.

#### 📋 Logging & Monitoring
| Log Type | Where | What It Captures |
|----------|-------|-----------------|
| Control-plane API calls | CloudTrail | CreateTrainingJob, CreateEndpoint, DeleteModel, etc. |
| Training job output | CloudWatch Logs | Container stdout/stderr during training |
| Endpoint invocation logs | CloudWatch Logs | Request/response metadata, latency |
| Model quality metrics | CloudWatch Metrics | Endpoint invocation count, latency, error rate |
| Data/concept drift | SageMaker Model Monitor | Detects input distribution shift in production (adversarial input detection) |

#### ⚡ Exam Key Points
| Scenario | Answer |
|----------|--------|
| Training job must have no internet access | VPC mode + `EnableNetworkIsolation=true` |
| Encrypt model artifacts and training volumes | SSE-KMS on S3 + KMS on EBS, both using CMK |
| Access SageMaker Studio without public internet | VPC-only domain + VPN/Direct Connect + IAM Identity Center |
| Database credentials needed inside a training script | Retrieve from Secrets Manager at runtime (not env variables) |
| Audit who deployed or deleted a SageMaker endpoint | CloudTrail logs `CreateEndpoint` / `DeleteEndpoint` |
| Encrypt traffic between training container nodes | Enable inter-container traffic encryption |
| Detect when production inference inputs drift from training data | SageMaker Model Monitor |

## Management and Governance
| Service | Security Relevance |
|---------|--------------------|
| AWS CloudFormation | Stack policies, drift detection, IAM role execution, secrets via SSM/Secrets Manager |
| AWS CloudTrail | API audit logging, log file integrity validation, Org trails |
| AWS CloudTrail Lake | SQL-based long-term event analysis and compliance querying |
| Amazon CloudWatch | Metric alarms, log groups, anomaly detection, Contributor Insights |
| AWS Config | Continuous compliance, config rules, remediation actions, conformance packs |
| AWS Control Tower | Landing zone, guardrails (preventive SCPs + detective Config rules), account vending |
| Amazon Managed Grafana | IAM Identity Center SSO, data source permission boundaries |
| AWS Organizations | SCPs, delegated admin, consolidated billing boundary, OU structure |
| AWS Resilience Hub | Resiliency policies, disruption testing scope |
| AWS Resource Access Manager (RAM) | Share resources cross-account without IAM credential sharing |
| AWS Service Catalog | Enforce pre-approved, compliant infrastructure templates |
| AWS Systems Manager | Patch Manager, Session Manager (no SSH), Parameter Store SecureString, Run Command IAM |
| AWS Trusted Advisor | Security checks (MFA on root, open SGs, public snapshots, exposed keys) |
| AWS User Notifications | Security event delivery to notification channels |
| AWS Well-Architected Tool | Security pillar review, risk identification |

## Networking and Content Delivery
| Service | Security Relevance |
|---------|--------------------|
| Amazon Application Recovery Controller | Zonal shift safety; resilience without data exposure |
| Amazon VPC | Isolation, subnets, route tables, flow logs, peering |
| Network Access Analyzer | Identify unintended network access paths |
| Network ACLs | Stateless subnet-level firewall (allow + deny); first-match rule order |
| Security Groups | Stateful ENI-level firewall (allow only); implicit deny |
| VPC Endpoints | Private AWS service access (no internet); Gateway (S3/DynamoDB) + Interface (others) |
| AWS Site-to-Site VPN | Encrypted IPSec tunnel; on-prem to VPC connectivity |
| VPC Flow Logs | Network traffic metadata logging (not packet content) to S3/CW |
| AWS Verified Access | Zero-trust application access without VPN; identity + device posture |
| AWS Client VPN | SSL/TLS VPN for remote users to VPC |
| Amazon CloudFront | HTTPS enforcement, WAF integration, signed URLs/cookies, OAC for S3 |
| Amazon Verified Permissions | Cedar policy-based fine-grained authorization for apps |
| Amazon Route 53 | DNSSEC, DNS Firewall (block malicious domains), query logging, private hosted zones |
| AWS Direct Connect | Dedicated private network connection (not encrypted by default; use MACsec/VPN) |
| Elastic Load Balancing (ELB) | SSL termination, WAF integration, access logs, security policies |
| AWS Transit Gateway | Hub-and-spoke VPC connectivity; centralized inspection via firewall |

## Security, Identity, and Compliance
| Service | Security Relevance |
|---------|--------------------|
| AWS Artifact | On-demand compliance reports (SOC, PCI, ISO) and AWS agreements |
| AWS Audit Manager | Continuous evidence collection for audit frameworks (HIPAA, PCI, GDPR) |
| AWS Certificate Manager (ACM) | Free SSL/TLS certs, auto-renewal, integration with ELB/CloudFront/API GW |
| AWS CloudHSM | Dedicated FIPS 140-2 Level 3 HSM; customer-managed keys; use for CA or custom crypto |
| Amazon Cognito | User pools (AuthN), identity pools (AWS credentials); OIDC/SAML federation |
| Amazon Detective | Visual investigation of security findings using ML graph analysis |
| AWS Directory Service | Managed AD (Microsoft AD or Simple AD); LDAP integration; trust relationships |
| AWS Firewall Manager | Centrally manage WAF, SG, Shield, Network Firewall policies across Org accounts |
| Automated Forensics Orchestrator for EC2 | Automated EC2 forensics workflow; memory/disk acquisition, isolation, snapshot |
| Amazon GuardDuty | ML-based threat detection; ingests CloudTrail, DNS, VPC Flow Logs, S3, EKS |
| AWS IAM Identity Center | Central SSO for multi-account access; permission sets; SCIM provisioning |
| AWS IAM | Policies, roles, users, groups, permission boundaries, STS, Access Analyzer |
| Amazon Inspector | Automated vulnerability assessment for EC2, Lambda, ECR container images |
| AWS KMS | CMKs, data key encryption, key policies + IAM (both needed), automatic rotation |
| Amazon Macie | ML-based S3 sensitive data (PII) discovery and classification |
| AWS Network Firewall | Stateful/stateless deep-packet inspection at VPC perimeter; Suricata rules |
| AWS Private Certificate Authority | Private PKI; issue/manage private SSL certs for internal services |
| AWS Secrets Manager | Auto-rotation of secrets; Lambda-driven; RDS/Redshift/DocumentDB native support |
| AWS Security Hub | Aggregated security findings; ASFF standard; automated checks; cross-account |
| Amazon Security Lake | Centralized OCSF-normalized security data lake; query with Athena or 3rd-party SIEMs |
| AWS STS | Temporary credentials; AssumeRole, AssumeRoleWithSAML, AssumeRoleWithWebIdentity |
| AWS Shield | DDoS protection; Standard (free, L3/L4) vs Advanced (paid, L3/L4/L7 + DRT + cost protection) |
| AWS WAF | Layer 7 firewall; rules for SQLi, XSS, rate limiting; attached to CloudFront/ALB/API GW |

## Storage and Data Management
| Service | Security Relevance |
|---------|--------------------|
| Amazon S3 | Bucket policies, Block Public Access, SSE (S3/KMS/DSSE-KMS), Object Lock (WORM), access logs, MFA delete |
| AWS Backup | Centralized backup with Vault Lock (WORM); cross-account/region backup; compliance |
| AWS DataSync | Encrypted data transfer with integrity checks; IAM-controlled; in-transit encryption |
| Amazon EFS | NFS with IAM auth, KMS encryption at rest, POSIX permissions, EFS Lifecycle policies |
| Amazon FSx for Lustre | High-performance scratch/persistent FS; KMS encryption; VPC-based access control |

---

# 🧠 HOW TO USE THIS CHEATSHEET
1. **Learn** services in each domain
2. **Apply** the decision engine to exam questions
3. **Avoid** 15 common exam traps
4. **Pick** architecture using elimination strategy

---

# 🚨 EXAM DECISION ENGINE (START HERE)

## Step 1 — Identify the Domain
- **Identity/Access** → IAM + STS
- **Network** → SG + NACL + VPC Endpoint + WAF
- **Data** → KMS + S3 + Encryption
- **Detection** → GuardDuty + CloudTrail + Config
- **Automation** → EventBridge + Lambda + Systems Manager

---

## Step 2 — Default Answers (80% accuracy!)

| Problem | Default Answer | Why |
|---------|-----------------|-----|
| Cross-account access | STS AssumeRole | Standard federation |
| Data encryption | SSE-KMS | Customer control |
| Private AWS service | VPC Endpoint | No internet |
| Threat detection | GuardDuty | Real-time ML |
| Central logging | Org CloudTrail | Multi-account view |
| Secrets storage | Secrets Manager | Auto-rotation |
| Compliance checking | AWS Config | Continuous |
| Network isolation | VPC + SG | Stateful firewall |
| Log analysis | CloudWatch Logs Insights | Fast queries |

---

## Step 3 — Eliminate Wrong Answers

❌ **Never pick:**
- IAM users (use roles)
- Hardcoded credentials
- Public access (Block Public Access)
- Manual fixes (automate)
- Over-engineering

---

## Step 4 — Pick Best Architecture
```
Does it:
✅ Use IAM roles? (not users)
✅ Use encryption? (KMS preferred)
✅ Use endpoints? (no internet)
✅ Use automation? (event-driven)
✅ Avoid manual work?

If YES to all = PICK THIS
```

---

# ⚠️ TOP 15 EXAM TRAPS (SCORE BOOSTERS!)

1. **SCP does NOT grant permission** — It's a boundary only
2. **Explicit DENY overrides everything** — Including Allow
3. **KMS needs both key policy + IAM** — Not just one
4. **S3 Block Public Access wins** — Over bucket policy
5. **SG = Allow only** — Implicit deny all else
6. **NACL = Stateless** — Return traffic needs explicit rule
7. **NAT Gateway ≠ Private AWS** — Data still on AWS network
8. **VPC Endpoint = Private AWS** — No internet exposure
9. **GuardDuty ≠ Prevention** — Detection only
10. **AWS Config ≠ Real-time** — Eventually consistent
11. **CloudTrail = API logs** — Not all actions
12. **Macie = S3 only** — Doesn't scan other services
13. **AssumeRole = Cross-account** — Only way to do it
14. **IAM role > IAM user** — Roles for services/apps
15. **SSE-KMS preferred** — Customer-managed encryption

---

# 🎯 EXAM SOLVING FLOWCHART

```
Question asks about...

❓ IDENTITY?
   ↓
   - STS AssumeRole (cross-account)
   - IAM role (not user)
   - Policy evaluation (DENY wins)
   ↓
   PICK: IAM-based answer

❓ DATA?
   ↓
   - Encryption at rest (KMS)
   - Encryption in transit (TLS)
   - Access controls (policies)
   ↓
   PICK: KMS + encryption answer

❓ NETWORK?
   ↓
   - Isolation (SG + NACL)
   - Private access (endpoint)
   - Protection (WAF + Shield)
   ↓
   PICK: Network isolation answer

❓ DETECTION?
   ↓
   - Logs (CloudTrail)
   - Threats (GuardDuty)
   - Compliance (Config)
   ↓
   PICK: Automation answer

❓ INCIDENT RESPONSE?
   ↓
   - Automated fix (Lambda)
   - Restrict access (SG)
   - Revoke credentials (IAM)
   ↓
   PICK: Automated remediation
```

---

# 🔐 IAM DEEP DIVE — THE FULL PICTURE (25% of Exam)

---

## 1. IAM Identities

| Identity | Purpose | When to Use |
|----------|---------|-------------|
| **IAM User** | Long-term credentials (username + password / access keys) | Human admins only when SSO is unavailable |
| **IAM Role** | Temporary credentials assumed by principals | Services, cross-account, federation — **exam default** |
| **IAM Group** | Logical collection of users that share policies | Organize users; cannot be used as a principal in policies |
| **Root User** | Full account access, no restriction | **Used only for account-level tasks** — enable MFA, never use daily |

> **Exam trap:** Questions about applications or AWS services accessing resources → always use IAM **roles**, never IAM users with access keys.

---

## 2. Policy Types — The Six Layers

AWS evaluates **six policy types** in a strict hierarchy. Understanding each is critical for answering "can this principal do X?" questions.

```
Policy Type          Attached To            Controls
─────────────────────────────────────────────────────
1. SCP               AWS Org OU/Account     Maximum allowed actions in an account
2. Permission Bound  IAM User/Role          Maximum allowed actions for that identity
3. Identity-Based    IAM User/Role/Group    What the principal is ALLOWED to do
4. Resource-Based    S3/KMS/SQS/Lambda      Who is ALLOWED to access the resource
5. Session Policy    STS temp session       Further restricts the assumed role session
6. ACL               S3/VPC                 Cross-account object/subnet access (legacy)
```

### 2a. Identity-Based Policies
Attached to **IAM users, roles, or groups**. Define what actions the principal can perform on which resources.

**Two forms:**
- **Managed policies:** Standalone reusable JSON documents.
  - *AWS Managed* — Created by AWS (e.g., `AmazonS3ReadOnlyAccess`). No customization.
  - *Customer Managed* — Created by you. Version-controlled, reusable, *recommended* for fine-grained control.
- **Inline policies:** Embedded directly in one user/role. Deleted when the identity is deleted. Use sparingly — hard to audit.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowS3BucketRead",
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-reports-bucket",
      "arn:aws:s3:::my-reports-bucket/*"
    ],
    "Condition": {
      "StringEquals": { "aws:RequestedRegion": "us-east-1" }
    }
  }]
}
```

**Exam nuance:** Both the bucket ARN (for `ListBucket`) AND the `/*` (for `GetObject`) are required. A common exam trap is missing one of them.

---

### 2b. Resource-Based Policies
Attached directly to an **AWS resource** (S3 bucket, KMS key, SQS queue, Lambda function, ECR repository, Secrets Manager secret, etc.). Define **who** (which principals, including from other accounts) can access the resource.

**Key services supporting resource-based policies:**

| Service | Policy Name | Notes |
|---------|------------|-------|
| S3 | Bucket Policy | Most common; controls access + cross-account |
| KMS | Key Policy | **Required** — KMS ignores IAM unless key policy allows it |
| SQS | Queue Policy | Cross-account message send |
| Lambda | Resource Policy | Define which service/account can invoke the function |
| ECR | Repository Policy | Cross-account image pull |
| Secrets Manager | Resource Policy | Cross-account secret access |
| API Gateway | Resource Policy | IP-based or cross-account restrictions |
| SNS | Topic Policy | Cross-account publish |

**Same-account resource policy:**
- If the resource policy grants access to a **same-account** principal → IAM identity policy is NOT required (resource policy alone is sufficient).
- Exception: **KMS key policies always require both** key policy + IAM policy.

**Cross-account resource policy:**
- The resource policy must explicitly name the **external account or principal**.
- The principal in the external account must also have an **IAM identity policy** allowing the cross-account action.
- Both resource policy AND identity policy must allow the action.

**S3 Bucket Policy example (cross-account):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "CrossAccountRead",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::222222222222:role/DataAnalystRole"
    },
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::central-logs/*",
    "Condition": {
      "StringEquals": { "aws:PrincipalOrgID": "o-xxxxxxxxxxxx" }
    }
  }]
}
```

> **`aws:PrincipalOrgID` Condition** — Ensures only principals *within your AWS Organisation* can access the resource, even if the bucket is shared cross-account. Prevents external AWS accounts from accessing the resource even if they know the bucket name.

---

### 2c. Service Control Policies (SCPs)
Applied at the **AWS Organization** level (to the Root, OUs, or individual accounts). They define the **maximum permissions boundary** — they **cannot grant permissions on their own**, only restrict them.

**Rules:**
- An SCP `Allow` does not grant access — it just removes a restriction.
- An SCP `Deny` **always wins** regardless of IAM policies.
- SCPs apply to every principal in the account **except the root user**.
- The management/master account is **never affected** by SCPs applied to it.

**Common SCP patterns:**

*Deny leaving the Organisation:*
```json
{
  "Effect": "Deny",
  "Action": "organizations:LeaveOrganization",
  "Resource": "*"
}
```

*Deny disabling CloudTrail:*
```json
{
  "Effect": "Deny",
  "Action": [
    "cloudtrail:StopLogging",
    "cloudtrail:DeleteTrail",
    "cloudtrail:UpdateTrail"
  ],
  "Resource": "*"
}
```

*Restrict to approved regions only:*
```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
    }
  }
}
```

**SCP Inheritance:**
```
Root (SCP A)
  └── OU: Production (SCP B)
        └── Account: prod-001 → Effective = A ∩ B ∩ account policies
```
Effective permissions = intersection of all SCPs in the hierarchy.

---

### 2d. Permission Boundaries
An advanced IAM feature that sets the **maximum permissions** an IAM user or role can ever have, regardless of what their identity policies grant.

**Critical concept:** A permission boundary does NOT grant permissions — it only limits them. The effective permissions are the **intersection** of the permission boundary AND the identity policy.

```
Identity Policy:  Allow s3:*, ec2:*
Permission Boundary: Allow s3:GetObject, s3:PutObject

Effective Permissions: s3:GetObject, s3:PutObject
   (ec2:* is silently blocked — the boundary doesn't include it)
```

**Key use case — delegating role creation:**
Allow a developer to create IAM roles for their applications, but prevent them from creating roles with more permissions than they themselves have. The developer can only create roles that have the permission boundary attached.

```json
{
  "Effect": "Allow",
  "Action": "iam:CreateRole",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DevBoundary"
    }
  }
}
```

---

### 2e. Session Policies
Passed inline when calling `sts:AssumeRole` (or `AssumeRoleWithSAML` / `AssumeRoleWithWebIdentity`). They **further restrict the session** beyond what the role policy allows — they cannot add permissions.

```python
sts.assume_role(
    RoleArn="arn:aws:iam::123456789012:role/DataRole",
    RoleSessionName="audit-session",
    Policy=json.dumps({          # ← Session policy (JSON string)
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Action": ["s3:GetObject"],
            "Resource": "arn:aws:s3:::reports/Q1/*"
        }]
    })
)
```

Effective session permissions = `RolePolicy ∩ SessionPolicy ∩ PermissionBoundary`

---

## 3. Policy Evaluation Logic — Full Decision Flow

```
Incoming API request:

STEP 1 — Explicit DENY check
  Is there an explicit Deny in any policy (SCP, resource, identity, boundary, session)?
  YES → DENY (game over, no override possible)
  
STEP 2 — Organization SCP check  
  Is the action allowed by all SCPs in the hierarchy?
  NO → DENY
  
STEP 3 — Resource-based policy check (same-account ONLY)
  Does a resource-based policy in the SAME account explicitly Allow?
  YES → ALLOW (identity policy NOT required for same-account)
  
STEP 4 — IAM Permission Boundary check
  Is the action within the permission boundary (if one exists)?
  NO → DENY
  
STEP 5 — Session Policy check  
  Is the action allowed by session policy (if one exists)?
  NO → DENY

STEP 6 — Identity-based Policy check
  Does an identity-based policy Allow the action?
  NO → IMPLICIT DENY (default deny)
  YES → ALLOW
```

**Cross-account special rule:**
Both the **resource-based policy (Account B)** AND the **identity-based policy (Account A)** must allow the action. Neither alone is sufficient for cross-account access.

### Quick Summary Table

| Check | Same-Account | Cross-Account |
|-------|-------------|--------------|
| Explicit Deny present | DENY | DENY |
| Resource policy allows | ALLOW (no identity policy needed) | Identity policy ALSO needed |
| Only identity policy allows | ALLOW | Resource policy ALSO needed |
| Permission boundary blocks | DENY | DENY |
| SCP blocks | DENY | DENY |

---

## 4. Trust Policies (Role Assumption)

A trust policy is the **resource-based policy on an IAM role** that defines **who can assume the role**. It uses the `sts:AssumeRole` action.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111111111111:role/AppRole"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "sts:ExternalId": "UniqueSecret-12345" },
      "Bool": { "aws:MultiFactorAuthPresent": "true" }
    }
  }]
}
```

**Service trust (EC2, Lambda):**
```json
{
  "Principal": { "Service": "ec2.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

**SAML/OIDC federation trust:**
```json
{
  "Principal": { "Federated": "arn:aws:iam::ACCOUNT:saml-provider/MyCorp" },
  "Action": "sts:AssumeRoleWithSAML",
  "Condition": {
    "StringEquals": { "SAML:aud": "https://signin.aws.amazon.com/saml" }
  }
}
```

---

## 5. Cross-Account Access — Full Pattern

```
Account A (111111111111)           Account B (222222222222)
─────────────────────              ────────────────────────
IAM User/Role                      IAM Role "DataReader"
  Identity Policy:                   Trust Policy:
  Allow sts:AssumeRole                 Principal: Account A role
  on "DataReader" role ARN             Condition: ExternalId = "XYZ"
                                     Permission Policy:
           ──────────────────►         Allow s3:GetObject on bucket/*
              sts:AssumeRole
              (with ExternalId)
                    │
            Temp credentials
         (AK + SK + Session Token)
                    │
              Access S3 in Account B
```

**External ID — Confused Deputy Attack Prevention:**
- A third-party SaaS vendor may manage resources in multiple customer accounts.
- Without External ID, a malicious customer could trick the vendor into assuming a different customer's role.
- External ID is a secret shared only between you and the vendor — injected into AssumeRole calls.
- Exam: *"A vendor needs access to your account — how do you prevent confused deputy?"* → **Add ExternalId condition to the trust policy**.

---

## 6. STS (Security Token Service) — Deep Dive

STS issues **temporary security credentials** (Access Key ID + Secret Access Key + Session Token). Credentials expire automatically — no manual rotation needed.

| STS API | Use Case | Who Calls It |
|---------|----------|-------------|
| `AssumeRole` | Cross-account, service-to-service | IAM roles, services |
| `AssumeRoleWithSAML` | Enterprise SSO (SAML 2.0) | Corporate IdP via browser |
| `AssumeRoleWithWebIdentity` | OIDC (Cognito, Google, GitHub) | Web/mobile apps, CI/CD |
| `GetSessionToken` | Add MFA for existing credentials | IAM users with MFA device |
| `GetFederationToken` | Legacy web identity federation | Proxy apps (not recommended) |

**Session Duration:**
- `AssumeRole`: 15 min – 12 hours (default: 1 hour)
- `AssumeRoleWithSAML` / `AssumeRoleWithWebIdentity`: 15 min – 12 hours
- `GetSessionToken`: 15 min – 36 hours
- Role max duration set on the role itself (`--max-session-duration`)

**Revoking active sessions:**
- Attach an inline `Deny` policy to the role with condition `aws:TokenIssueTime < now` to invalidate all existing sessions.

---

## 7. IAM Federation

### SAML 2.0 (Enterprise/Corporate SSO)
```
Corporate User → Corporate IdP (AD/Okta) → SAML Assertion → AWS STS
→ sts:AssumeRoleWithSAML → Temp Credentials → AWS Console/API
```
- The IdP authenticates the user.
- The SAML assertion maps to one or more IAM roles via **attribute mapping**.
- No AWS credentials stored in the directory.

### OIDC (OpenID Connect) — Modern Apps & CI/CD
```
GitHub Actions / Google → ID Token (JWT) → AWS STS
→ sts:AssumeRoleWithWebIdentity → Temp Credentials
```
- Common for GitHub Actions → no static AWS keys in CI/CD secrets.
- Trust policy uses `token.actions.githubusercontent.com` as the federated principal.
- Condition: `"StringEquals": { "token.actions.githubusercontent.com:sub": "repo:org/repo:ref:refs/heads/main" }` — restricts to specific repo and branch.

### Amazon Cognito (App User Federation)
```
App User (signup/login via Cognito User Pool)
  → Cognito User Pool (AuthN = who are you?)
  → Cognito Identity Pool (AuthZ = what AWS can you access?)
  → STS AssumeRoleWithWebIdentity → Temp Credentials → S3/DynamoDB directly
```
- **User Pool**: Authentication — manages user directory, MFA, sign-in.
- **Identity Pool**: Authorization — maps authenticated users to IAM roles for direct AWS resource access.
- **Unauthenticated role**: Give guest users limited S3/DynamoDB access without logging in.

---

## 8. ABAC — Attribute-Based Access Control

Instead of creating one policy per team/project (RBAC), ABAC uses **tags** on both the principal and resource to make one dynamic policy that scales.

```json
{
  "Effect": "Allow",
  "Action": ["ec2:StartInstances", "ec2:StopInstances"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "ec2:ResourceTag/Project": "${aws:PrincipalTag/Project}",
      "ec2:ResourceTag/Env": "${aws:PrincipalTag/Env}"
    }
  }
}
```

A developer tagged `Project=Alpha, Env=Dev` can only start/stop EC2 instances tagged with the same values. **One policy, any number of projects.**

**ABAC advantages:**
- Scales without creating new policies per team.
- Works across services (S3, EC2, RDS) using the same tag keys.
- Session tags from SAML attributes enable per-user dynamic permissions.

---

## 9. IAM Conditions — Key Condition Keys for Exam

| Condition Key | Purpose | Example |
|--------------|---------|---------|
| `aws:MultiFactorAuthPresent` | Require MFA | `"Bool": {"aws:MultiFactorAuthPresent": "true"}` |
| `aws:SourceIp` | Allow from IP/CIDR | `"IpAddress": {"aws:SourceIp": "203.0.113.0/24"}` |
| `aws:RequestedRegion` | Restrict to region | `"StringEquals": {"aws:RequestedRegion": "us-east-1"}` |
| `aws:PrincipalOrgID` | Restrict to Org | `"StringEquals": {"aws:PrincipalOrgID": "o-xxxx"}` |
| `aws:PrincipalTag` | ABAC by principal tag | `"StringEquals": {"aws:PrincipalTag/Env": "prod"}` |
| `aws:ResourceTag` | ABAC by resource tag | `"StringEquals": {"aws:ResourceTag/Owner": "${aws:username}"}` |
| `aws:SourceVpc` | Restrict S3 to VPC | `"StringEquals": {"aws:SourceVpc": "vpc-xxxx"}` |
| `aws:SourceVpce` | Restrict to VPC endpoint | `"StringEquals": {"aws:SourceVpce": "vpce-xxxx"}` |
| `aws:SecureTransport` | Enforce HTTPS | `"Bool": {"aws:SecureTransport": "true"}` |
| `aws:TokenIssueTime` | Revoke old sessions | `"DateLessThan": {"aws:TokenIssueTime": "2026-01-01T00:00:00Z"}` |
| `sts:ExternalId` | Confused deputy prevention | `"StringEquals": {"sts:ExternalId": "my-external-id"}` |
| `kms:ViaService` | KMS used only via specific service | `"StringEquals": {"kms:ViaService": "s3.us-east-1.amazonaws.com"}` |

---

## 10. IAM Access Analyzer

**What it does:** Identifies resources in your account (or organization) that are **shared with external principals** — finds unintended public or cross-account exposure.

| Feature | Description |
|---------|-------------|
| **External access findings** | Flags S3 buckets, KMS keys, IAM roles, SQS queues, Lambda functions, Secrets Manager secrets accessible from outside your account |
| **Unused access findings** | Identifies IAM roles/users with permissions never used in the last 90 days (right-size policies) |
| **Policy validation** | Before applying a policy, validate it against IAM best practices and grammar |
| **Policy generation** | Generate a least-privilege policy from CloudTrail activity logs |
| **Zone of trust** | Set the scope to account or organization — anything outside = finding |

**Exam scenario:** "How do you identify which S3 buckets are accessible to external AWS accounts?" → IAM Access Analyzer.

---

## 11. IAM Identity Center (SSO) — Enterprise Access

**What it is:** Centralized SSO for accessing **multiple AWS accounts** and **business applications** from a single sign-on portal — replaces per-account IAM federation.

```
Corporate IdP (Azure AD/Okta)
  → IAM Identity Center
      → Permission Sets (IAM policies bundled together)
          → Assigned to Users/Groups in AWS Accounts
              → User logs into SSO portal
                  → Accesses Account A, Account B, Account C
                     with short-lived temporary credentials
```

**Key concepts:**

| Concept | Description |
|---------|-------------|
| **Permission Sets** | Bundles of IAM policies; assigned to user/group + account combinations |
| **Attribute-based access** | Map IdP attributes (department, title) to permission sets dynamically |
| **SCIM provisioning** | Auto-sync users/groups from IdP to Identity Center (no manual user management) |
| **MFA enforcement** | Enforce MFA at Identity Center level for all federated users |
| **Access Portal** | Web portal where users see all their assigned accounts and apps |

**Exam scenario:** "Company has 50 AWS accounts, employees need SSO access. Minimize credential management." → **IAM Identity Center with permission sets + SCIM from Okta/AD**.

---

## 12. IAM Best Practices for Exam

| Practice | Why |
|----------|-----|
| Enable MFA on root account | Root has unrestricted power |
| Never use root for day-to-day work | Accidental changes are irreversible |
| Create individual IAM users (not shared) | Audit trail per person |
| Prefer IAM roles over IAM users for apps | Temp creds auto-expire, no rotation needed |
| Use IAM groups to assign policies | Easier to manage than per-user assignment |
| Grant least privilege | Reduce blast radius |
| Use customer managed policies | Version-controlled, auditable, reusable |
| Enable credential reports | Audit key rotation, MFA status |
| Rotate access keys every 90 days | Limit exposure window |
| Use Access Analyzer | Detect unintended external access |

---

## 13. IAM Exam Traps — Critical Distinctions

| Trap | Correct Understanding |
|------|----------------------|
| **SCP grants permission** | ❌ SCPs only restrict — never grant on their own |
| **Explicit Deny can be overridden** | ❌ Explicit Deny always wins, no exceptions |
| **Resource policy alone is enough cross-account** | ❌ Both resource policy + identity policy required for cross-account |
| **KMS key policy alone is enough** | ❌ KMS requires key policy AND IAM policy — one alone is insufficient |
| **Permission boundary grants permissions** | ❌ It limits maximum — never grants on its own |
| **IAM group is a principal** | ❌ Groups cannot be used as principals in resource or trust policies |
| **Root user is affected by SCPs** | ❌ Root user is **exempt** from SCPs in all accounts |
| **Management account is affected by SCPs** | ❌ SCPs attached to the management account do not restrict it |
| **Session token is optional** | ❌ Temporary credentials from STS **require all 3**: AK + SK + Session Token |
| **AssumeRole duration is unlimited** | ❌ Maximum is 12 hours (set on the role); default is 1 hour |

---

## 14. Policy Evaluation — Same-Account vs Cross-Account Flowchart

```
Same-Account Request:
  Explicit Deny in any policy? → DENY
  SCP allows? → if NO → DENY
  Permission Boundary allows? → if NO → DENY
  Session Policy allows? → if NO → DENY
  (Identity Policy OR Resource Policy) allows? → if neither → DENY → else ALLOW

Cross-Account Request (Account A principal → Account B resource):
  Explicit Deny in any policy in EITHER account? → DENY
  SCP in Account A allows? → if NO → DENY
  SCP in Account B allows? → if NO → DENY
  Resource Policy in Account B allows Account A principal? → if NO → DENY
  Identity Policy in Account A allows cross-account action? → if NO → DENY
  Permission Boundary in Account A allows? → if NO → DENY
  All checks pass? → ALLOW
```

---

## 15. Common IAM Exam Scenarios

| Scenario | Solution |
|----------|----------|
| EC2 app needs to call S3 | IAM role attached as instance profile; identity policy grants `s3:GetObject` |
| Lambda needs to read Secrets Manager | Execution role with `secretsmanager:GetSecretValue` + KMS `kms:Decrypt` |
| Dev team creates roles but can't escalate privileges | Permission boundary on all created roles; condition on `iam:CreateRole` requiring boundary |
| Multi-account central S3 logging bucket | S3 bucket policy with `aws:PrincipalOrgID` condition; cross-account identity policies |
| Corp users need AWS access without IAM users | SAML 2.0 federation via IAM Identity Center with SCIM provisioning |
| GitHub Actions deploy to AWS without static keys | OIDC provider in IAM; trust policy with `token.actions.githubusercontent.com` |
| One policy scales to all project teams | ABAC with `aws:PrincipalTag/Project` matching `aws:ResourceTag/Project` |
| Revoke all sessions of a compromised role | Inline Deny with `aws:TokenIssueTime` condition on the role |
| Audit unintended external S3/KMS access | IAM Access Analyzer — external access findings |
| Least-privilege policy from actual usage | IAM Access Analyzer — policy generation from CloudTrail |

---

# 🌐 DOMAIN 3: INFRASTRUCTURE & NETWORK SECURITY (26%)

## Security Groups vs NACLs (Common Trap!)

| Feature | Security Group | NACL |
|---------|----------------|------|
| **Stateful** | ✅ Yes | ❌ No |
| **Level** | ENI (instance) | Subnet |
| **Allow/Deny** | Allow only | Allow + Deny |
| **Rule order** | All evaluated | First match |
| **Default egress** | Allow all | Depends |

**Exam Trick:** Question asks about stateless? → NACL

---

## VPC Endpoints (Key for Private Access!)

| Type | Purpose | Services |
|------|---------|----------|
| **Gateway** | Free route | S3, DynamoDB |
| **Interface** | ENI-based | Most others |

```
Want to access S3 from EC2 without NAT/IGW?
→ Use S3 Gateway Endpoint
→ Update route table
→ No internet traffic
```

---

## WAF (Web Application Firewall)

- Layer 7 protection (HTTP/HTTPS)
- Attached to: CloudFront, ALB, API Gateway
- Blocks: SQL injection, XSS, rate limiting
- Returns: 403 Forbidden

---

## DDoS Protection

| Level | Cost | Protection |
|-------|------|-----------|
| **Shield Standard** | Free | L3/L4 only |
| **Shield Advanced** | Paid | L3/L4/L7 + DRT |

---

## EC2 Security

| Type | Purpose |
|------|---------|
| **Security Groups** | Firewall (allow rules) |
| **NACLs** | Firewall (allow + deny) |
| **IAM role** | Permissions |
| **IMDSv2** | Secure metadata |

**Exam Focus:** IMDSv1 = vulnerable ❌ | IMDSv2 = secure ✅

---

## Container Security

| Service | Key Feature |
|---------|------------|
| **ECR** | Private image repo + scanning |
| **ECS** | Task roles + secrets |
| **EKS** | IRSA + network policies + RBAC |

---

---

# 🔐 DOMAIN 4: ENCRYPTION & DATA PROTECTION (22%)

---

## AWS KMS — Deep Dive

**What it is:** Managed service for creating and controlling the encryption keys used to encrypt your data. Every call to KMS is logged in CloudTrail.

### Key Concepts

#### Envelope Encryption (The Core Pattern)
```
Problem: KMS can only encrypt data up to 4KB directly.
Solution: Envelope encryption — encrypt a Data Key with KMS, use Data Key to encrypt large data.

Encrypt flow:
  1. Call kms:GenerateDataKey → returns plaintext DK + encrypted DK
  2. Encrypt data locally with plaintext DK (AES-256)
  3. Store encrypted data + encrypted DK together
  4. Discard plaintext DK from memory immediately

Decrypt flow:
  1. Call kms:Decrypt with encrypted DK → returns plaintext DK
  2. Decrypt data locally with plaintext DK
  3. Discard plaintext DK from memory
```

**Why this matters for exam:** Any question about encrypting large files/objects → envelope encryption. The plaintext data key is NEVER stored.

### KMS Key Types

| Key Type | Created By | Rotation | Cost | Use Case |
|----------|-----------|---------|------|----------|
| **AWS Managed Key** | AWS automatically | Auto every year | Free | Default for AWS services (S3, EBS) |
| **Customer Managed Key (CMK)** | You | Optional auto (yearly) or manual | $1/month | Full control, cross-account, custom policy |
| **AWS Owned Key** | AWS (shared) | Managed by AWS | Free | IAM Identity Center, Cognito (invisible) |
| **Data Key** | You (via GenerateDataKey) | N/A | Per API call | Encrypting data >4KB locally |

### KMS Key Access — Both Conditions Must Be Met

```
Can principal X use KMS key Y?

Condition 1: Key Policy allows it
       AND
Condition 2: IAM policy allows the KMS action

If ONLY key policy → Access DENIED (unless key policy explicitly gives full IAM control)
If ONLY IAM policy → Access DENIED (key policy takes precedence)

Exception: If key policy contains:
  {"Principal": {"AWS": "arn:aws:iam::ACCOUNT:root"}, "Action": "kms:*"}
  → IAM policies CAN grant access without key policy entries
```

**Exam trap:** KMS questions always involve BOTH key policy AND IAM policy. If a question says "the user has IAM kms:Decrypt permission but can't decrypt" → Key policy is missing the user.

### KMS Key Policy Template
```json
{
  "Statement": [
    {
      "Sid": "Enable IAM Control",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow Specific Role",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:role/AppRole"},
      "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "*"
    }
  ]
}
```

### KMS Key Grant
- Alternative to key policy for **programmatic, temporary access**.
- Grant allows a principal to use specific KMS operations on a key.
- Grantee can delegate grant to another principal (constrained operations only).
- **Retire or revoke** a grant to remove access without editing the key policy.
- Used by AWS services internally (e.g., S3 SSE-KMS creates a grant to use the CMK for your bucket).

### Key Rotation
| Rotation Type | How | Effect |
|--------------|-----|--------|
| **Automatic** (CMK) | Enabled on CMK — new key material every year | Old versions retained; existing ciphertext auto-decryptable |
| **Manual** | Create new CMK, re-encrypt data, update all references | Full key replacement |
| **AWS Managed** | Auto every 3 years | Cannot disable |

**Exam trap:** Rotating a CMK does NOT invalidate existing ciphertext. The old backing key is retained for decryption. To remove access to all data encrypted with a key → schedule key deletion.

### Multi-Region Keys
- Primary key in one region; replica keys in other regions.
- Same key ID, same key material — encrypted in one region can be decrypted in another.
- Use case: Global applications, cross-region disaster recovery, global client-side encryption.
- **Not the same as CRR** — key material is replicated, not the ciphertext.

### Key Deletion
- **Minimum waiting period: 7 days** (default 30 days).
- During the waiting period, the key is **disabled** — can cancel deletion.
- After deletion: ciphertext encrypted with this key becomes **permanently unrecoverable**.
- **Cryptographic erasure:** Fastest and most cost-effective way to render data unreadable — delete the CMK instead of overwriting every object.

### KMS Condition Keys for Exam
| Condition | Purpose |
|-----------|--------|
| `kms:ViaService` | Key usable ONLY when called via a specific AWS service (e.g., only S3 can use it) |
| `kms:CallerAccount` | Restrict to specific account |
| `kms:EncryptionContext` | Require specific context values (authentication binding) |
| `kms:RequestAlias` | Restrict to specific key alias |

### Encryption Context
- Key-value pairs passed with every KMS encrypt/decrypt call.
- **Not secret** — stored as plaintext metadata in CloudTrail.
- Acts as additional authenticated data (AAD): decryption requires the **same context** used during encryption.
- Use case: Bind encrypted data to a specific resource (e.g., `{"BucketName": "prod-data", "ObjectKey": "reports/q1.csv"}`).

### S3 Encryption Options
| Type | Key Owner | Key Location | Cost | CMK Control |
|------|----------|-------------|------|-------------|
| **SSE-S3** | AWS | AWS-managed | Free | ❌ No |
| **SSE-KMS** | You (CMK) | KMS | Per call | ✅ Yes |
| **DSSE-KMS** | You (CMK) | KMS | Per call | ✅ Yes — double-layer |
| **SSE-C** | You | Client-provided per request | Free | ✅ Yes, fully |
| **Client-Side** | You | Client | Free | ✅ Yes, fully |

**S3 Bucket Key:** Reduces KMS API calls by caching a data key at the bucket level (instead of one KMS call per object). Reduces cost by ~99%. Enable at bucket or object level.

### KMS vs CloudHSM

| Feature | AWS KMS | AWS CloudHSM |
|---------|---------|-------------|
| **Hardware** | Shared, multi-tenant HSM | Dedicated, single-tenant HSM |
| **FIPS Level** | FIPS 140-2 Level 2 | **FIPS 140-2 Level 3** |
| **Key Control** | AWS manages the HSM | **You fully control** — AWS has NO access |
| **Management** | Fully managed (serverless) | You manage the cluster |
| **Performance** | Throttled (10,000 req/sec per account) | High throughput, no throttle |
| **Use Case** | General encryption for AWS services | Regulatory FIPS L3 requirement, custom crypto, export controls |
| **Integration** | Native to all AWS services | Custom apps via PKCS#11, JCE, CNG |
| **Cost** | $1/CMK/month + API calls | ~$1.45/hour per HSM instance |

**Exam rule:** Question mentions "FIPS 140-2 Level 3", "dedicated HSM", "you control the key" → **CloudHSM**.

### KMS Exam Scenarios
| Scenario | Answer |
|----------|--------|
| Encrypt large file with KMS | GenerateDataKey → envelope encryption |
| S3 bucket data encrypted, customer controls the key | SSE-KMS with CMK |
| Cross-account access to encrypted EBS snapshot | Share snapshot + key grant to target account |
| Delete data instantly (compliance) | Schedule CMK deletion — cryptographic erasure |
| Auditor says KMS access is too broad | Use kms:ViaService to restrict to specific AWS service only |
| Key rotation without replacing ciphertext | Enable automatic rotation on CMK |
| FIPS 140-2 Level 3 required | CloudHSM |
| Reduce KMS API call costs for S3 | Enable S3 Bucket Key |

---

## S3 Security — Deep Dive

**What it is:** Object storage with the richest security feature set of any AWS storage service. S3 security combines access control, encryption, immutability, and logging.

### Access Control — Priority Order (Top Wins)
```
1. S3 Block Public Access (account or bucket level) — OVERRIDES all policies
2. Bucket Policy (resource-based)
3. ACL (legacy — use policies instead)
4. IAM Identity Policy (what the caller is allowed)
5. VPC Endpoint Policy (restricts S3 access from a VPC)

If Block Public Access is ON → even a bucket policy granting public access is blocked.
```

### Block Public Access (BPA)
- **4 settings** that can be applied at **account level** (applies to all buckets) or **bucket level**:
  1. `BlockPublicAcls` — Reject PUT requests that include public ACLs
  2. `IgnorePublicAcls` — Ignore all public ACLs on the bucket/objects
  3. `BlockPublicPolicy` — Reject bucket policies that grant public access
  4. `RestrictPublicBuckets` — Block cross-account public access even if policy allows
- **Best practice:** Enable all 4 at account level — override per bucket only if absolutely necessary.
- **Exam trap:** A bucket policy grants `Principal: *` but BPA is enabled → **access is still denied**.

### Bucket Policies — Key Patterns

**Enforce HTTPS only (deny HTTP):**
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
  "Condition": {"Bool": {"aws:SecureTransport": "false"}}
}
```

**Restrict S3 access to VPC only:**
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {"StringNotEquals": {"aws:SourceVpc": "vpc-xxxxxxxx"}}
}
```

**Restrict to Org only:**
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {"StringNotEquals": {"aws:PrincipalOrgID": "o-xxxxxxxxxxxx"}}
}
```

**Enforce KMS encryption on upload:**
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringNotEquals": {"s3:x-amz-server-side-encryption": "aws:kms"}
  }
}
```

### S3 Versioning & MFA Delete
- **Versioning:** Keeps all versions of every object — protects against accidental delete/overwrite.
- **MFA Delete:** Requires MFA token to permanently delete a version or suspend versioning.
  - Can only be enabled by the **root account**.
  - Must use CLI (not console) with root credentials.
  - Protects against insider threat and compromised admin credentials.

### S3 Object Lock (WORM)
Prevents objects from being deleted or overwritten for a fixed time or indefinitely.

| Mode | Who Can Override | Use Case |
|------|-----------------|----------|
| **Compliance** | Nobody — **not even root** | Regulatory (SEC Rule 17a-4, FINRA) |
| **Governance** | Users with `s3:BypassGovernanceRetention` permission | Internal compliance |

- **Retention Period:** Object cannot be deleted before expiry date.
- **Legal Hold:** Object cannot be deleted (no expiry — must explicitly remove hold).
- Requires **versioning enabled** on the bucket.
- **Glacier Vault Lock:** Same concept for Glacier vaults — lock policies are immutable once locked.

### Pre-Signed URLs
- Time-limited URL granting temporary access to a **private** S3 object.
- URL is signed with the **credentials of the IAM identity that generates it**.
- If the IAM role/user is deleted or permissions revoked, the presigned URL immediately stops working.
- Expiry: default 3600 seconds; maximum 7 days (when using IAM roles, max ~12 hours).
- **Use case:** Share private object with external party without making bucket public.
- **Exam trap:** Presigned URL generated by a role — if the role session expires, URL also expires.

### S3 Access Points
- Named network endpoints attached to a bucket — each with its own access policy.
- Simplify managing access for large datasets accessed by many applications.
- **VPC Access Points:** Restrict access to only requests from inside a specific VPC.
- Internet Access Points: Accessible from internet (normal presigned URLs).

### Origin Access Control (OAC) for CloudFront
- Restricts S3 bucket access to **only CloudFront** — prevents direct S3 URL access.
- OAC replaces the older OAI (Origin Access Identity).
- Bucket policy grants `cloudfront.amazonaws.com` principal with condition on distribution ARN.
- **Exam:** S3 content behind CloudFront, direct S3 access must be blocked → OAC.

### S3 Logging
| Log Type | What It Captures | Where | Latency |
|----------|-----------------|-------|--------|
| **S3 Server Access Logs** | Every request to the bucket (GET, PUT, DELETE) | Another S3 bucket | Best effort |
| **CloudTrail S3 Data Events** | API calls (GetObject, PutObject, DeleteObject) | CloudTrail trail | Near real-time |
| **CloudTrail Management Events** | Bucket-level operations (CreateBucket, PutBucketPolicy) | CloudTrail trail | Near real-time |

**For compliance/audit → CloudTrail Data Events** (more reliable, queryable with Athena).

### S3 Cross-Region Replication (CRR) Security
- Replication IAM role needs `s3:GetObject` on source + `s3:ReplicateObject` on destination.
- **Encrypted objects:** CMK decrypt permission needed on source CMK + encrypt permission on destination CMK.
- Objects are replicated with the **same encryption** unless you specify a different destination CMK.
- **Delete markers:** By default NOT replicated — enable explicitly if needed for compliance.

### S3 Exam Scenarios
| Scenario | Answer |
|----------|--------|
| S3 bucket accessible from internet despite deny policy | Check if Block Public Access is OFF |
| Prevent any HTTP requests to S3 | Bucket policy with `aws:SecureTransport: false` → Deny |
| Immutable audit records in S3 (regulators can't delete) | Object Lock — Compliance mode |
| Share private file with partner for 24 hours | Presigned URL with 86400s expiry |
| CloudFront serves S3 content, direct S3 URLs must be blocked | OAC on CloudFront distribution |
| Log all S3 GetObject/PutObject for SIEM | CloudTrail S3 Data Events |
| S3 access only from company VPC | Bucket policy with `aws:SourceVpc` condition |
| Encrypt all objects at upload with CMK | Bucket default encryption SSE-KMS + deny policy without encryption header |

---

## AWS Macie — Deep Dive

**What it is:** ML-powered service that **discovers and classifies sensitive data** stored in S3. It does not encrypt, mask, or remediate — discovery and alerting only.

### What Macie Detects
| Category | Examples |
|----------|----------|
| **PII** | Names, addresses, SSNs, passport numbers, driver's licence |
| **Financial** | Credit card numbers, bank account numbers, IBAN |
| **Credentials** | AWS access keys, private keys, API tokens, passwords |
| **Health** | Medical record numbers, drug codes (PHI) |
| **Custom** | Your own regex-based patterns (custom data identifiers) |

### How It Works
1. **Automated Discovery:** Continuously samples S3 objects across all buckets — cost-effective, lower coverage.
2. **Sensitive Data Discovery Jobs:** One-time or scheduled — full or sampled scan of specific buckets.
3. **Findings:** Generated when sensitive data is detected or a bucket has a security issue.

### Finding Types
| Type | Description |
|------|-------------|
| **SensitiveData findings** | Sensitive data found in an object (e.g., `SensitiveData:S3Object/Personal`) |
| **Policy findings** | Bucket security issue (e.g., bucket is public, encryption disabled, replication changed) |

### Integration
- **Security Hub:** Macie findings published in ASFF format.
- **EventBridge:** Trigger Lambda for automated response (quarantine, tag, notify).
- **S3:** Findings stored in S3 as JSON.
- **Organizations:** Centralize Macie across all accounts under a delegated admin.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Find all S3 buckets storing credit card data | Amazon Macie |
| Macie found PII — now mask it | Macie does NOT mask — use Glue DataBrew or Lambda |
| Continuous S3 sensitive data monitoring at low cost | Macie automated discovery |
| Custom data pattern (employee IDs) | Macie custom data identifiers |

---

## Secrets Manager vs Parameter Store — Deep Dive

| Feature | Secrets Manager | SSM Parameter Store |
|---------|----------------|--------------------|
| **Auto-rotation** | ✅ Built-in Lambda rotation | ❌ Manual only |
| **Secret versioning** | ✅ Always | ✅ Always |
| **Encryption** | KMS (always encrypted) | KMS for SecureString (optional) |
| **Cross-account** | ✅ Resource policy | ❌ Not supported |
| **Cost** | $0.40/secret/month + $0.05/10K API calls | Free (Standard); $0.05/advanced parameter/month |
| **Max size** | 64KB | 4KB (Standard) / 8KB (Advanced) |
| **Supported rotations** | RDS, Redshift, DocumentDB, custom Lambda | Manual only |
| **Best for** | Database credentials, API keys needing rotation | Config values, non-rotating parameters |

### Secrets Manager Rotation Mechanics
```
Rotation trigger (schedule or manual)
  → Lambda function (built-in or custom)
      1. createSecret: Generate new credentials
      2. setSecret: Update the target service (RDS password)
      3. testSecret: Validate new credentials work
      4. finishSecret: Mark new version as AWSCURRENT
  → Old version becomes AWSPREVIOUS (retained for rollback)
  → AWSPREVIOUS deleted after next rotation
```

### Parameter Store Hierarchy
```
/myapp/
  prod/
    db/password      (SecureString — KMS encrypted)
    db/username      (String)
    feature-flags/   (StringList)
  dev/
    db/password      (SecureString)
```

- Applications retrieve by path prefix: `GetParametersByPath("/myapp/prod")`.
- IAM policy can restrict to a path hierarchy: `"Resource": "arn:aws:ssm:*:*:parameter/myapp/prod/*"`.

### Exam Scenario
| Scenario | Answer |
|----------|--------|
| RDS password auto-rotated every 30 days | Secrets Manager with RDS native rotation |
| Store 100 config values cheaply | Parameter Store (free tier) |
| Dev team needs access to dev secrets only | Parameter Store with path-based IAM policy |
| Share secret with another AWS account | Secrets Manager resource policy |

---

## ACM & Private CA — Deep Dive

### AWS Certificate Manager (ACM)
- Issues **free public SSL/TLS certificates** for use with AWS services.
- **Supported services:** ELB (ALB/NLB), CloudFront, API Gateway, AppSync.
- **Cannot** export certificate private key — use ACM Private CA for that.
- **Auto-renewal:** ACM renews certificates automatically before expiry (60 days before).
- **DNS validation** (preferred) vs Email validation — DNS validation allows automated renewal.
- **Wildcard certs:** `*.example.com` covers all single-level subdomains.
- **Multi-domain SANs:** One cert covering multiple distinct domains.

### ACM Private CA (AWS Private Certificate Authority)
- Issue and manage **private certificates** for internal use (not trusted by browsers).
- Use cases: mTLS between microservices, internal APIs, IoT device certificates.
- **Exportable private keys** — unlike public ACM certificates.
- Supports **short-lived certificates** (hours) for Zero-Trust environments.
- **Subordinate CA:** Create a hierarchy — Root CA (offline/HSM) → Subordinate CA (AWS Private CA) → end-entity certs.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Free TLS cert for ALB | ACM public certificate |
| mTLS between internal microservices | ACM Private CA |
| Certificate with exportable private key | ACM Private CA |
| Auto-renewing wildcard cert for CloudFront | ACM (DNS validation) |

---

## EXAM TIPS & QUICK REFERENCE

### High-Priority Services (by test weight)
1. **IAM** - 25% of exam
2. **KMS & Encryption** - 20% of exam
3. **Logging & Monitoring** - 20% of exam
4. **Infrastructure Protection** - 26% of exam
5. **Incident Response** - 7% of exam

### Common Exam Scenarios
- **Overly permissive policies** → Use Access Analyzer
- **Compliance with regulations** → CloudTrail, Config, Macie
- **Data exfiltration detection** → Macie, GuardDuty, CloudWatch
- **Cross-account access** → Assume role, update trust policy, external ID
- **Encryption key compromise** → KMS key scheduling deletion, rotate data key
- **Container secrets** → Secrets Manager, Parameter Store, ECR private repo
- **Least privilege access** → IAM session policies, permission boundaries, resource policies

### Important Distinctions
- **KMS vs CloudHSM:** KMS = serverless, CloudHSM = dedicated hardware
- **Security groups vs NACLs:** SG = stateful, NACL = stateless
- **SSE-S3 vs SSE-KMS:** SSE-S3 = AWS managed, SSE-KMS = customer controlled
- **Secrets Manager vs Parameter Store:** Secrets = rotation, Parameter = free tier
- **VPC Endpoint Gateway vs Interface:** Gateway = S3/DynamoDB, Interface = others
- **SAML vs OIDC:** SAML = enterprise, OIDC = modern (use both)

---

## Key Formulas & Calculations

### Policy Priority (AWS evaluation logic)
```
1. Explicit Deny (game over)
2. Organization SCP (permission boundary)
3. Session Policy (permission boundary)
4. Identity Policy (allow permission)
5. Resource Policy (allow permission)
6. IAM Permissions Boundary (limit)
= Final decision on allow/deny
```

### Encryption Decision Tree
```
Need customer control? → YES → KMS (SSE-KMS for S3)
Different key per object? → YES → Data keys
Data > 4KB? → YES → Data key + envelope encryption
On-premises? → YES → Asymmetric KMS or Customer keys
FIPS 140-2 L3 required? → YES → CloudHSM
Serverless preferred? → YES → KMS
```

---

Last Updated: March 2026

---

## QUICK REFERENCE TABLES

### Service Feature Matrix

| Service | Encryption at Rest | At Transit | Logging | Multi-Region | Cost |
|---------|-------------------|------------|---------|--------------|------|
| S3 | SSE-S3/KMS/C2 | HTTPS | CloudTrail | CRR | Per GB |
| RDS | KMS | SSL/TLS | CloudWatch | Read Replicas | Per hour |
| DynamoDB | KMS/SSE-S3 | HTTPS | CloudWatch | Global Tables | Per request |
| EBS | KMS | N/A | CloudWatch | Snapshots | Per GB |
| ElastiCache | Redis AUTH | HTTPS | CloudWatch | Cross-AZ | Per hour |
| Secrets Manager | KMS | HTTPS | CloudTrail | Multi-region | Per secret |
| CloudTrail | S3 Encryption | HTTPS | CloudTrail | Multi-region | Per 100K |

### IAM Policy Elements Cheat Sheet

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DescriptiveStatementName",
      "Effect": "Allow|Deny",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/ROLE-NAME",
        "Service": "ec2.amazonaws.com",
        "Federated": "arn:aws:iam::ACCOUNT:saml-provider/PROVIDER"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:*"
      ],
      "Resource": [
        "arn:aws:s3:::bucket-name/*",
        "arn:aws:s3:::bucket-name"
      ],
      "Condition": {
        "StringEquals": {
          "aws:username": "johndoe"
        },
        "IpAddress": {
          "aws:SourceIp": ["192.0.2.0/24"]
        },
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        },
        "DateGreaterThan": {
          "aws:CurrentTime": "2026-03-18T00:00:00Z"
        }
      }
    }
  ]
}
```

### KMS Key Policy Template

```json
{
  "Sid": "Enable IAM User Permissions",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::ACCOUNT-ID:root"
  },
  "Action": "kms:*",
  "Resource": "*"
}
```

### ARN Reference

```
arn:aws:service:region:account-id:resource-type/resource-id
arn:aws:iam::123456789012:user/Development/product_1030/*
arn:aws:s3:::my_corporate_bucket/*
arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0
arn:aws:rds:us-east-1:123456789012:db:mydbinstance
arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012
```

---

## COMPLIANCE & REGULATORY FRAMEWORKS

### Compliance Standards Covered by AWS

#### HIPAA (Health Insurance Portability & Accountability Act)
- **Applies to:** Healthcare organizations, health plans
- **Requirements:** Encryption at rest/transit, audit logs, access controls
- **AWS Services:** HIPAA-eligible include RDS, S3, EBS, KMS, CloudTrail
- **BAA Required:** Business Associate Agreement with AWS

#### PCI DSS (Payment Card Industry Data Security Standard)
- **Applies to:** Organizations handling credit card data
- **Levels:** 1-4 based on transaction volume
- **Requirements:** Network segmentation, encryption, access logging
- **AWS Services:** All AWS services can be PCI-compliant with proper configuration

#### GDPR (General Data Protection Regulation)
- **Applies to:** EU resident data processing
- **Key Rights:** Right to be forgotten, data portability, consent
- **Data Processing:** DPA (Data Processing Addendum) with AWS
- **Data Residency:** EU regions (eu-west-1, eu-central-1, etc.)
- **AWS Services:** All available in EU regions

#### SOC 2 (Service Organization Control)
- **Type I:** Design of controls (at point in time)
- **Type II:** Effectiveness of controls (over time period)
- **Trust Services Criteria:** Security, availability, processing integrity, confidentiality, privacy

#### FedRAMP (Federal Risk and Authorization Management Program)
- **Applies to:** Federal government cloud usage
- **Levels:** Low, Moderate, High
- **AWS Services:** EC2, S3, RDS in GovCloud regions

### Compliance Best Practices
1. **CloudTrail Logging** - Audit API calls
2. **Config Rules** - Continuous compliance checking
3. **Encryption Keys** - Customer-managed CMK for sensitive data
4. **Network Isolation** - VPCs, subnets, security groups
5. **Access Controls** - IAM roles, least privilege
6. **Data Retention** - S3 Object Lock, lifecycle policies
7. **Monitoring** - CloudWatch, GuardDuty, Security Hub

---

## SCENARIO-BASED DECISION MAKING

### Scenario 1: Securing API Keys & Credentials
**Problem:** Database passwords and API keys scattered across code
**Solution Path:**
- ✅ Use **Secrets Manager** for automatic rotation
- ✅ Use **Parameter Store** for non-rotating secrets
- ✅ Enable **KMS encryption** for both
- ✅ Rotate keys every 30-90 days
- ✅ Use **IAM roles** instead of hardcoded credentials
- ❌ Don't store in environment variables
- ❌ Don't check into version control

### Scenario 2: Cross-Account Access for Partner Company
**Problem:** Partner needs to access S3 bucket in your account
**Solution Path:**
1. Create IAM **role in your account**
2. Set **trust relationship** to partner AWS account
3. Add **external ID** to prevent confused deputy
4. Set **permissions** to S3 bucket (least privilege)
5. Partner assumes role from their account
6. **Never** share IAM credentials

### Scenario 3: Data Exfiltration Detection
**Problem:** Concern about large data transfers
**Solution Path:**
- **Enable CloudTrail** for S3 API logging
- **Enable Macie** for data discovery & anomalies
- **Enable GuardDuty** for threat detection
- **Set CloudWatch alarms** for unusual API activity
- **Enable VPC Flow Logs** for network analysis
- **Use S3 access advisor** to identify unused permissions

### Scenario 4: Compliance Audit Required
**Problem:** Need to prove security controls
**Solution Path:**
- **CloudTrail** - Shows all API calls
- **Config** - Configuration changes over time
- **AWS Config rules** - Compliance violations
- **CloudWatch** - Metrics and alarms
- **Security Hub** - Centralized findings
- **Access Analyzer** - Permission analysis
- **Organization Trail** - All linked accounts

### Scenario 5: Encryption Key Compromised
**Problem:** Someone has access to your KMS key
**Solution Path:**
1. **Immediately disable the key** in KMS console
2. **Create new CMK** for future encryption
3. **Re-encrypt data** with new key
4. **Schedule key deletion** (7-30 day waiting period)
5. **Investigate access** via CloudTrail
6. **Audit permissions** on the key
7. **Revoke compromised credentials**

---

## COMMON MISTAKES & HOW TO AVOID THEM

### Mistake 1: Overly Permissive Policies
```json
// WRONG - Too permissive
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// RIGHT - Least privilege
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-bucket/documents/*"
}
```

### Mistake 2: Forgetting Principal in Trust Policy
```json
// WRONG - No principal specified
{
  "Effect": "Allow",
  "Action": "sts:AssumeRole"
}

// RIGHT - Principal specified
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/EC2Role"
  },
  "Action": "sts:AssumeRole"
}
```

### Mistake 3: Sharing Encrypted Snapshots Directly
- **Problem:** EC2 snapshots encrypted with customer KMS key cannot be directly shared
- **Solution:** 
  1. Join shared snapshot to EC2 instance
  2. Create unencrypted snapshot
  3. Re-encrypt with recipient's key
  4. OR use shared KMS key grants

### Mistake 4: Not Enabling MFA Delete on S3
- **Problem:** Anyone with delete permission can delete s3 objects
- **Solution:** Enable versioning + MFA delete
- **Cost:** Minimal + MFA device cost

### Mistake 5: Leaving EC2 IMDSv1 Enabled
- **Problem:** IMDS v1 vulnerable to SSRF attacks
- **Solution:** 
  ```bash
  # Disable IMDSv1 or require tokens (IMDSv2)
  aws ec2 modify-instance-metadata-options \
    --instance-id i-1234567890abcdef0 \
    --http-tokens required \
    --http-put-response-hop-limit 1
  ```

### Mistake 6: Not Rotating Access Keys
- **Problem:** Compromised keys give permanent access
- **Solution:** Rotate every 90 days
- **Automation:** Use IAM access key last used info

### Mistake 7: Public S3 Buckets
- **Problem:** Accidental data exposure
- **Solutions:**
  1. Enable **Block Public Access** at account level
  2. Enable **Bucket versioning** for recovery
  3. Use **CloudTrail** to audit access
  4. Use **bucket policies** to restrict access

---

## PERFORMANCE OPTIMIZATION TIPS

### CloudTrail Optimization
- **Use S3 prefix** to organize trails by date/service
- **Enable log file validation** for integrity
- **Use Athena** for querying large volumes
- **Partition by date** for faster Athena queries

### CloudWatch Cost Optimization
- **Filter logs at source** to reduce ingestion
- **Adjust retention** from 1 month to 1 week if possible
- **Use log sampling** for high-volume logs
- **Archive to S3** via Kinesis Firehose

### KMS Cost Optimization
- **Use S3 Bucket Keys** to reduce encryption API calls
- **Data key caching** reduces KMS API calls
- **Use SSE-S3** for non-sensitive data (free)
- **Batch operations** to encrypt in bulk

### VPC Flow Logs Optimization
- **Filter out internal traffic** to reduce costs
- **Send only to S3** for cost savings vs CloudWatch
- **Aggregate to CloudWatch via Firehose**

---

## EXAM QUESTION PATTERNS

### Pattern 1: "Which service provides..."
- **Data discovery?** → Macie
- **Threat detection?** → GuardDuty
- **Configuration tracking?** → Config
- **API logging?** → CloudTrail
# 📊 DOMAIN 5: LOGGING, MONITORING & THREAT DETECTION (20%)

---

## AWS CloudTrail — Deep Dive

**What it is:** Service that records every **API call** made in your AWS account — who did what, when, from where, and on which resource.

### Trail Types

| Trail Type | Scope | Best Practice |
|-----------|-------|---------------|
| **Management Events** | Control-plane actions (default, free for first copy) | Always on |
| **Data Events** | Object-level operations (S3 GetObject, Lambda invoke) | Enable for compliance — extra cost |
| **Insights Events** | Anomalous API call volume/error rates | Enable for unusual activity detection |
| **Org Trail** | Single trail delivering all accounts' events to central S3 | Enable from management account — cannot be disabled by member accounts |

### Log File Integrity Validation
- CloudTrail generates a **SHA-256 hash** of each log file and stores it in a **digest file** (signed with AWS private key).
- To validate: `aws cloudtrail validate-logs` — detects if logs were tampered with or deleted.
- **Exam:** Regulatory requirement to prove logs weren't altered → enable log file integrity validation.

### Protecting CloudTrail Logs
```
Attacker goal: Delete CloudTrail logs to hide activity.
Defenses:
  1. S3 Object Lock (WORM) on the log bucket → cannot delete
  2. S3 MFA Delete on log bucket versioning
  3. SCP Deny cloudtrail:StopLogging, cloudtrail:DeleteTrail
  4. CloudWatch Alarm on CloudTrail API calls (StopLogging, DeleteTrail)
  5. S3 Block Public Access on log bucket
  6. Restrict S3 bucket policy — only CloudTrail can write, no delete
```

### CloudTrail Log Structure — Key Fields
```json
{
  "eventTime": "2026-03-23T10:00:00Z",
  "eventSource": "s3.amazonaws.com",
  "eventName": "DeleteObject",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "203.0.113.45",
  "userAgent": "aws-cli/2.0",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAXXXXXXXXX:session-name",
    "arn": "arn:aws:sts::123456789012:assumed-role/DevRole/session-name",
    "accountId": "123456789012"
  },
  "requestParameters": {"bucketName": "prod-data", "key": "reports/q1.csv"},
  "responseElements": null,
  "errorCode": null
}
```

**Exam tip:** `userIdentity.type` tells you how the API was called:
- `Root` → root account
- `IAMUser` → direct IAM user
- `AssumedRole` → role assumption (cross-account, federation)
- `AWSService` → AWS service on your behalf

### Querying CloudTrail Logs
| Tool | Use Case |
|------|----------|
| **Athena** | Ad-hoc SQL queries on CloudTrail logs in S3 — cost-effective, fast |
| **CloudTrail Lake** | Managed event data store; SQL queries without Athena setup; retain up to 7 years |
| **CloudWatch Logs Insights** | If CloudTrail delivers to CloudWatch Logs — real-time queries |

**CloudTrail Lake vs Athena:**
- CloudTrail Lake = simpler setup, integrated, immutable event data store — no separate S3/Glue setup.
- Athena = more flexible, cheaper for large scale, can query any S3 data.

### Multi-Account Logging Architecture
```
All accounts → Org Trail (management account)
                    ↓
             Central S3 Bucket (log archive account)
                    ↓
         S3 Object Lock + restricted bucket policy
                    ↓
       Athena / CloudTrail Lake / SIEM for analysis
```

### Exam Scenarios
| Scenario | Answer |
|----------|--------|
| Prove audit logs were not modified | Enable log file integrity validation + validate with CLI |
| Prevent CloudTrail logs from being deleted | S3 Object Lock (Compliance mode) on log bucket |
| Query: who deleted this S3 object last month? | Athena on CloudTrail S3 Data Events |
| Single trail for all 50 AWS accounts | Org Trail from management account |
| Real-time alert when someone calls CreateUser | CloudTrail → CloudWatch Logs → Metric Filter → Alarm |
| Immutable, long-term event retention without S3 | CloudTrail Lake event data store |

---

## Amazon GuardDuty — Deep Dive

**What it is:** Continuous threat detection service using ML, anomaly detection, and threat intelligence to identify malicious or unauthorized behavior in AWS accounts.

### Data Sources GuardDuty Analyzes
| Source | What It Sees |
|--------|-------------|
| **CloudTrail Management Events** | API calls, console sign-ins, IAM changes |
| **CloudTrail S3 Data Events** | Object-level S3 activity (unusual access patterns) |
| **VPC Flow Logs** | Network traffic to/from EC2 instances |
| **DNS Logs** | EC2 instance DNS queries (detect C2 callbacks) |
| **EKS Audit Logs** | Kubernetes API server activity |
| **Lambda Network Logs** | Function network connections |
| **RDS Login Events** | Failed/unusual DB login attempts |
| **S3 Access Logs** | *(via S3 Protection feature)* |
| **Malware Scan** | EBS volume scan for malware on EC2/EKS |

**Key:** GuardDuty does NOT need you to enable/configure these sources — it pulls them directly from AWS infrastructure.

### Finding Categories & Examples
| Category | Example Finding | What It Means |
|----------|----------------|---------------|
| **Backdoor** | `Backdoor:EC2/C&CActivity.B` | EC2 communicating with known C2 server |
| **CryptoCurrency** | `CryptoCurrency:EC2/BitcoinTool.B` | EC2 mining cryptocurrency |
| **Recon** | `Recon:EC2/PortProbeUnprotectedPort` | Port scanning from outside |
| **Stealth** | `Stealth:IAMUser/CloudTrailLoggingDisabled` | Someone disabled CloudTrail |
| **Trojan** | `Trojan:EC2/BlackholeTraffic` | EC2 sending traffic to known blackhole |
| **UnauthorizedAccess** | `UnauthorizedAccess:IAMUser/TorIPCaller` | API calls from Tor exit node |
| **PenTest** | `PenTest:IAMUser/KaliLinux` | API calls from Kali Linux |
| **Exfiltration** | `Exfiltration:S3/ObjectRead.Unusual` | Unusual S3 read pattern |

### Finding Severity
- **High (7.0–8.9):** Immediate action needed (e.g., compromised EC2)
- **Medium (4.0–6.9):** Investigate soon
- **Low (0.1–3.9):** Informational

### GuardDuty Automation Pattern
```
GuardDuty Finding (High severity)
  → EventBridge Rule (filter by severity/type)
      → SNS Topic → Security team email
      → Lambda → Isolate EC2 (modify SG to deny all)
                → Disable IAM credentials
                → Take EBS snapshot for forensics
                → Create Security Hub finding
```

### Multi-Account GuardDuty
- Enable via **AWS Organizations** — delegated admin account manages all member accounts.
- Delegated admin sees all findings from all accounts in one pane.
- Member accounts **cannot disable** GuardDuty once enrolled by admin.
- Suppression rules (suppress low-value findings) managed centrally.

### Trusted IP Lists & Threat Lists
- **Trusted IP list:** IPs that GuardDuty will NOT generate findings for (e.g., your pen test IPs, approved CIDR blocks).
- **Threat intelligence list:** Custom IPs/domains GuardDuty treats as malicious.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Detect EC2 instance mining crypto | GuardDuty |
| Detect IAM credentials used from Tor | GuardDuty UnauthorizedAccess finding |
| GuardDuty finding → auto-isolate EC2 | EventBridge → Lambda → modify security group |
| Prevent member accounts disabling GuardDuty | Enroll via Organizations + SCP denying guardduty:DeleteDetector |
| GuardDuty found issue — how to prevent? | GuardDuty = detection only; fix with Systems Manager / Lambda remediation |
| Single view of findings for 50 accounts | GuardDuty Organizations with delegated admin |

---

## AWS Security Hub — Deep Dive

**What it is:** Centralized security findings aggregation and compliance dashboard. Collects findings from multiple AWS security services and applies automated security checks.

### What Security Hub Aggregates
| Source | Finding Type |
|--------|-------------|
| Amazon GuardDuty | Threat detection |
| Amazon Inspector | Vulnerability findings |
| Amazon Macie | Sensitive data |
| AWS Config | Compliance evaluation |
| IAM Access Analyzer | External access |
| AWS Firewall Manager | WAF/SG violations |
| AWS Health | Service events |
| Third-party partners | SIEM, EDR, custom tools |

### ASFF (Amazon Security Finding Format)
- All findings normalized into a **standard JSON schema** — enables cross-service analysis and automation.
- Key fields: `ProductArn`, `Types`, `Severity.Label`, `Resources`, `Compliance.Status`, `RecordState`.

### Security Standards (Automated Compliance Checks)
| Standard | Controls |
|----------|----------|
| **AWS Foundational Security Best Practices (FSBP)** | ~280 automated checks across services |
| **CIS AWS Foundations Benchmark** | CIS Level 1 & 2 controls |
| **PCI DSS** | Payment card requirements |
| **NIST SP 800-53** | US government framework |

Each control is either **PASSED**, **FAILED**, or **NOT AVAILABLE**.

### Cross-Region & Cross-Account Aggregation
- **Aggregation Region:** Designate one region to receive findings from all other regions.
- **Organizations integration:** Delegated admin sees all findings from all accounts.
- Findings flow: Member account → Security Hub regional → Aggregation region → SIEM/EventBridge.

### Automation Rules (New)
- Define rules that automatically update findings (suppress, enrich, route) based on criteria.
- Example: Auto-suppress `INFORMATIONAL` severity findings for test accounts.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Single dashboard for all security findings across services | Security Hub |
| Automated CIS benchmark compliance checking | Security Hub — CIS standard |
| SIEM integration for normalized findings | Security Hub ASFF → EventBridge → SIEM |
| Auto-suppress low-priority findings from dev accounts | Security Hub Automation Rules |

---

## AWS Config — Deep Dive

**What it is:** Records **configuration changes** to AWS resources over time, evaluates whether resource configurations comply with rules, and can automatically remediate violations.

### Core Concepts
| Concept | Description |
|---------|-------------|
| **Configuration Item (CI)** | Point-in-time snapshot of resource attributes |
| **Configuration History** | Timeline of all CIs for a resource — answers "what did this look like on date X?" |
| **Configuration Recorder** | The engine that captures resource changes — must be enabled per region |
| **Delivery Channel** | Where CIs are sent — S3 (required) + SNS (optional) |
| **Config Rule** | Evaluates whether a CI is compliant or non-compliant |
| **Conformance Pack** | Bundle of Config rules + remediation actions deployed as a unit |

### Config Rules
| Rule Type | Description | Example |
|-----------|-------------|--------|
| **AWS Managed** | Pre-built by AWS (~300+ rules) | `s3-bucket-public-read-prohibited` |
| **Custom Lambda** | Your Lambda function evaluates compliance | Check custom tagging standards |
| **Custom Policy (Guard)** | AWS CloudFormation Guard DSL rules | Evaluate JSON config against policy |

### Evaluation Triggers
- **Configuration change:** Rule evaluated whenever the resource config changes.
- **Periodic:** Evaluated on a schedule (1hr, 3hr, 6hr, 12hr, 24hr).
- **Hybrid:** Both.

### Remediation
```
Config Rule evaluates → NON_COMPLIANT
  → Remediation Action (SSM Automation document)
      → Auto-remediation: runs immediately on violation
      → Manual: security team clicks "Remediate"
  → Retry on failure (configurable attempts)
```

Common SSM Automation documents for remediation:
- `AWS-EnableS3BucketEncryption` — enable SSE on non-compliant bucket
- `AWS-DisablePublicAccessToRDSInstance` — disable public accessibility
- `AWS-RevokeUnusedIAMUserCredentials` — deactivate unused keys

### Multi-Account Config — Aggregator
- **Config Aggregator:** Collects Config data from multiple accounts/regions into one account.
- Central compliance view without granting cross-account IAM access.
- Works with AWS Organizations for auto-enrollment.

### Config vs CloudTrail
| Question | Service |
|----------|--------|
| What is the current config of this resource? | Config |
| What did this resource look like 30 days ago? | Config (history) |
| Who changed this resource? | CloudTrail |
| Is this resource compliant? | Config Rule |
| Which API call made the change? | CloudTrail |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Auto-remediate non-encrypted S3 buckets | Config Rule + SSM Automation remediation |
| Compliance check: are all EBS volumes encrypted? | Config managed rule `encrypted-volumes` |
| Historical config: what was SG rule 60 days ago? | Config configuration history |
| Deploy same security rules across all accounts | Config Conformance Pack |
| Config is not real-time | ✅ True — evaluate on change or schedule, not instant |

---

## Amazon CloudWatch — Security Patterns

**Security-relevant CloudWatch features:**

### Metric Filters on CloudTrail Logs
CloudTrail → CloudWatch Logs → Metric Filter → Alarm → SNS → Security team

**Critical metric filters to know:**
```
Filter Pattern                            Alert When
─────────────────────────────────────────────────────────
{ $.eventName = "StopLogging" }          CloudTrail disabled
{ $.errorCode = "AccessDenied" }         Multiple access denials
{ $.eventName = "CreateUser" }           IAM user created
{ $.eventName = "DeleteTrail" }          Trail deleted
{ $.userIdentity.type = "Root" }         Root user activity
{ $.eventName = "AuthorizeSecurityGroup" }  SG rule added
```

### CloudWatch Logs Insights — Security Queries
```sql
-- Find all AccessDenied errors in last 24h
filter errorCode = "AccessDenied"
| stats count(*) as denials by userIdentity.arn, eventName
| sort denials desc
| limit 20

-- Find all root account activity
filter userIdentity.type = "Root"
| fields @timestamp, eventName, sourceIPAddress
| sort @timestamp desc
```

### CloudWatch Anomaly Detection
- Applies ML to a metric's historical data to establish a normal band.
- Alarm fires when metric goes outside the band (e.g., unusual S3 API call volume).

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Alert when root user logs in | CloudTrail → CloudWatch Logs → Metric Filter (Root activity) → Alarm → SNS |
| Real-time query of security events | CloudWatch Logs Insights |
| Detect unusual spike in API errors | CloudWatch Anomaly Detection on error metric |

---

## Amazon VPC Flow Logs — Security Patterns

**What it captures:** Network traffic metadata (NOT packet contents) for VPC network interfaces.

### Flow Log Fields
```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

**Security-relevant fields:**
- `action`: ACCEPT or REJECT — rejected traffic = SG or NACL blocked it
- `srcaddr`/`dstaddr`: IPs involved
- `dstport`: destination port (80 = HTTP, 443 = HTTPS, 22 = SSH, 3389 = RDP)

### Common Security Patterns
| Pattern | What to Look For |
|---------|----------------|
| Port scan | Many REJECT entries from same src IP across many dstports |
| Data exfiltration | Unusually high bytes to external IP |
| Unauthorized SSH/RDP | ACCEPT on port 22/3389 from unexpected IPs |
| C2 communication | Repeated connections to unknown external IPs |

### Destinations
- **CloudWatch Logs** — for real-time alerting and Insights queries.
- **S3** — for long-term storage and Athena querying.
- **Kinesis Data Firehose** — for real-time streaming to SIEM.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Detect unusual outbound traffic from EC2 | VPC Flow Logs → Athena or CloudWatch Insights |
| Flow logs show REJECT on port 443 | SG or NACL blocking HTTPS traffic |
| Flow log traffic not captured | Not Flow Logs = packet inspector → use Network Firewall or Traffic Mirroring |

---

# 🚨 DOMAIN 6: INCIDENT RESPONSE & AUTOMATION (7%)

---

## Automated Remediation Pattern (EventBridge → Lambda)

```
Threat Detected (GuardDuty / Security Hub / Config)
  → EventBridge Rule (filter by finding type + severity)
      → Lambda function:
            • Describe current state (get instance/user/resource details)
            • Contain: isolate SG, disable key, detach role
            • Preserve: snapshot EBS, dump memory, copy logs
            • Notify: SNS → Slack / PagerDuty / Security team
            • Document: create Security Hub finding / ticket
```

---

## EC2 Instance Compromise — Full Playbook

```
Step 1 — DETECT
  GuardDuty finding: UnauthorizedAccess:EC2/SSHBruteForce or CryptoCurrency:EC2/BitcoinTool

Step 2 — CONTAIN (do NOT terminate yet)
  a. Modify security group → deny all inbound/outbound (isolate from network)
  b. Detach IAM instance profile (revoke permissions)
  c. Disable any associated IAM keys
  d. Tag instance: Quarantine=true

Step 3 — PRESERVE (forensic evidence)
  a. Create EBS snapshot of all attached volumes
  b. Enable memory capture if agent present (SSM Run Command)
  c. Preserve CloudTrail, VPC Flow Logs, OS logs in S3
  d. Note: do NOT stop/reboot — volatile memory data lost

Step 4 — INVESTIGATE
  a. Launch forensic EC2 in isolated VPC
  b. Attach snapshot as secondary volume (read-only)
  c. Analyse with Amazon Detective or manual tools
  d. Query CloudTrail for API calls from compromised role
  e. Query VPC Flow Logs for C2 IPs, exfil targets

Step 5 — REMEDIATE
  a. Terminate compromised instance
  b. Launch clean replacement from trusted AMI
  c. Patch vulnerability that enabled compromise
  d. Rotate all credentials that instance had access to
  e. Update SG, NACL, WAF rules

Step 6 — POST-INCIDENT
  a. Root cause analysis
  b. Update runbooks and detection rules
  c. File AWS Security report if required
```

---

## Credentials Compromise — Full Playbook

```
Scenario: IAM access key found on GitHub

Step 1 — DETECT
  GuardDuty: UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B
  or developer discovers key in public repo

Step 2 — CONTAIN (speed is critical)
  a. Immediately DEACTIVATE (not delete) the access key in IAM
     aws iam update-access-key --access-key-id EXAMPLEKEY --status Inactive
  b. Attach inline DENY policy to the user/role with TokenIssueTime condition
     (invalidates all active STS sessions from that credential)
  c. If root key: change root password, revoke all root access keys

Step 3 — INVESTIGATE
  a. CloudTrail: query all API calls by that key in last 24-48 hours
  b. Check: what resources were accessed/created/modified/deleted?
  c. Check: were any new IAM users/keys/roles created? (lateral movement)
  d. Check: were any data exfiltration events (S3 GetObject, RDS snapshot)?
  e. Check: were any new SGs opened?

Step 4 — REMEDIATE
  a. Undo all unauthorized changes
  b. Delete compromised access key
  c. Create new key (or switch to IAM roles)
  d. Notify affected data owners
  e. If data exfiltrated: invoke breach notification procedures
```

---

## Amazon Detective — Deep Dive

**What it is:** Security investigation service that uses ML and graph analysis to rapidly investigate the **root cause and scope** of security findings. It does not detect — it investigates.

### What Detective Analyzes
- Ingests: CloudTrail, VPC Flow Logs, GuardDuty findings, EKS audit logs, Security Hub findings.
- Builds a **behavior graph** — relationships between entities (IPs, roles, instances, accounts) over time.

### Key Capabilities
| Capability | Description |
|-----------|-------------|
| **Entity profiles** | Timeline of all activity for an IAM role, IP, instance, or S3 bucket |
| **Finding groups** | Cluster related findings that likely share a root cause |
| **Pivoting** | Start from a GuardDuty finding → pivot to related IPs → related instances → related roles |
| **Geolocation** | API calls from unexpected countries — visual map |
| **Peer comparison** | Compare entity behavior to similar entities (is this role acting like other roles?) |

### Detective vs GuardDuty vs Security Hub
| Service | Purpose |
|---------|--------|
| **GuardDuty** | Detect threats → generates findings |
| **Security Hub** | Aggregate + prioritize findings across services |
| **Detective** | Investigate findings → establish root cause + scope |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Investigate root cause of GuardDuty finding | Amazon Detective |
| Visualize which resources an attacker accessed | Amazon Detective entity profiles |
| Correlate GuardDuty + CloudTrail + Flow Logs in one place | Amazon Detective |

---

## AWS Automated Forensics Orchestrator for EC2

**What it is:** AWS Solution (not a managed service — a CloudFormation-deployed solution) that automates the forensic collection process when a GuardDuty finding is triggered on an EC2 instance.

### Workflow
```
GuardDuty High-Severity Finding
  → EventBridge
      → Step Functions state machine:
            1. Isolate EC2 (modify SG)
            2. Create EBS snapshots
            3. Copy memory dump (SSM agent)
            4. Store evidence in S3 (versioned + Object Lock)
            5. Calculate hashes (MD5/SHA-256) for chain of custody
            6. Create forensic EC2 in isolated VPC
            7. Attach snapshots
            8. Notify security team via SNS
            9. Post finding to Security Hub
```

### Chain of Custody
- All evidence stored in **S3 with Object Lock (Compliance mode)** — immutable.
- Hash values stored separately — prove evidence integrity.
- Every action logged in CloudTrail.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Automate EC2 forensics on GuardDuty finding | Automated Forensics Orchestrator |
| Maintain chain of custody for digital evidence | S3 Object Lock (Compliance) + hash verification |
| Forensic investigation without disrupting production | Snapshot + separate forensic VPC |

---

## Incident Response — Quick Reference

| Scenario | Immediate Action | Investigation | Remediation |
|----------|-----------------|--------------|-------------|
| EC2 compromised | Isolate SG + detach IAM role | Detective + CloudTrail + Flow Logs | Snapshot → forensic analysis → terminate → patch |
| IAM key leaked | Deactivate key + deny sessions | CloudTrail query all API calls | Delete key → rotate → check lateral movement |
| S3 data exfiltration | Add DENY bucket policy | Macie + CloudTrail Data Events | Revoke access + notify data owners |
| Ransomware on EBS | Isolate + snapshot | Check CloudTrail for initial access | Restore from clean backup |
| Unauthorized console login | Disable IAM user + MFA revoke | CloudTrail for all actions in session | Change passwords + enable hardware MFA |
| DDoS attack | Auto-absorbed by Shield Standard | CloudWatch metrics | Shield Advanced + WAF rate rules |
| GuardDuty: cryptomining EC2 | Isolate SG | CloudTrail for IAM actions that enabled it | Terminate + patch |

---

# 🎯 FINAL EXAM STRATEGY

## Time Management
- **65 questions in 170 minutes** = 2.5 min/question
- Read carefully (underline keywords)
- Flag hard questions, return later

---

## Answer Selection (4-Step Process)

```
1️⃣ READ question (identify domain)
2️⃣ APPLY decision engine (default answers)
3️⃣ ELIMINATE wrong (hardcoded, manual, users, public)
4️⃣ PICK best (IAM role, encrypted, automated, least privilege)
```

---

## Trap Recognition

❌ Uses IAM users → WRONG  
❌ Hardcoded credentials → WRONG  
❌ Manual process → WRONG  
❌ Public access → WRONG  
❌ SCP grants permission → WRONG  
❌ GuardDuty prevents → WRONG  

---

## Passing Strategy

✅ **AWS Default Answers:**
- IAM roles (not users)
- KMS encryption
- VPC endpoints
- GuardDuty detection
- Config compliance
- EventBridge automation
- CloudTrail logging

---

## Domain Priorities

1. **IAM (25%)** ← Study HARD
   - Policy evaluation logic
   - Cross-account access
   - Trust relationships
   
2. **Infrastructure (26%)**
   - SG vs NACL differences
   - VPC endpoints
   - WAF rules
   
3. **Data (22%)**
   - KMS key policy + IAM
   - S3 encryption + access
   
4. **Logging (20%)**
   - CloudTrail, GuardDuty, Config
   
5. **Incident Response (7%)**
   - Automation with Lambda

---

## Last 30 Minutes

- Review all flagged questions
- Check policy evaluation logic
- Verify encryption included
- Ensure IAM roles (not users)
- Submit confidently

---

## Final Mindset

When in doubt, pick answer that:
- ✅ Uses **IAM roles**
- ✅ Uses **KMS encryption**
- ✅ Uses **VPC endpoints**
- ✅ **Automates** everything
- ✅ **Avoids** manual work
- ✅ **Logs everything**

> **Coverage + Decision Engine + Trap Avoidance = PASS SCS-C03**

✅ **YOU CAN DO THIS. GO PASS THE EXAM!**

---

**Document:** AWS Security Specialty (SCS-C03) Strategic Cheatsheet  
**Last Updated:** March 18, 2026

## LAST-MINUTE REVIEW CHECKLIST

- [ ] **Root account:** Has MFA, not used regularly
- [ ] **IAM users:** Created, not root, have access keys rotated
- [ ] **CloudTrail:** Enabled, logs to S3, log file validation on
- [ ] **S3 buckets:** Block public access enabled, encryption enabled
- [ ] **RDS:** Encryption at rest enabled, SSL/TLS for connections
- [ ] **KMS keys:** Customer-managed, rotation enabled
- [ ] **VPC:** Security groups & NACLs minimal
- [ ] **GuardDuty:** Enabled for threat detection
- [ ] **Config:** Rules enabled for compliance
- [ ] **Secrets Manager:** Used for database credentials
- [ ] **VPC Flow Logs:** Enabled for network monitoring
- [ ] **CloudWatch:** Alarms set for critical events

---

**Study Tip:** Review at least 3-5 practice exams before attempting the real exam. Each exam typically has 2-3 questions per domain, with emphasis on IAM and infrastructure protection.

**Last Minute Tips:**
- Read questions carefully (trick answers are common)
- Eliminate obvious wrong answers first
- Look for "best practice" vs "works but suboptimal"
- Remember: AWS prefers managed services over manual work
- When in doubt, choose encryption or least privilege
- AWS Security Specialty often tests knowledge of when NOT to use a service

---

**Consolidated Cheatsheet Complete** ✓

# AWS Security Specialty (SCS-C03) — REAL Cheat Sheet

---

## 🧠 1. IAM — POLICY EVALUATION (CRITICAL)

### Evaluation Order
1. Explicit DENY → FINAL DENY
2. SCP (Org level)
3. Permission Boundary
4. IAM Policy
5. Resource Policy

> ANY DENY = DENY

---

## 🔥 Cross-Account Access

**Correct Pattern:**
- STS AssumeRole
- Trust Policy (target account)
- Permission Policy (source account)

**Avoid:**
- Access keys sharing
- Direct IAM users

---

## 🧠 When to Use What

| Scenario | Service |
|---------|--------|
| AWS internal app | IAM Role |
| External app | STS |
| Enterprise login | SAML |
| Social login | Cognito |

---

## ⚠️ Traps
- SCP does NOT grant permissions
- Root user = never use

---

## 🔐 2. KMS (HIGH WEIGHT)

### Core Concept
- KMS manages keys, NOT data

---

### Envelope Encryption
- KMS encrypts data key
- Data key encrypts actual data

---

### Access Requirements
- IAM Policy → Allow
- Key Policy → Allow

> BOTH required

---

### Key Types

| Type | Use |
|-----|-----|
| AWS-managed | Simple |
| Customer-managed | Full control |

---

### Traps
- Region-specific
- Cross-account requires key policy + IAM

---

## 🪣 3. S3 SECURITY

### Priority Order
1. Block Public Access (BPA)
2. Bucket Policy
3. IAM Policy

---

### Encryption
- Default → SSE-KMS

---

### Access Patterns

| Scenario | Solution |
|--------|---------|
| Public site | Bucket Policy |
| Private app | IAM Role |
| Cross-account | Bucket + IAM |

---

### Traps
- ACLs = almost always WRONG
- BPA overrides everything

---

## 🌐 4. NETWORK SECURITY

### SG vs NACL

| Feature | SG | NACL |
|--------|----|------|
| Stateful | Yes | No |
| Allow/Deny | Allow only | Allow + Deny |

> Default answer = Security Group

---

### Private AWS Access

| Service | Endpoint |
|--------|---------|
| S3/DynamoDB | Gateway |
| Others | Interface |

---

### NAT vs Endpoint
- NAT → Internet required
- Endpoint → Private AWS access

---

## 🛡️ 5. THREAT DETECTION

| Service | Purpose |
|--------|--------|
| GuardDuty | Threat detection |
| Macie | PII detection (S3) |
| Security Hub | Aggregation |
| Inspector | Vulnerabilities |

---

### Traps
- GuardDuty does NOT block attacks

---

## 🚨 6. INCIDENT RESPONSE

### Golden Pattern
EventBridge → Lambda → Remediation

---

### Compromised EC2
- Detach IAM role
- Isolate via SG
- Snapshot EBS

---

### Traps
- Manual steps = wrong
- Automation = correct

---

## 🏛️ 7. ORGANIZATIONS

### SCP
- Restricts max permissions
- Does NOT grant access

---

### Logging Pattern
- Org-wide CloudTrail
- Central logging account

---

### Traps
- Config ≠ real-time
- CloudTrail = API logs only

---

## 🔥 EXAM PATTERNS (REMEMBER THESE)

### Pattern 1 — Least Privilege
- IAM role
- Minimal permissions

---

### Pattern 2 — No Internet Exposure
- Private subnet
- VPC endpoint

---

### Pattern 3 — Encryption
- KMS
- SSE-KMS

---

### Pattern 4 — Cross-Account
- AssumeRole (STS)

---

## 💀 ELIMINATION STRATEGY

Always eliminate:
- Long-term credentials
- Hardcoded secrets
- Public access
- Manual processes

---

## 🧠 FINAL RULE

Pick the option that:
- Scales
- Is automated
- Uses least privilege
- Avoids human intervention

---