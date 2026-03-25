# AWS Security Specialty (SCS-C03) — COMPLETE CHEAT SHEET

## 📊 EXAM OVERVIEW
- **Exam Code:** SCS-C03
- **Duration:** 170 minutes
- **Questions:** 65 (roughly 30 are long 7–8 sentence scenario questions)
- **Passing Score:** 750/1000

---

## ⚡ REAL EXAM TIPS (From March 2026 Sitting)

| Tip | Details |
|-----|---------|
| **Trust your first answer** | Do not change answers at review time — instinct is usually right |
| **Select before flagging** | Always pick your best answer BEFORE flagging — never leave a question blank |
| **Read every option** | Even if you recognize the concept, distractor options are intentionally close |
| **30 of 65 are scenario questions** | Expect 7–8 sentence scenarios — do not skim, read every sentence |
| **Pacing** | ~2.5 minutes per question — flag hard ones and return at end |
| **ESL +30 min accommodation** | Non-native English speakers: claim free +30 minutes via aws.training → Request Exam Accommodations → ESL +30 MINUTES |
| **Study schedule (last week)** | One practice exam + 3–5 topic quizzes per day in the final week |

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
| Amazon GuardDuty | ML-based threat detection; ingests CloudTrail, DNS, VPC Flow Logs (core); S3 data events, EKS, RDS, Lambda, malware scan via optional protection plans (off by default) |
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

# 🔍 EXAM INTERPRETATION LAYER

> **Core rule:** Do NOT solve based on real-world design. Solve based on the AWS expected pattern. The keyword in the question drives the answer — not your architecture preference.

## Keyword → Service Mapping

| Keyword in question | Expected service |
|--------------------|-----------------|
| Aggregation / streaming (logs) | Kinesis Firehose or Kinesis Data Streams |
| View / query logs | CloudWatch Logs Insights |
| Central logging (multi-account) | CloudTrail Org Trail |
| Real-time threat detection | GuardDuty |
| Compliance / audit / drift | AWS Config |
| Secrets at runtime / auto-rotation | Secrets Manager |
| Bootstrap / startup config / non-secret params | SSM Parameter Store |
| Private access to AWS services (no internet) | VPC Endpoint |
| Security findings aggregation | Security Hub |
| Long-term security data lake | Amazon Security Lake |
| Encrypt data larger than 4 KB with KMS | `GenerateDataKey` (envelope encryption) |
| Apply policy to all future accounts in Org | Attach SCP / policy to the OU (not individual accounts) |

> **Tie-breaker:** When two answers look correct, pick the one whose service name matches a keyword in the question stem.

---

# 🚨 EXAM DECISION ENGINE (START HERE)

## Step 0 — Identify Intent First

Before classifying the domain, classify **what the question is asking you to do**:

| Intent | Correct tool family |
|--------|--------------------|
| **Visibility** — see / query / monitor | CloudWatch, CloudTrail |
| **Detection** — find threats / anomalies | GuardDuty, Inspector, Macie |
| **Aggregation** — collect / route / stream | Kinesis, Security Lake, EventBridge |
| **Enforcement** — restrict / prevent / govern | IAM, SCP, Config, Permission Boundary |
| **Response** — fix / isolate / remediate | Lambda, SSM Automation, EventBridge |

> If the question says "centralise" or "aggregate" → route to Kinesis/Security Lake, not CloudWatch.

---

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
| Log analysis / querying | CloudWatch Logs Insights | Fast queries — NOT aggregation |
| Log aggregation / streaming | Kinesis Firehose | Real-time delivery to S3/OpenSearch |

> ⚠️ **CloudWatch Logs Insights = querying only.** It does not aggregate logs from multiple accounts or stream data. Use Kinesis for aggregation pipelines.

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

# 🔐 IAM DEEP DIVE — THE FULL PICTURE

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
- SCPs apply to **every principal in member accounts, including the root user**.
- The management account is **never affected** by SCPs — they do not apply to the management account at all.

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

### SCP Exam Safety Rule

> ⚠️ **Never choose an SCP that can unintentionally block all access.**

| Dangerous SCP pattern | Why it's wrong on the exam |
|-----------------------|---------------------------|
| Broad `Deny *` with no exception clause | Locks all principals including break-glass roles |
| Condition on dynamic values (AMI IDs, resource names) | Can drift and block legitimate operations |
| No `aws:PrincipalArn` escape hatch | Emergency access becomes impossible |

**AWS exam preference:** Targeted SCPs with scoped conditions (`aws:PrincipalArn`, `aws:RequestedRegion`) over broad blanket denies. When in doubt, the answer that preserves emergency access is correct.

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

## 12. Amazon Verified Permissions — Fine-Grained App Authorization

**What it is:** Managed authorization service that externalizes access control logic from application code using the **Cedar policy language**. Handles "can this user do this action on this resource?" decisions.

### Core Concepts
| Concept | Description |
|---------|-------------|
| **Policy Store** | Repository of Cedar policies and schema — the source of truth for authorization |
| **Cedar** | Expressive, fine-grained policy language designed for correctness and auditability |
| **IsAuthorized API** | Application calls this API with `principal`, `action`, `resource` → receives `ALLOW/DENY` |
| **Schema** | Defines entity types, actions, and relationships — enables policy validation |
| **Entities** | Real-world objects passed at authorization time (e.g., user attributes, resource tags) |

### Cedar Policy Example
```cedar
// Allow managers to approve expense reports in their department  
permit (
  principal in Role::"Manager",
  action == Action::"ApproveExpense",
  resource
) when {
  principal.department == resource.department
};
```

### Authorization Patterns
| Pattern | Cedar Approach |
|---------|---------------|
| **RBAC** | Define roles (groups of principals), permit role → action |
| **ABAC** | Use `.when` clauses with attributes: `principal.clearance >= resource.classification` |
| **ReBAC** | Hierarchy-based: `principal in Group::"Admins"` or parent/child relationships |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Externalize fine-grained auth from app code | Amazon Verified Permissions + Cedar |
| Application needs "can user X read document Y?" decision | Verified Permissions `IsAuthorized` API call |
| Centralize authorization policies across microservices | Single Verified Permissions policy store |
| Audit every authorization decision | CloudTrail logs all `IsAuthorized` API calls |

---

## 13. IAM Roles Anywhere — On-Premises Workload Access

**What it is:** Enables on-premises servers, containers, and applications to obtain **temporary AWS credentials** using X.509 certificates — eliminating long-term IAM access keys for on-prem workloads.

### How It Works
```
On-prem workload holds X.509 certificate (from your CA or AWS Private CA)
  → Presents certificate to IAM Roles Anywhere
      → Roles Anywhere validates against a Trust Anchor
          → Returns temporary STS credentials (AK + SK + session token)
              → Workload calls AWS APIs with temporary credentials
```

### Key Concepts
| Concept | Description |
|---------|-------------|
| **Trust Anchor** | Reference to a CA (AWS Private CA or external CA). Certificates signed by this CA are trusted |
| **Profile** | Defines which IAM roles can be assumed and the session duration (max 12h) |
| **Certificate** | X.509 cert presented by the workload — must be signed by a trusted CA |
| **Credential Helper** | AWS-provided tool (`aws_signing_helper`) that handles cert-based credential retrieval |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| On-prem servers need AWS access without long-term keys | IAM Roles Anywhere with X.509 certs |
| Eliminate static access keys from physical data center | IAM Roles Anywhere + AWS Private CA |
| On-prem CI/CD pipeline needs temporary AWS credentials | IAM Roles Anywhere (or OIDC for cloud CI) |
| Certificate-based authentication to AWS APIs | IAM Roles Anywhere Trust Anchor |

---

## 14. AWS Directory Service

**What it is:** Managed Microsoft Active Directory in the AWS Cloud. Three variants for different use cases.

### Three Types Compared
| Type | Description | Use Case |
|------|-------------|----------|
| **AWS Managed Microsoft AD** | Full Microsoft AD (2019) running on managed Windows DCs in AWS | Large orgs, trust with on-prem AD, MFA, RDS/WorkSpaces integration |
| **AD Connector** | Proxy — redirects LDAP/Kerberos queries to your on-prem AD | Small orgs that want AD auth but keep directory on-prem; no directory replication |
| **Simple AD** | Samba-based, limited AD-compatible service | Small orgs with basic needs, no trust relationships, no MFA |

### Key Protocols and Ports
| Protocol | Port | Use |
|----------|------|-----|
| LDAP | 389 (TCP/UDP) | Directory queries (cleartext) |
| LDAPS | 636 (TCP) | Encrypted directory queries |
| Kerberos | 88 (TCP/UDP) | Authentication tickets |
| DNS | 53 (TCP/UDP) | Domain name resolution |
| SMB/CIFS | 445 (TCP) | File shares |

### Trust Relationships
- **AWS Managed Microsoft AD** can establish **two-way forest trust** with on-prem AD.
- Users in on-prem AD can then access AWS resources via federated IAM roles.
- Requires Direct Connect or VPN for connectivity.
- **AD Connector** does NOT replicate the directory — it proxies queries to on-prem.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Users log in AWS with on-prem AD credentials | AD Connector (proxy) or Managed AD (trust) |
| Need MFA with AD for AWS WorkSpaces | AWS Managed Microsoft AD + MFA (RADIUS) |
| Small org needs basic LDAP-compatible auth | Simple AD |
| Keep all directory data on-premises | AD Connector — no data stored in AWS |
| RDS SQL Server Windows Authentication | AWS Managed Microsoft AD |

---

## 15. IAM Policy Simulator

**What it is:** Tool (console and API) that tests and validates IAM policies WITHOUT making real API calls. Simulates what an IAM user, role, or group can or cannot do.

### What You Can Test
- Evaluate policies for users, groups, or roles
- Test the effect of SCPs, permission boundaries, and resource policies together
- **Identify why an action is denied** — shows which policy is causing the deny
- Simulate with custom context keys (conditions like `aws:RequestedRegion`, `aws:MultiFactorAuthPresent`)

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Test if a role can call `kms:Decrypt` before deploying | IAM Policy Simulator |
| Diagnose why `s3:PutObject` returns AccessDenied | IAM Policy Simulator — shows exact deny source |
| Validate SCPs before applying to account | IAM Policy Simulator with SCP context |

---

## 16. IAM Best Practices for Exam

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

## 17. IAM Exam Traps — Critical Distinctions

| Trap | Correct Understanding |
|------|----------------------|
| **SCP grants permission** | ❌ SCPs only restrict — never grant on their own |
| **Explicit Deny can be overridden** | ❌ Explicit Deny always wins, no exceptions |
| **Resource policy alone is enough cross-account** | ❌ Both resource policy + identity policy required for cross-account |
| **KMS key policy alone is enough** | ❌ KMS requires key policy AND IAM policy — one alone is insufficient |
| **Permission boundary grants permissions** | ❌ It limits maximum — never grants on its own |
| **IAM group is a principal** | ❌ Groups cannot be used as principals in resource or trust policies |
| **Root user is exempt from SCPs** | ❌ SCPs apply to **all principals in member accounts, including root**. Exception: the management account root is NOT affected by SCPs |
| **Management account is affected by SCPs** | ❌ SCPs do not apply to the management account at all |
| **Session token is optional** | ❌ Temporary credentials from STS **require all 3**: AK + SK + Session Token |
| **AssumeRole duration is unlimited** | ❌ Maximum is 12 hours (set on the role); default is 1 hour |

---

## 18. Policy Evaluation — Same-Account vs Cross-Account Flowchart

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

## IAM Quick Decision Tree

When the question involves access control, use this shortcut before reading answer options:

```
Is it cross-account?
  YES → sts:AssumeRole (role in the target account)
  NO  ↓

Is it same-account, one resource?
  YES → Resource-based policy may be enough (S3, KMS, Lambda)
  NO  ↓

Need to cap maximum permissions for a developer/role?
  YES → Permission Boundary
  NO  ↓

Need org-wide or OU-wide restriction?
  YES → SCP
  NO  ↓

Tag-based dynamic access?
  YES → ABAC (aws:PrincipalTag / aws:ResourceTag)
```

---

## 19. Common IAM Exam Scenarios

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

## Least Operational Overhead — Decision Rule

> **"Least operational overhead" does NOT always mean the simplest service.** It means the most AWS-native managed solution that requires the fewest ongoing manual actions.

| Goal | Least-overhead answer |
|------|----------------------|
| View / alert on logs | CloudWatch Logs + Metric Filters + Alarms |
| Process / route / aggregate logs | Kinesis Firehose (no servers to manage) |
| Long-term storage + ad-hoc query | S3 + Athena (or CloudTrail Lake for API logs) |
| Centralised multi-account security posture | Security Hub (delegated admin) |
| Rotate secrets automatically | Secrets Manager (not Parameter Store — no built-in rotation) |
| Scan new ECR images for vulnerabilities | Inspector v2 (continuous, no scheduled tasks) |

---

# 🌐 DOMAIN 3: INFRASTRUCTURE & NETWORK SECURITY

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

## EC2 Image Builder — Golden AMI Pipeline

**What it is:** Automates the creation, testing, and distribution of secure, hardened EC2 AMIs (Amazon Machine Images).

### Pipeline Components
| Component | Description |
|-----------|-------------|
| **Recipe** | Base image + build components (software installs, patches, config) |
| **Build Component** | YAML document — runs scripts to install and configure software |
| **Test Component** | YAML document — runs validation tests (CIS benchmark checks, compliance scans) |
| **Pipeline** | Orchestrates build → test → distribute on a schedule or trigger |
| **Distribution Settings** | Share AMI to other accounts/regions or publish to Marketplace |

### Golden AMI Security Strategy
```
Latest base AMI (Amazon Linux 2023 / Windows Server 2022)
  → Apply OS patches (via SSM/dnf/yum)
  → Install agents (CW agent, SSM agent, Inspector agent)
  → Apply CIS hardening scripts
  → Run automated compliance tests
  → Distribute hardened AMI to all AWS accounts in Org
  → Enforce via SCP: EC2 instances must use images from this AMI pipeline
```

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Automate patch-hardened AMI builds | EC2 Image Builder pipeline |
| Ensure all EC2 instances boot from approved images | Image Builder + SCP requiring specific AMI tags |
| Automate CIS benchmark hardening before AMI distribution | Image Builder test components |

---

## AWS Systems Manager — Security Deep Dive

### Session Manager
**What it is:** Browser-based and CLI-based interactive shell access to EC2 and on-premises servers — **without opening port 22 or 3389**.

| Feature | Details |
|---------|---------|
| **No inbound ports required** | SSM agent on instance → communicates outbound to SSM endpoint (HTTPS) |
| **Session logging** | Full keystroke logging to CloudTrail, S3, or CloudWatch Logs |
| **IAM-controlled** | Access governed by IAM policies (`ssm:StartSession`), not SSH keys |
| **OS user restriction** | Can restrict which OS user the session runs as |
| **PrivateLink support** | Use VPC endpoint for SSM so traffic never hits public internet |

### EC2 Instance Connect (EIC)
| Feature | Details |
|---------|---------|
| **Port 22 required** | Inbound SSH still required in Security Group (but only from EC2 Instance Connect IP range) |
| **Short-lived keys** | EC2 Instance Connect pushes a one-time public SSH key valid for 60 seconds |
| **Auth via IAM** | IAM controls who can push keys (`ec2-instance-connect:SendSSHPublicKey`) |
| **No persistent keys** | No need to manage SSH key pairs in the OS |

### Session Manager vs EC2 Instance Connect
| Aspect | Session Manager | EC2 Instance Connect |
|--------|----------------|---------------------|
| **Open port required** | ❌ None | ✅ Port 22 inbound |
| **SSH required** | ❌ No | ✅ Yes |
| **Key management** | ❌ No keys | ✅ One-time ephemeral key |
| **Full session logging** | ✅ CloudTrail + S3 + CW | ⚠️ Limited (IAM action only) |
| **Network requirement** | HTTPS to SSM endpoint | SSH to instance IP |
| **Best for** | Compliance, no-internet instances | Quick SSH when port 22 is acceptable |

### Patch Manager
**What it is:** Automates OS and application patching across EC2 and on-premises instances at scale.

| Concept | Description |
|---------|-------------|
| **Patch Baseline** | Defines which patches to approve (auto-approve by severity, exclude by CVE) |
| **Maintenance Window** | Scheduled time window for patch application (e.g., every Sunday 2 AM) |
| **Patch Group** | Tag-based grouping of instances — assign different baselines per group (prod vs dev) |
| **Pre-defined baselines** | AWS provides baselines for Amazon Linux, RHEL, Windows, Ubuntu |
| **Custom baselines** | Create your own — approve specific CVEs, block specific patches |
| **Inspector integration** | Inspector v2 findings feed into Patch Manager for vulnerability-driven remediation |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Patch all EC2 instances without SSH or RDP access | Session Manager + Patch Manager (no inbound ports) |
| Apply different patch policies to prod vs dev EC2 | Patch Manager Patch Groups + separate baselines |
| Schedule weekly patching with rollback on failure | Patch Manager with Maintenance Window |

### State Manager
**What it is:** Maintains desired configuration state on EC2 and on-premises instances using SSM Associations.

- An **Association** defines: which document to run, on which instances (targets), on what schedule.
- Continuously re-applies the configuration if it drifts — ensures consistent security posture.
- Use cases: Ensure CW agent is always running, disable root SSH login, enforce host-based firewall rules.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Continuously enforce security configuration on EC2 | SSM State Manager Association |
| Ensure SSM agent is always running on all instances | State Manager Association for SSM agent install |

---

## Network Access Analyzer

**What it is:** Identifies unintended network access to AWS resources by analyzing VPC configurations — without generating actual traffic.

### How It Works
- Define a **Network Access Scope** (source: internet, specific CIDR; destination: EC2 instance, RDS) .
- Network Access Analyzer analyzes security groups, route tables, NACLs, VPC endpoints, and peering connections.
- Returns **findings** where a path exists that you didn't intend.
- Use case: Verify that database instances are NOT reachable from the internet; verify no publicly-routed path to private instances.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Verify no unintended internet access to private RDS instances | Network Access Analyzer |
| Analyze VPC network paths without generating traffic | Network Access Analyzer |
| Part of compliance check: no database is internet-accessible | Network Access Analyzer scope for 0.0.0.0/0 → RDS |

---

## AWS Direct Connect — MACsec Encryption

**What it is:** Layer 2 encryption for Direct Connect (DC) connections between your on-premises router and the AWS Direct Connect location.

### Key Facts
- **MACsec** (Media Access Control Security) encrypts traffic at Layer 2 — below TCP/IP.
- Available on **dedicated connections** (1 Gbps, 10 Gbps, 100 Gbps) — not hosted connections.
- Encryption occurs between the customer router and the Direct Connect endpoint — **before entering AWS**.
- Does NOT replace application-level TLS — provides layer 2 encryption for the physical connection.
- Configured via the Direct Connect console: enable MACsec, exchange CKN/CAK key pairs.

| Attribute | Value |
|-----------|-------|
| Layer | Layer 2 (Ethernet frames) |
| Algorithm | AES-256-GCM |
| Available on | Dedicated connections only |
| Key management | Customer provides CKN/CAK pair |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Encrypt Direct Connect link at Layer 2 | Enable MACsec on dedicated connection |
| Direct Connect is not encrypted by default | True — use MACsec for L2 or Site-to-Site VPN over DC for L3 encryption |
| Highest-throughput encrypted on-prem to AWS link | 100 Gbps DC + MACsec |

---

## GenAI OWASP Top 10 for LLM Applications

> **Exam note:** SCS-C03 added generative AI security. Focus on attack categories and AWS mitigations.

| Rank | Vulnerability | Description | AWS Mitigation |
|------|--------------|-------------|----------------|
| **LLM01** | Prompt Injection | Malicious prompt overrides LLM instructions | Bedrock Guardrails (topic filters, denied topics) |
| **LLM02** | Insecure Output Handling | LLM output executed as code/HTML without validation | Output validation in app logic; Bedrock Guardrails |
| **LLM03** | Training Data Poisoning | Corrupt training data compromises model behavior | Data validation pipelines; SageMaker data quality |
| **LLM04** | Model Denial of Service | Overload model with expensive requests | Throttling; Bedrock model invocation limits; WAF rate limiting |
| **LLM05** | Supply Chain Vulnerabilities | Compromised third-party model or plugin | Use only verified models; ECR scanning for containers |
| **LLM06** | Sensitive Information Disclosure | Model leaks training data or confidential context | Bedrock Guardrails PII detection; data protection policies |
| **LLM07** | Insecure Plugin Design | Plugin gives LLM excessive permissions | IAM least privilege for Bedrock Agents actions |
| **LLM08** | Excessive Agency | LLM takes unintended actions via tools | Bedrock Agents with human-in-the-loop approval |
| **LLM09** | Overreliance | Users over-trust LLM outputs without validation | App design; not an AWS service mitigation |
| **LLM10** | Model Theft | Attacker extracts model weights or logic | IAM policies restricting `bedrock:InvokeModel`; VPC endpoints |

### Key AWS GenAI Security Services
- **Amazon Bedrock Guardrails** — topic filters, denied topics, PII detection/redaction, word filters, grounding checks.
- **Amazon Bedrock Agents** — define allowed tool actions with IAM roles; human approval for sensitive steps.
- **VPC Interface Endpoints** — keep all Bedrock API calls off the public internet.

---

## Container Security

| Service | Key Feature |
|---------|------------|
| **ECR** | Private image repo + scanning |
| **ECS** | Task roles + secrets |
| **EKS** | IRSA + network policies + RBAC |

---

## AWS Network Firewall — Deep Dive

**What it is:** Managed stateful/stateless deep-packet inspection (DPI) firewall deployed **inside your VPC**. Goes beyond Security Groups and NACLs — provides IDS/IPS, FQDN-based egress filtering, and Suricata-compatible signature rules.

### Architecture
```
Internet → IGW
  → Firewall Subnet (Network Firewall endpoint, one per AZ)
    → Private Subnet (EC2 / ALB / RDS)

Or centralized via Transit Gateway (hub-and-spoke):
  Spoke VPCs → TGW → Inspection VPC (Network Firewall) → Internet
```
- Deploy **one firewall endpoint per AZ** in a dedicated firewall subnet.
- Modify VPC route tables so traffic passes through the firewall endpoint before reaching its destination.

### Rule Group Types

| Rule Type | Layer | What It Matches | Use Case |
|-----------|-------|-----------------|----------|
| **Stateless** | L3/L4 | 5-tuple: src/dst IP, src/dst port, protocol | Bulk allow/deny by CIDR or port |
| **Stateful (standard)** | L3–L7 | Connection state, application layer | Block SSH outbound; allow HTTPS only |
| **Stateful (Suricata IDS/IPS)** | L7 | Payload signatures (Suricata rule format) | Detect malware C2 traffic, exploit patterns |
| **Stateful (domain list)** | L7 (DNS/TLS SNI) | Fully-qualified domain names | Egress allowlist (only `.amazonaws.com`); block DGA domains |

### Network Firewall vs WAF vs SG vs NACL

| Control | Layer | Scope | Protocols | Use When |
|---------|-------|-------|-----------|----------|
| Security Group | L3–L4 stateful | Per ENI | All | Per-instance allow rules |
| NACL | L3–L4 stateless | Per subnet | All | Subnet-level explicit deny |
| WAF | L7 | CloudFront / ALB / API GW | HTTP/HTTPS only | Block SQLi, XSS, rate-limit HTTP |
| Network Firewall | L3–L7 stateful+stateless | VPC perimeter | All protocols | IDS/IPS, FQDN filtering, non-HTTP protocol control |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Block all egress except approved domain list | Network Firewall stateful domain-list rule group |
| Detect malware signatures in network payloads | Network Firewall with Suricata rules |
| Centralized inspection for traffic between 20 VPCs | Network Firewall in inspection VPC behind Transit Gateway |
| Block FTP or SMB outbound (non-HTTP protocols) | Network Firewall stateful rules (WAF cannot do this) |
| WAF is deployed — why is non-HTTP port 22 traffic getting through? | WAF only inspects HTTP/HTTPS; add Network Firewall for all other protocols |

---

## AWS Firewall Manager — Deep Dive

**What it is:** Single pane of glass to **centrally deploy and enforce** security policies across all accounts in an AWS Organization. Takes away the ability of individual accounts to deviate from the baseline.

### Prerequisites
1. **AWS Organizations** enabled.
2. **AWS Config** enabled in every member account and region in scope.
3. Designate a **Firewall Manager administrator account** (typically a security tooling account).

### Policy Types

| Policy Type | What It Deploys/Enforces | Auto-Remediation |
|-------------|--------------------------|-----------------|
| **WAF** | Same WebACL to all ALBs / CloudFront / API GW | Auto-associates; removes non-compliant associations |
| **Security Group** | Enforce required SG rules; audit for overly permissive rules (0.0.0.0/0) | Add required rules; remove violating rules |
| **Network Firewall** | Deploy Network Firewall + routing to all in-scope VPCs | Creates firewall endpoints and route table entries |
| **Shield Advanced** | Subscribe all accounts; enable proactive engagement | Auto-subscribes new accounts joined to Org |
| **DNS Firewall** | Deploy Route 53 Resolver DNS Firewall rule groups to VPCs | Auto-associates rule groups |

### Key Distinctions

| Question | Right Answer |
|----------|--------------|
| Ensure all ALBs across 50 accounts have the same WAF | Firewall Manager WAF policy |
| Report which accounts have non-compliant SGs | Firewall Manager Security Group audit policy (reports and can auto-remediate) |
| Prevent developers from detaching WAF from their ALB | Firewall Manager — member accounts cannot modify a Firewall Manager-managed WebACL |
| New account added to Org — WAF must apply automatically | Firewall Manager with `Include all accounts under Organization` scope |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Centrally enforce WAF rules across entire organisation | Firewall Manager WAF policy (delegated admin) |
| Auto-subscribe all new accounts to Shield Advanced | Firewall Manager Shield Advanced policy |
| Detect EC2 instances with port 22 open to 0.0.0.0/0 across all accounts | Firewall Manager Security Group audit policy |
| Deploy identical Network Firewall config to every VPC in the Org | Firewall Manager Network Firewall policy |

---

# 🔐 DOMAIN 4: ENCRYPTION & DATA PROTECTION

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
| **AWS Managed** | Auto every year (365 days) | Cannot disable |

**Exam trap:** Rotating a CMK does NOT invalidate existing ciphertext. The old backing key is retained for decryption. To remove access to all data encrypted with a key → schedule key deletion.

### Multi-Region Keys
- Primary key in one region; replica keys in other regions.
- Same key ID, same key material — encrypted in one region can be decrypted in another.
- Use case: Global applications, cross-region disaster recovery, global client-side encryption.
- **Not the same as CRR** — key material is replicated, not the ciphertext.

### KMS Imported Key Material (BYOK)
- Bring your own key material generated outside AWS (HSM, custom crypto tool).
- You generate a key pair in KMS → KMS returns a public key + import token → You encrypt your key material with the public key → Upload ciphertext.
- **Key differences from standard CMK:**

| Attribute | Standard CMK | Imported Key Material |
|-----------|-------------|----------------------|
| **Auto-rotation** | Supported | ❌ NOT supported — must rotate manually by re-importing |
| **Key deletion** | 7–30 day waiting period | Can delete immediately by deleting the key material (makes key unusable instantly) |
| **Re-import** | N/A | Yes — re-import the same or different material into same key ID |
| **Key origin** | `AWS_KMS` | `EXTERNAL` |

- **Exam trap:** Questions about immediate, cryptographic erasure of data → delete imported key material → data instantly unreadable.
- **Exam trap:** Imported keys **cannot use automatic key rotation**.

### KMS External Key Store (XKS)
- Keys are stored and managed in your **own HSM or external key manager** — outside AWS.
- KMS operations are **proxied** to your external key manager via an **XKS proxy** you deploy.
- Key material **never enters AWS** — AWS only stores a reference to the external key.
- Use case: Highly regulated industries where keys must remain in customer-controlled hardware.
- **Latency tradeoff:** Every encrypt/decrypt call traverses the XKS proxy — higher latency than standard KMS.
- **Availability dependency:** If your XKS proxy is unreachable, encryption/decryption fails.

| Attribute | Standard KMS CMK | XKS Key |
|-----------|-----------------|---------|
| **Key material location** | AWS KMS HSMs | Your HSM / external key manager |
| **Revocation** | Disable/delete key | Remove proxy access immediately |
| **Compliance use case** | General | Keys outside AWS mandate |

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
- **short-lived cert use case:** Issue certs valid for hours/days to on-premises workloads via IAM Roles Anywhere — removes need to revoke; certs expire automatically.
- **Subordinate CA:** Create a hierarchy — Root CA (offline/HSM) → Subordinate CA (AWS Private CA) → end-entity certs.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Free TLS cert for ALB | ACM public certificate |
| mTLS between internal microservices | ACM Private CA |
| Certificate with exportable private key | ACM Private CA |
| Auto-renewing wildcard cert for CloudFront | ACM (DNS validation) |
| Short-lived certs for Zero-Trust on-prem access | ACM Private CA + IAM Roles Anywhere |

---

## CloudWatch Logs — Data Protection Policy

**What it is:** Detect and mask sensitive data patterns in CloudWatch log groups automatically — without application code changes.

### How It Works
- Enable a **Data Protection Policy** on a log group.
- Policy defines which data identifiers to detect (PII patterns: SSN, credit card, email, phone, passport, etc.).
- **Audit action:** Log masked data findings to S3 or Kinesis Firehose for compliance review.
- **Masking action:** Detected sensitive data is replaced with `[REDACTED]` in all reads by users without special permission.
- Only users with `logs:Unmask` IAM permission can see original data.

### Managed Data Identifiers
AWS provides 100+ built-in patterns: SSN, credit card numbers (Visa/MC/Amex), IBAN, passport numbers, drug enforcement, National ID numbers for multiple countries, email addresses, IP addresses, etc.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Prevent developers from seeing PII in application logs | CloudWatch Logs Data Protection Policy |
| Automatically redact credit card numbers in logs | CW Logs Data Protection Policy with financial identifiers |
| Audit what sensitive data is present in log groups | CW Logs Data Protection Policy → audit to S3 |

---

## SNS — Message Data Protection

**What it is:** Detect, audit, redact, or block sensitive data in SNS messages before delivery.

### Three Actions
| Action | Effect |
|--------|--------|
| **Audit** | Log message with sensitive data detected (to CloudWatch Logs or Kinesis) |
| **De-identify (mask)** | Replace sensitive data with `[REDACTED]` before delivering to subscriber |
| **Block** | Reject the Publish API call entirely — message never delivered |

- Uses the same 100+ AWS managed data identifiers as CloudWatch Logs Data Protection.
- Policy is attached to the SNS topic.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Prevent PII from flowing through an event-driven messaging pipeline | SNS message data protection policy |
| Block messages containing SSNs before delivery | SNS data protection — Block action |

---

## AWS Signer — Digital Code Signing

**What it is:** Managed code signing service that digitally signs code artifacts to verify publisher identity and detect tampering.

### Supported Signing Targets
| Target | Use |
|--------|-----|
| **AWS Lambda** | Sign deployment packages; verify code hasn't been tampered between deploy and execution |
| **IoT Devices** | Sign OTA firmware updates; IoT Greengrass verifies signature before executing |
| **Container Images** | Sign container images (notation-based signing) |

### How It Works
1. Create a signing profile in AWS Signer (selects signing algorithm and validity period).
2. Submit an artifact to sign → Signer generates a digital signature.
3. AWS Lambda validates signature at deployment using a Code Signing Config.
4. Lambda rejects unsigned or signature-mismatch deployments.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Ensure Lambda functions were not tampered before execution | AWS Signer + Lambda Code Signing Config |
| Verify firmware authenticity before deploying to IoT edge devices | AWS Signer + IoT Greengrass |
| Cryptographically verify who built and published a Lambda package | AWS Signer digital signature |

---

## Nitro System — Inter-Instance Encryption

**What it is:** Automatic hardware-level encryption of all traffic between EC2 Nitro instances within a VPC.

- Applies to: **C5n, C6gn, C6i, P4d, R6i**, and other Nitro-based instance types when communicating on supported enhanced networking.
- Encryption happens transparently at the hardware layer — **no application changes needed**.
- Uses AES-256-GCM with keys managed by Nitro hardware — customer has no key management overhead.
- **VPC stays fully encrypted even without VPN or TLS at the application layer**.
- EMR cluster inter-node encryption and EKS node-to-node traffic also benefit from this on Nitro instances.

| Attribute | Value |
|-----------|-------|
| Encryption algorithm | AES-256-GCM |
| Key management | AWS Nitro hardware (automatic) |
| Configuration needed | None — automatic on supported types |
| Performance impact | None — offloaded to Nitro hardware |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Encrypt all EC2 traffic within VPC without app changes | Use Nitro-based instance types — automatic hardware encryption |
| EMR compliance requires in-transit encryption between nodes | Enable Nitro-based instances + at-rest KMS encryption |

---

## AWS Client VPN — Remote Access Authentication

**What it is:** Managed OpenVPN service allowing remote users to securely access AWS and on-premises resources.

### Authentication Options
| Method | How It Works |
|--------|-------------|
| **Active Directory (Managed AD / AD Connector)** | Users authenticate with corporate AD credentials |
| **SAML 2.0 federated (IdP)** | Uses external IdP (Okta, Azure AD) via SAML — supports MFA at IdP layer |
| **Mutual TLS (certificate-based)** | Client presents X.509 certificate; ACM Private CA issues client certs |
| **Dual authentication** | Combine AD/SAML + mutual TLS for two-factor |

### Authorization
- **Authorization rules** define which CIDR ranges (AWS subnets, on-prem networks) each AD group or role can reach.
- Supports **connection logging** to CloudWatch Logs — logs each connection attempt, source IP, user ID.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Remote employees need VPN access using corporate SSO (Okta) | Client VPN with SAML 2.0 federation |
| Remote access with certificate-based auth (no password) | Client VPN with mutual TLS + ACM Private CA |
| Enforce MFA for remote VPN access | Client VPN with SAML IdP that enforces MFA |

---

## Elastic Load Balancing (ELB) — Security Policies (TLS Enforcement)

**What it is:** ELB security policies define which TLS versions and cipher suites the load balancer will accept from clients.

### Key Policies
| Policy | TLS Versions Allowed | Use Case |
|--------|---------------------|----------|
| **ELBSecurityPolicy-TLS13-1-3-2021** | TLS 1.3 only | Maximum security, newest clients |
| **ELBSecurityPolicy-TLS13-1-2-2021** | TLS 1.2 + 1.3 | Recommended — broad compatibility + security |
| **ELBSecurityPolicy-2016-08** | TLS 1.0–1.3 | Legacy — avoid for new deployments |
| **ELBSecurityPolicy-FS** | TLS 1.2 + forward secrecy cipher suites | PCI compliance; ECDHE for perfect forward secrecy |

- Apply security policies to **HTTPS and TLS listeners** on ALB/NLB.
- Policy selection must balance security requirements vs. client compatibility.
- **Forward secrecy (FS) policies** use ECDHE cipher suites — even if server private key is compromised later, past sessions cannot be decrypted.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Enforce TLS 1.2 minimum on public ALB | ELBSecurityPolicy-TLS13-1-2-2021 |
| PCI DSS requires forward secrecy | ELBSecurityPolicy-FS policy |
| Prevent TLS 1.0/1.1 connections | Apply modern security policy to ALB HTTPS listener |

---

## AWS DataSync — Secure Data Transfer

**What it is:** Managed data transfer service for moving data between on-premises storage (NFS/SMB), cloud storage (S3, EFS, FSx), and AWS.

### Security Features
| Feature | Details |
|---------|---------|
| **In-transit encryption** | TLS 1.2 for all data transferred over public internet |
| **At-rest encryption** | Honors destination service encryption (S3 SSE-KMS, EFS KMS) |
| **Integrity checks** | SHA-256 checksums verified end-to-end — detects data corruption |
| **Task filters** | Exclude specific file patterns to prevent sensitive data transfer |
| **Bandwidth throttling** | Limits max throughput to avoid impacting production workloads |
| **IAM execution role** | DataSync assumes an IAM role to write to S3/EFS/FSx |
| **VPC deployment** | Agent can use VPC endpoints — traffic never traverses public internet |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Securely migrate NFS share to EFS with integrity verification | DataSync with TLS + checksums |
| Limit DataSync from consuming all bandwidth | DataSync bandwidth throttling |

---

## EXAM TIPS & QUICK REFERENCE

### High-Priority Services
1. **IAM** — Policy evaluation, cross-account, federation, Verified Permissions
2. **KMS & Encryption** — CMKs, envelope encryption, key policies, imported keys
3. **Logging & Monitoring** — CloudTrail, GuardDuty protection plans, Config, Security Hub
4. **Infrastructure Protection** — SG/NACL, WAF, VPC Endpoints, Network Firewall
5. **Incident Response** — Playbooks, forensics, automated remediation, OpsCenter
6. **Governance** — SCPs, RCPs, Declarative Policies, Control Tower proactive controls

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
1. Explicit Deny (anywhere) → always DENY
2. Organization SCP → caps account-level permissions
3. IAM Permissions Boundary → caps identity-level permissions
4. Session Policy → further restricts assumed-role sessions
5. Identity-Based Policy → grants actions to the principal
6. Resource-Based Policy → grants/denies cross-account access
= Final decision: must be allowed at every applicable layer
```
> ⚠️ Permission Boundary comes BEFORE Identity Policy in evaluation — it limits what identity policies can grant, not the other way around.
> ⚠️ Resource-Based Policy context matters: for **same-account** access, a resource policy Allow alone is sufficient (step 3, not step 6). Step 6 reflects **cross-account** access, where BOTH the resource policy (in Account B) AND an identity policy (in Account A) must allow the action.

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
| S3 | SSE-S3/KMS/SSE-C | HTTPS | CloudTrail | CRR | Per GB |
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

## COMPLIANCE & REGULATORY FRAMEWORKS — AWS Control Mapping

> Exam questions appear as: "Company must meet [regulation] — which AWS controls satisfy requirement X?" Focus on which AWS service implements each requirement, not the framework definition itself.

### HIPAA — Key AWS Controls

| HIPAA Requirement | AWS Service & Implementation |
|-------------------|------------------------------|
| Encrypt PHI at rest | SSE-KMS with CMK on S3/RDS/EBS; KMS key policy restricts access |
| Encrypt PHI in transit | TLS enforced via ELB security policy; S3 bucket policy `aws:SecureTransport: false → Deny` |
| Audit access to PHI | CloudTrail Data Events on S3; CloudWatch Logs with metric filters |
| Access controls for PHI systems | IAM roles + least privilege; MFA enforced via IAM conditions |
| Agreement with cloud provider | Sign AWS Business Associate Agreement (BAA) via AWS Artifact |
| Breach detection | GuardDuty (threat detection) + Macie (sensitive data discovery in S3) |

### PCI DSS — Key AWS Controls

| PCI DSS Requirement | AWS Service & Implementation |
|---------------------|------------------------------|
| Req 1: Network segmentation of CDE | VPC private subnets + SGs + NACLs; VPC endpoints (no internet) |
| Req 3: Encrypt cardholder data at rest | SSE-KMS or client-side encryption with CMK |
| Req 7: Restrict access to cardholder data | IAM least privilege; `aws:PrincipalOrgID` on bucket policies |
| Req 10: Log and monitor all access | CloudTrail + VPC Flow Logs + Config continuous recording |
| Req 11: Vulnerability scanning | Amazon Inspector (automated CVE scanning) |
| Req 11: Pen testing | Customer responsibility; submit to AWS before testing |
| Enforce TLS + forward secrecy | ELB with `ELBSecurityPolicy-FS` (ECDHE ciphers, TLS 1.2+) |
| Compliance evidence collection | AWS Audit Manager (PCI DSS framework) |

### GDPR — Key AWS Controls

| GDPR Requirement | AWS Service & Implementation |
|------------------|------------------------------|
| Data residency (EU only) | SCP `Deny *` with `StringNotEquals aws:RequestedRegion` allowing only EU regions |
| Right to erasure | S3 object deletion + KMS CMK key deletion (cryptographic erasure — instant) |
| PII discovery and classification | Amazon Macie |
| DLP (prevent PII leaking through logs/messages) | CW Logs Data Protection Policy; SNS message data protection |
| Data processing audit trail | CloudTrail + AWS Audit Manager (GDPR framework) |
| Breach notification (72 hours) | GuardDuty finding → EventBridge → SNS → Security team |

### FedRAMP — Key AWS Controls

| FedRAMP Requirement | AWS Service & Implementation |
|---------------------|------------------------------|
| FIPS 140-2 Level 3 encryption | CloudHSM (Level 3); KMS (Level 2 only) |
| US-only data residency | AWS GovCloud regions (us-gov-west-1, us-gov-east-1) |
| Continuous monitoring | AWS Config + Security Hub (NIST SP 800-53 standard) |
| Privileged access management | IAM + MFA + CloudTrail + Session Manager (no SSH) |

### Compliance Service Quick-Reference

| Need | Correct AWS Service |
|------|---------------------|
| Collect evidence for auditors automatically | AWS Audit Manager |
| Download AWS compliance reports (SOC 2, ISO, PCI) | AWS Artifact |
| Continuous compliance checking with rules | AWS Config + Conformance Packs |
| Aggregate compliance status across 50 accounts | Security Hub (CIS/PCI/FSBP/NIST standards) |
| Immutable audit log that cannot be deleted | CloudTrail + S3 Object Lock (Compliance mode) |
| Verify what sensitive data exists in S3 | Amazon Macie |

---

## ADVANCED EXAM TRAPS — MULTI-SERVICE SCENARIOS

> Each scenario has two plausible answers. The wrong choice is what a real architect might reach for; the right choice is what AWS rewards on the exam.

### Trap 1: SCP vs Resource Control Policy (RCP)
**Question type:** "Prevent S3 bucket owners from disabling server-side encryption, even if they modify their own bucket policies."

| Control | What It Restricts |
|---------|-------------------|
| **SCP** | What IAM **principals** in the account can **call** (actions) |
| **RCP** | What **resources** in the account will **accept** (conditions on inbound requests) |

✅ **Right:** RCP on S3 denying `s3:PutObject` unless `s3:x-amz-server-side-encryption` is present — enforces encryption on the resource regardless of who calls it or what their identity policy says.
❌ **Wrong:** SCP only — SCPs restrict the calling principal but cannot enforce what the resource itself will accept from all principals.

---

### Trap 2: Firewall Manager vs CloudFormation StackSets for WAF
**Question type:** "Ensure every ALB across 40 accounts has the same WAF WebACL. Developers must not be able to remove it."

✅ **Right:** Firewall Manager WAF policy (delegated admin) — auto-deploys WebACL to all ALBs in scope, automatically remediates non-compliant resources, and member accounts cannot detach the managed WebACL.
❌ **Wrong:** CloudFormation StackSets — deploys WAF rules once, but does NOT prevent a developer from manually detaching the WebACL from their ALB after deployment.

---

### Trap 3: AWS RAM vs Resource-Based Policy for Sharing
**Question type:** "Share a Transit Gateway with 15 member accounts in the same Organisation."

✅ **Right:** AWS RAM — Transit Gateway does not support resource-based policies; RAM is the only way to share it cross-account.
❌ **Wrong:** Resource-based policy — Transit Gateways, VPC subnets, and Route 53 Resolver rules have no resource-based policy support.

**Decision rule:**
| Resource | Share via |
|----------|-----------|
| S3, KMS, Lambda, Secrets Manager, SNS | Resource-based policy |
| Transit Gateway, VPC subnet, License configs, Route 53 Resolver rules | AWS RAM |

---

### Trap 4: GuardDuty S3 Protection Must Be Explicitly Enabled
**Question type:** "GuardDuty is enabled across all accounts. Security team notices it is not detecting unusual S3 API access patterns."

✅ **Right:** Enable **GuardDuty S3 Protection** plan — S3 data event monitoring is OFF by default. Core GuardDuty only consumes CloudTrail management events, VPC Flow Logs, and DNS logs.
❌ **Wrong:** "Enable CloudTrail S3 Data Events separately" — GuardDuty pulls S3 data events directly when the protection plan is on; a separate trail is not required.

---

### Trap 5: Declarative Policies vs SCPs for Service Configuration Enforcement
**Question type:** "Enforce IMDSv2 (token-required) on all EC2 instances in every account, even if account admins try to override it."

✅ **Right:** **Declarative Policy** for EC2 metadata defaults — enforces a baseline service configuration that cannot be overridden at the account or instance level.
❌ **Wrong:** SCP denying `ec2:ModifyInstanceMetadataOptions` — this can be bypassed at launch via `--metadata-options` in the RunInstances call, and does not address existing instances.

---

### Trap 6: KMS Cross-Account Encrypted Snapshot
**Question type:** "Share an EBS snapshot encrypted with your CMK to Account B. Account B needs to create volumes from it."

✅ **Right:** Share snapshot + add Account B to the **source CMK key policy** (`kms:ReEncrypt*`, `kms:CreateGrant`, `kms:DescribeKey`) → Account B copies the snapshot re-encrypting with their own CMK → creates volumes from the copy.
❌ **Wrong (security anti-pattern — explicitly marked wrong):** Creating an unencrypted intermediate snapshot to share it.

---

### Trap 7: Security Hub vs Amazon Detective for Incident Investigation
**Question type:** "A GuardDuty finding shows EC2 communicating with a known cryptocurrency mining endpoint. Team needs to trace the timeline and identify the attack path."

| Service | Purpose |
|---------|---------|
| **Security Hub** | Aggregates findings; compliance status; automation rules — not investigation |
| **Amazon Detective** | ML graph-based investigation — traces relationships between findings, IPs, roles, and time |

✅ **Right:** Amazon Detective — root cause analysis and investigation of an existing finding.
❌ **Wrong:** Security Hub — aggregates and reports findings but does not help trace what happened or why.

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
- **Problem:** EC2 snapshots encrypted with a customer-managed KMS key cannot be directly shared without key access
- **Correct solution:**
  1. Share the encrypted snapshot with the target account (`ModifySnapshotAttribute`)
  2. Add the target account to the **source CMK key policy** so it can use the key
  3. Target account copies the snapshot, re-encrypting with their own CMK
> ⚠️ Creating an **unencrypted intermediate snapshot** is a security anti-pattern — the exam marks this WRONG

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

## EXAM QUESTION PATTERNS

### Pattern 1: "Which service provides..."
- **Data discovery?** → Macie
- **Threat detection?** → GuardDuty
- **Configuration tracking?** → Config
- **API logging?** → CloudTrail
# 📊 DOMAIN 5: LOGGING, MONITORING & THREAT DETECTION

---

## Logging Service Decision Table

The most commonly confused domain on the exam. Match the requirement to the correct service before reading answer options.

| Requirement | Correct service | Common wrong answer |
|------------|----------------|--------------------|
| Record AWS API calls | CloudTrail | CloudWatch |
| Application / OS / container logs | CloudWatch Logs | CloudTrail |
| Real-time threat detection | GuardDuty | CloudTrail |
| Check resource configuration compliance | AWS Config | GuardDuty |
| Aggregate security findings across accounts | Security Hub | CloudWatch |
| Centralise logs into a queryable security data lake | Amazon Security Lake | S3 alone |
| Stream and deliver logs in real-time | Kinesis Firehose | CloudWatch Logs Insights |
| SQL query on CloudTrail (lowest overhead) | CloudTrail Lake | S3 + Athena |
| Investigate root cause of a finding | Amazon Detective | Security Hub |

> ⚠️ **GuardDuty ≠ compliance. Config ≠ threat detection. CloudWatch ≠ aggregation.** These are the three pairs the exam exploits most.

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

### GuardDuty Protection Plans (Must Be Explicitly Enabled)
GuardDuty core protects EC2 and IAM. The following are **add-on protection plans that are OFF by default** and must be enabled separately:

| Protection Plan | What It Detects | Data Source Added |
|----------------|-----------------|-------------------|
| **S3 Protection** | Unusual S3 access patterns, public exposure, policy changes | CloudTrail S3 Data Events |
| **EKS Audit Log Monitoring** | Suspicious Kubernetes API calls, privilege escalation in K8s | EKS Audit Logs |
| **EKS Runtime Monitoring** | Container runtime threats (process injection, crypto mining) | EKS Runtime Agent |
| **Lambda Network Activity Monitoring** | Lambda functions calling malicious endpoints or unusual IPs | Lambda Network Logs |
| **RDS Protection** | Unusual login patterns, credential scanning, brute force | RDS Login Events |
| **EC2 Malware Protection** | Scan EBS volumes for malicious files without agent | EBS snapshot scan |
| **EC2/ECS Runtime Monitoring** | Process executions, file access, network inside EC2 containers | Runtime Agent |

**Exam trap:** Enabling GuardDuty does NOT automatically protect S3, EKS, Lambda, or RDS — you must explicitly enable each protection plan.

### Extended Threat Detection
- Correlates multiple findings across multiple data sources to detect sophisticated, multi-stage attacks.
- Example: Detects when an IAM credential theft leads to lateral movement then S3 exfiltration — connects events as a single attack sequence.
- Generates a summary finding in addition to individual findings.

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

## AWS Transit Gateway Flow Logs

**What it is:** Capture metadata for traffic passing through a Transit Gateway — covers **east-west traffic between VPCs** that VPC Flow Logs per-VPC cannot aggregate.

### Key Facts
- Same format as VPC Flow Logs (version 2+ fields).
- Captures traffic at the **Transit Gateway attachment level** — includes VPC-to-VPC, VPN-attached, and Direct Connect-attached traffic.
- Destinations: **S3**, **CloudWatch Logs**, **Kinesis Data Firehose**.
- Use case: Centralized visibility for hub-and-spoke architectures; detect lateral movement between VPCs.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Monitor all inter-VPC traffic in a hub-and-spoke architecture | Transit Gateway Flow Logs |
| VPC Flow Logs per VPC — too many to manage for 50 VPCs | Enable Transit Gateway Flow Logs centrally |
| Detect east-west lateral movement between VPCs | Transit Gateway Flow Logs |

---

## Amazon Route 53 Resolver — DNS Query Logging

**What it is:** Logs all DNS queries made by resources in your VPC to Route 53 Resolver — enables detection of DNS-based attacks and exfiltration.

### What Gets Logged
- DNS query name (domain requested)
- Query type (A, AAAA, MX, TXT, etc.)
- Response code (NOERROR, NXDOMAIN, SERVFAIL)
- VPC ID and source IP of querying instance
- Timestamp

### DNS Security Use Cases
| Threat | Detection Method |
|--------|--------------------|
| **DNS tunneling** | Long TXT record queries to unusual domains |
| **DNS exfiltration** | High-frequency queries encoding data in subdomains |
| **C2 via DNS** | Queries to known malicious domains (blocked by Route 53 Resolver DNS Firewall) |
| **Domain generation algorithm (DGA)** | GuardDuty Threat Intelligence on DNS queries |

### Route 53 Resolver DNS Firewall
- Block or allow DNS queries based on **domain lists** (AWS managed + custom).
- Integrates with Route 53 Resolver query logging.
- Block categories: malware, phishing, botnet C2, cryptocurrency mining.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Detect DNS exfiltration from EC2 instances | Route 53 Resolver DNS Query Logging + GuardDuty |
| Block DNS queries to known malicious domains | Route 53 Resolver DNS Firewall |
| GuardDuty finding: EC2 contacting known C2 via DNS | GuardDuty reads Resolver query logs (no config needed) |

---

## API Gateway — Logging Configuration

**What it is:** Two logging mechanisms for API Gateway — execution logs (debugging) and access logs (audit).

### Two Log Types
| Log Type | What It Captures | Destination |
|----------|-----------------|-------------|
| **Execution Logs** | Detailed request/response flow, auth decisions, Lambda integration errors | CloudWatch Logs |
| **Access Logs** | Who called the API (IP, auth token, status), in custom or CLF/JSON/XML format | CloudWatch Logs or Kinesis Firehose |

### Common Exam Trap — CloudWatch Logs Role
- To enable execution or access logging, the API Gateway stage must have a **CloudWatch Logs role ARN** configured at the **account level** (not just stage level).
- Troubleshooting: If API Gateway logs are not appearing in CloudWatch → check the account-level CW Logs role ARN is set.

### Access Log Fields (key ones)
`$context.identity.sourceIp`, `$context.requestId`, `$context.httpMethod`, `$context.resourcePath`, `$context.status`, `$context.authorizer.principalId`

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| API Gateway logs not appearing in CloudWatch | Check account-level CloudWatch Logs role ARN is configured |
| Audit every API call to a REST API including caller identity | Enable API Gateway access logs with authorizer fields |
| Debug Lambda integration errors via API Gateway | Enable API Gateway execution logs |
| Monitor API request rates and errors | CloudWatch metrics (automatic) + API Gateway access logs |

---

## Amazon CloudFront — Security Logging

### Two Log Types
| Type | Delivery | Latency | Use Case |
|------|----------|---------|----------|
| **Standard (Access) Logs** | S3 bucket | Minutes to hours | Long-term audit, compliance |
| **Real-Time Logs** | Kinesis Data Streams | <1 second | Live traffic monitoring, WAF correlation |

### Standard Log Fields (key)
`cs-uri-stem`, `sc-status`, `cs-method`, `x-edge-location`, `x-forwarded-for`, `cs(User-Agent)`, `cs-uri-query`, `x-edge-result-type` (Hit/Miss/Error)

### CloudFront Security Features
| Feature | Description |
|---------|-------------|
| **Origin Access Control (OAC)** | Restrict S3 origin to CloudFront only — S3 denies all direct requests. Successor to OAI |
| **Custom Headers from CloudFront** | Add a secret header (e.g., `X-CF-Secret`) → origin ALB only accepts requests with this header |
| **Field-Level Encryption** | Encrypt specific POST fields (e.g., credit card) at the edge; only specified app components can decrypt |
| **Geographic Restrictions** | Whitelist/blacklist countries at the CloudFront distribution level |
| **Signed URLs / Signed Cookies** | Time-limited access to specific content; signed with a CloudFront key pair |
| **HTTPS enforcement** | Redirect HTTP to HTTPS or reject HTTP entirely at the distribution |
| **WAF integration** | Attach AWS WAF to CloudFront for global L7 protection |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Restrict S3 content to only CloudFront requests | S3 Origin Access Control (OAC) + bucket policy |
| Block requests without secret header at origin | CloudFront custom origin header + ALB rule |
| Encrypt credit card field before it reaches origin | CloudFront field-level encryption |
| Block users from specific countries | CloudFront geographic restriction |
| Real-time log correlation with WAF events | CloudFront real-time logs → Kinesis |

---

## Amazon Managed Grafana — Security Deep Dive

**What it is:** Fully managed Grafana service for operational dashboards and metrics visualization from multiple AWS data sources and security monitoring platforms.

### Authentication and Access
| Method | Details |
|--------|---------|
| **IAM Identity Center (SSO)** | Primary auth method — federate with corporate IdP |
| **SAML 2.0** | Direct SAML federation for orgs not using IAM Identity Center |

### Data Source Permissions
- Each data source connection uses **IAM roles** — Grafana assumes a service role per data source.
- Data source permissions can be scoped so teams only see specific dashboards or data sources.
- Supports: CloudWatch, OpenSearch, Prometheus, X-Ray, Timestream, Athena, and 3rd-party SIEMs.

### Security Monitoring Use Cases
- Visualize **Security Hub findings** over time as dashboards.
- Build SOC dashboards from **Amazon Security Lake** OCSF data (via Athena).
- Monitor **GuardDuty** finding trends and severity distributions.
- Correlate **CloudWatch metrics** with security events.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Centralized security dashboard across all accounts | Amazon Managed Grafana + Security Hub / Security Lake data sources |
| SOC team needs real-time security metrics with SSO | Managed Grafana + IAM Identity Center |

---

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

## Incident Response — Six-Phase Framework

```
PREPARATION → DETECTION → CONTAINMENT → ERADICATION → RECOVERY → LESSONS LEARNED
```

| Phase | Key Activities |
|-------|---------------|
| **1. Preparation** | IR runbooks, IAM break-glass roles, forensic VPC pre-built, S3 evidence bucket, SNS alerts configured |
| **2. Detection** | GuardDuty finding, CloudTrail Insights, Security Hub score drop, CloudWatch alarm, manual alert |
| **3. Containment** | Isolate SG, revoke IAM credentials, block IP at WAF/NACL/Network Firewall, quarantine instance |
| **4. Eradication** | Remove malware, patch vulnerability, rotate all affected credentials |
| **5. Recovery** | Restore from clean backup/AMI, validate integrity, re-enable services incrementally |
| **6. Lessons Learned** | Root cause analysis, update runbooks, improve detection rules, file AWS security report |

### Containment Strategies
| Resource Type | Containment Method |
|--------------|-------------------|
| **Network (block inbound)** | Modify Security Group — deny all inbound |
| **Network (block subnet)** | NACL deny — overrides SG; apply to whole subnet |
| **Network (VPC level)** | Network Firewall drop rule or WAF block rule |
| **Route isolation** | Modify route table — remove internet gateway route |
| **Identity** | `PutUserPolicy` inline Deny with `aws:TokenIssueTime` condition |
| **IAM key** | Deactivate access key via `UpdateAccessKey` |
| **IAM Identity Center** | Revoke active sessions in Identity Center console |
| **EC2** | Detach instance profile (IAM role), isolate with empty SG |

---

## Systems Manager OpsCenter — Operational Issue Tracking

**What it is:** Centralized dashboard for viewing, investigating, and resolving **OpsItems** — operational issues generated by AWS services.

### Key Concepts
| Concept | Description |
|---------|-------------|
| **OpsItem** | A data record representing an issue — includes details, related resources, runbooks, investigation data |
| **Auto-creation** | EventBridge rules, CloudWatch alarms, Config rules, and Security Hub findings can auto-create OpsItems |
| **Runbook integration** | SSM Automation runbooks linked to OpsItems for one-click remediation |
| **Cross-account aggregation** | Organizations integration — aggregate OpsItems from all member accounts in admin account |
| **Deduplication** | Similar OpsItems grouped to avoid alert storm |

### Security Use Cases
- GuardDuty finding → EventBridge rule → creates OpsItem with remediation runbook linked.
- Config rule violation → auto-creates OpsItem with affected resource details.
- Centralized view of all security incidents across Org accounts.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Central tracking of security incidents across all AWS accounts | Systems Manager OpsCenter + Organizations |
| Auto-create ticket when GuardDuty finding fires | EventBridge rule → SSM OpsCenter OpsItem |
| One-click remediation from incident ticket | OpsCenter with linked SSM Automation runbook |

---

## AWS Resilience Hub — RTO/RPO Assessment

**What it is:** Analyzes application resilience, identifies resiliency gaps, and provides recommendations to meet RTO (Recovery Time Objective) and RPO (Recovery Point Objective) targets.

### How It Works
1. **Import application definition** from CloudFormation stack, Resource Groups, AppRegistry, or Terraform.
2. **Set resiliency policy** — define RTO and RPO targets per disruption type (AZ outage, hardware failure, region outage).
3. **Run assessment** — Resilience Hub evaluates alarms, backups, and recovery procedures.
4. **Recommendations** — provided if RTO/RPO targets are not met (e.g., enable Multi-AZ, add backup policy).
5. **Score** — Resiliency score (0–100) tracks improvement over time.

### Integration with AWS Backup
- Resilience Hub automatically checks if resources have compliant backup policies meeting the RPO targets.
- Creates alarms and runbooks linked to AWS Backup for recovery.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Verify application can recover within defined RTO/RPO | AWS Resilience Hub assessment |
| Company needs resiliency score and gap analysis | AWS Resilience Hub |
| Automate resiliency assessment in CI/CD pipeline | Resilience Hub API in CodePipeline |

---

## AWS Fault Injection Service (FIS) — Chaos Engineering

**What it is:** Managed chaos engineering service for running controlled fault injection experiments to test system resilience.

### Core Concepts
| Concept | Description |
|---------|-------------|
| **Experiment Template** | Defines targets, actions (faults), stop conditions, and IAM role |
| **Actions** | Inject CPU stress, network latency, AZ outage, EC2 termination, ECS task kill, RDS failover |
| **Targets** | EC2 instances, ECS tasks, EKS nodes, RDS instances, Spot instances — filtered by tags or IDs |
| **Stop Conditions** | CloudWatch alarm threshold — FIS automatically halts experiment if alarm fires |
| **Blast Radius** | Use filters and percentage-based targeting to limit impact |

### Security Considerations
- FIS requires an **IAM role** with permissions to perform each fault action.
- Stop conditions using CloudWatch alarms limit damage from runaway experiments.
- Experiments can be run in a **non-production VPC** or staging environment first.
- All experiment actions are logged in **CloudTrail**.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Test if app survives AZ failure without real outage | AWS FIS experiment template with AZ disruption action |
| Safely limit chaos experiment blast radius | FIS stop conditions on CloudWatch alarm |
| Validate RTO after chaos injection | FIS + AWS Resilience Hub assessment post-experiment |

---

## Amazon Application Recovery Controller (ARC)

**What it is:** Enables continuous readiness checking and fast, safe failover for multi-region, multi-AZ applications.

### Two Capabilities
| Capability | Purpose |
|-----------|---------|
| **Readiness Checks** | Continuously verify that recovery resources (replicas, backups, routing) are sufficient to handle failover |
| **Routing Controls** | Manual or automated switches to redirect traffic during failover (backed by Route 53 health checks) |

### Safety Rules on Routing Controls
- Define constraints on routing control state changes — prevent unsafe failovers.
- Example: "At least 1 cell must remain active" — prevents accidentally routing 0% traffic.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Verify multi-region readiness before a failover drill | ARC Readiness Checks |
| Rapid manual failover to DR region with safety guardrails | ARC Routing Controls + Safety Rules |

---

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

1. **IAM** ← Study HARD
   - Policy evaluation logic (all 6 layers)
   - Cross-account access + trust relationships
   - Verified Permissions (Cedar), Roles Anywhere, Directory Service
   
2. **Infrastructure Security**
   - SG vs NACL differences (stateful vs stateless)
   - VPC endpoints, WAF, Network Firewall
   - Session Manager, EC2 Image Builder, Patch Manager
   
3. **Data Protection**
   - KMS key policy + IAM (both required)
   - S3 encryption + Object Lock
   - Imported keys, XKS, Signer, Nitro encryption
   
4. **Detection & Logging**
   - CloudTrail (data events off by default), GuardDuty protection plans
   - CloudWatch Data Protection, SNS Data Protection
   - Transit GW logs, Route 53 Resolver logs
   
5. **Incident Response**
   - Automation: EventBridge + Lambda
   - Forensics playbooks, OpsCenter, Resilience Hub, FIS
   - Six-phase IR framework + containment strategies
   
6. **Governance** ← New SCS-C03 content
   - RCPs (restrict resources) + Declarative Policies (enforce config)
   - Control Tower proactive controls (CloudFormation Hooks)
   - CloudFormation Guard, Service Catalog, RAM

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

## FAST RECALL — EXAM HEURISTICS

### Layered Security Rule
When a question asks for the **BEST**, **MOST SECURE**, or **MINIMUM RISK** solution, a single-control answer is almost always wrong. The correct answer combines at least two layers:

| Layer | Controls |
|-------|----------|
| **Network** | VPC Endpoint, Security Group, NACL, Network Firewall |
| **Identity** | IAM policy, Resource policy, SCP, Permission Boundary |
| **Monitoring** | GuardDuty, Config, CloudTrail |

> ❌ "Use a bucket policy" alone → wrong
> ✅ "Use VPC Endpoint + bucket policy restricting `aws:SourceVpc`" → right

### GuardDuty vs Inspector Discriminator
Both detect security issues but answer completely different questions:

| Service | Mental Model | Trigger Keywords |
|---------|-------------|------------------|
| **GuardDuty** | Behavior-based — ML + log analysis detects *what is happening* | compromise, anomaly, suspicious activity, threat, C2, exfiltration |
| **Inspector** | Vulnerability-based — CVE database scan detects *what could happen* | vulnerability, CVE, outdated package, unpatched, container image scan |

> ❌ Trap: Both appear as "security detection" options. Split on **behavior vs. vulnerability**.

### Cross-Account Mechanism Discriminator
AssumeRole is NOT the only cross-account mechanism — but it's the right one in most exam scenarios:

| Situation | Correct mechanism |
|-----------|------------------|
| Application/service in Account A needs to act in Account B | STS `AssumeRole` — creates session in Account B |
| Account B principal reads/writes a specific resource in Account A | Resource-based policy on the resource (S3, KMS, SQS, Lambda) — no role assumption needed |
| S3 cross-account object access | S3 bucket policy (Account A) + IAM identity policy (Account B) — both required |

> ❌ Trap: "Cross-account" doesn't always mean AssumeRole. If the question is about accessing a named resource (S3 bucket, KMS key), a resource policy may be the simpler correct answer.

### NAT Gateway vs VPC Endpoint
The exam trap is treating them as equivalent private-access options:

| Feature | NAT Gateway | VPC Endpoint |
|---------|------------|-------------|
| **What it provides** | Outbound internet routing for private instances | Private path to specific AWS services |
| **Traffic destination** | Any internet host | Only the target AWS service |
| **Policy enforcement** | ❌ None — routing only | ✅ Endpoint policy restricts which API actions and resources |
| **Data stays in AWS?** | ❌ No — goes to internet | ✅ Yes — never leaves AWS network |
| **Cost** | Per GB processed | Gateway (S3/DynamoDB) = free; Interface = hourly |

> Rule: NAT = routing. VPC Endpoint = routing **plus** security control. When the question says "private access" or "no internet" → always Endpoint.

---

### Data Exfiltration Prevention Pattern (S3)
To prevent data leaving AWS via S3, you must combine **both**:

1. **VPC Endpoint** (Gateway type for S3) — force all S3 traffic through private network
2. **Bucket policy** denying requests that do NOT come through the endpoint:
   ```json
   { "Effect": "Deny", "Principal": "*", "Action": "s3:*", "Resource": "...",
     "Condition": { "StringNotEquals": { "aws:SourceVpce": "vpce-xxxx" } } }
   ```

> ❌ Bucket policy alone — a user outside the VPC can still access S3 directly
> ❌ Security Groups — **do not apply to S3** (S3 is a regional service, not VPC-attached)
> ✅ VPC Endpoint + `aws:SourceVpce` bucket policy condition = enforced private-only access

---

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
- **Read the LAST sentence of the question first** — it tells you the intent (detect/prevent/audit/encrypt) before you're distracted by the scenario details
- Read questions carefully (trick answers are common)
- Eliminate obvious wrong answers first
- Look for "best practice" vs "works but suboptimal"
- Remember: AWS prefers managed services over manual work
- When in doubt, choose encryption or least privilege
- AWS Security Specialty often tests knowledge of when NOT to use a service

---


# 🏛️ DOMAIN 7: SECURITY GOVERNANCE & COMPLIANCE

---

## AWS Organizations Policy Types — Overview

SCS-C03 covers **five policy types** in AWS Organizations, each with a distinct scope and purpose:

| Policy Type | Affects | Purpose |
|-------------|---------|---------|
| **SCP** (Service Control Policy) | IAM principals in member accounts | Restrict what principals can do |
| **RCP** (Resource Control Policy) | AWS resources in member accounts | Restrict what any principal can do to resources |
| **Declarative Policy** | Service configurations (EC2 attributes) | Enforce desired resource configuration state |
| **Tag Policy** | Resource tagging | Enforce consistent tag keys and values |
| **AI Service Opt-Out Policy** | AI services (Comprehend, Rekognition, etc.) | Control whether AWS can use your data for model training |

---

## Service Control Policies (SCPs) — Recap

- Restrict **what IAM principals (users/roles) can do** in member accounts.
- Applied to OUs or accounts — evaluated BEFORE IAM policies.
- **Do NOT apply to management account** and **do NOT apply to service-linked roles**.
- **Never grant permissions on their own** — only restrict.

---

## Resource Control Policies (RCPs) — NEW SCS-C03 Topic

**What it is:** A new Organizations policy type that restricts what **any principal (including external principals)** can do to resources in your organization's member accounts.

### SCP vs RCP
| Aspect | SCP | RCP |
|--------|-----|-----|
| **What it restricts** | **Principals** — limits what identities can do | **Resources** — limits what can be done to resources |
| **Applies to external principals** | ❌ No — only org principals | ✅ Yes — blocks even cross-account external access |
| **Use case** | Prevent org principals from using certain services | Prevent anyone (including external accounts) from accessing your resources in insecure ways |
| **Example** | Deny ec2:TerminateInstances across all accounts | Require all S3 PutObject requests to use SSE-KMS |

### What RCPs Can Do
- Require S3 objects to only be accessed over HTTPS (deny HTTP).
- Require SSE-KMS on all S3 PutObject calls within the org.
- Prevent any external principal from accessing specific resource types.
- Complement SCPs: SCPs prevent principals FROM doing things; RCPs prevent things FROM BEING DONE to resources.

### RCP Example — Enforce HTTPS for all S3 Access
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyNonHTTPS",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": "*",
    "Condition": {
      "Bool": { "aws:SecureTransport": "false" }
    }
  }]
}
```

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Prevent external principals from accessing org S3 buckets without encryption | RCP |
| Ensure all S3 buckets in org cannot be accessed via HTTP | RCP denying insecure transport |
| SCP blocks principals from doing X — what blocks external access to resources? | RCP (Resource Control Policy) |
| New SCS-C03 policy that limits resource-level access org-wide | RCP |

---

## Declarative Policies

**What it is:** Enforce a desired **configuration state** for services regardless of any API calls — configuration is enforced continuously, not just at creation time.

### Current Scope
- **EC2 attributes**: IMDSv2 enforcement, EBS default encryption, allowed VPC endpoints, instance metadata token hop limit.

### How They Work
- Unlike SCPs (which say "you can't call this API"), Declarative Policies say "this attribute **must be** this value on all resources in these accounts."
- If an EC2 instance is launched without IMDSv2, the Declarative Policy **enforces** the setting at launch — the instance gets IMDSv2 automatically.

### Key Use Cases
| Use Case | What to Enforce |
|----------|----------------|
| Enforce IMDSv2 on all EC2 instances | `aws:ec2:DefaultImdsV2` = `required` |
| Require EBS encryption by default | `aws:ec2:EbsDefaultKmsKeyId` + encryption enabled |
| Restrict allowed instance metadata hop limit | Prevents SSRF escalation via IMDSv1/IMDSv2 tokens |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Ensure IMDSv2 is required across ALL EC2 in org regardless of launch parameters | Declarative Policy |
| What's the difference between SCP denying IMDSv1 vs Declarative Policy? | SCP denies the API call; Declarative Policy **enforces** the attribute state at the platform level |

---

## AI Service Opt-Out Policies

**What it is:** Controls whether AWS AI services (Amazon Comprehend, Rekognition, Kendra, CodeGuru, Translate, etc.) can use your content to improve their models.

- By default, AWS may use your content inputs/outputs for service improvements.
- Opt-Out Policy at the Organization level **prevents this for all member accounts**.
- Policy is set per-service or for all opt-outable AI services at once.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Prevent AWS from training AI models with your Rekognition or Comprehend data | AI Service Opt-Out Policy |
| Org-wide AI data privacy compliance requirement | AI Service Opt-Out Policies in AWS Organizations |

---

## Tag Policies

**What it is:** Enforce consistent tag key capitalization and allowed values across all AWS resources in the organization.

- **Tag compliance** — identify resources that don't follow tagging standards.
- Can enforce: specific keys must exist, specific case (`Environment` not `environment`), specific allowed values (`prod`, `dev`, `staging`).
- **Does NOT auto-tag** — it enforces when tags are applied.
- Reports non-compliant resources via the **Tag Editor** in Resource Groups console.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Ensure all EC2 instances have `CostCenter` and `Environment` tags with approved values | Tag Policies |
| Standardize tag capitalization across all accounts in org | Tag Policies (enforces key case) |

---

## AWS Control Tower

**What it is:** Orchestrates setup of a secure multi-account AWS environment (Landing Zone) following AWS best practices.

### Three Types of Controls (Guardrails)
| Type | Mechanism | Example |
|------|-----------|---------|
| **Preventive** | SCP (blocks API calls) | "Disallow creation of access keys for root user" |
| **Detective** | Config Rule (detects violation) | "Detect if MFA is not enabled for root" |
| **Proactive** | CloudFormation Hooks (checks before provisioning) | "Check resource config BEFORE it's created" |

### Proactive Controls (New Feature)
- **CloudFormation Hooks** — intercepts CloudFormation resource creation/update BEFORE the resource is provisioned.
- If the check fails, the CloudFormation deployment is rejected.
- Use case: Reject any CloudFormation stack that creates an unencrypted S3 bucket or opens port 22 to 0.0.0.0/0.
- Proactive controls are applied via Control Tower — no custom Hook development required.

### Landing Zone Components
| Component | Purpose |
|-----------|---------|
| **Log Archive account** | Central S3 bucket for CloudTrail + Config logs from all accounts |
| **Audit account** | Security team access for cross-account read-only review |
| **Management account** | Root of org — runs Control Tower |
| **Enrollment** | Onboard existing accounts into Control Tower governance |

### Recommended OU Structure
```
Root
├── Security OU
│   ├── Log Archive Account
│   └── Audit Account
├── Infrastructure OU
│   ├── Network Account (Transit Gateway, DNS)
│   └── Shared Services Account
├── Workloads OU
│   ├── Production OU
│   │   └── Production Accounts
│   └── Development OU
│       └── Developer Accounts
└── Sandbox OU
    └── Experimental Accounts
```

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Prevent a new CloudFormation resource from being created without encryption | Control Tower proactive controls (CloudFormation Hooks) |
| Detect MFA not enabled on root account across all accounts | Control Tower detective control (Config rule) |
| Prevent all accounts from disabling CloudTrail | Control Tower preventive control (SCP) |
| Newly created AWS accounts auto-enrolled with security controls | Control Tower Account Factory |

---

## Root User Centralized Management

**What it is:** Organizations feature in new accounts that **removes root user credentials** from member accounts — preventing root access entirely from member accounts.

### Key Facts
- In new AWS accounts created through Organizations, root user credentials can be **deleted** — only the management account admin can perform root-level tasks on behalf of member accounts.
- **Break-glass access:** Management account can grant temporary root access to a member account for password reset or account-level operations.
- Prevents scenarios where a member account root is compromised and used to bypass SCPs.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Prevent member account users from ever using root credentials | Centralized root access management via Organizations |
| Perform account-level task (close account, change payment) on member account | Management account performs it — member account root not needed |

---

## AWS CloudFormation Guard (cfn-guard)

**What it is:** Open-source, policy-as-code tool for validating CloudFormation templates (and Terraform, Kubernetes manifests) against custom security rules.

### Key Features
- Write rules in the **Guard DSL** (domain-specific language) — declarative rule format.
- Validates JSON/YAML templates BEFORE deployment — shift-left security in CI/CD.
- Integrates with **CodePipeline, CodeBuild** for automated policy checks.
- Works with **AWS Control Tower proactive controls** — predefined Guard rules for AWS security standards.

### Guard Rule Example
```
# Ensure S3 buckets have versioning enabled
rule S3_BUCKET_VERSIONING_ENABLED {
  AWS::S3::Bucket {
    Properties.VersioningConfiguration.Status == "Enabled"
  }
}
```

### cfn-lint
- Separate linting tool for CloudFormation templates — checks for syntax errors, deprecated properties, and valid resource configurations.
- cfn-guard = **policy enforcement** (security rules) | cfn-lint = **syntax/correctness checks**.

| Tool | Purpose | Output |
|------|---------|--------|
| **cfn-guard** | Policy-as-code rule evaluation | Pass/Fail with rule violation details |
| **cfn-lint** | Template syntax + correctness | Warnings and errors on template structure |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Reject CloudFormation templates that create public S3 buckets in CI/CD | cfn-guard with S3 public access rule |
| Validate templates for correct syntax before deploying | cfn-lint |
| Policy-as-code for infrastructure compliance | cfn-guard |

---

## AWS Service Catalog

**What it is:** Centrally manage and distribute approved AWS infrastructure as "products" — ensures teams only deploy compliant, pre-approved configurations.

### Core Concepts
| Concept | Description |
|---------|-------------|
| **Portfolio** | Collection of products — shared with specific IAM users/groups/roles or OUs |
| **Product** | A CloudFormation template (defines what gets deployed) |
| **Launch Constraint** | IAM role that deploys the product — end user doesn't need direct CloudFormation permissions |
| **Notification Constraint** | SNS topic notified on product events |
| **TagOption Library** | Enforce consistent tags on all launched products |
| **Self-Service Portal** | End users browse and launch approved products without IT tickets |

### Security Pattern
- Security team creates products (locked-down CloudFormation stacks with KMS, no public access).
- Shares the portfolio with developer OUs.
- Developers launch products via Service Catalog — they get pre-approved infrastructure.
- Launch constraint IAM role has permissions developers don't — prevents privilege escalation.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Allow developers to provision VPCs without direct CloudFormation permissions | Service Catalog launch constraint with IAM role |
| Ensure all provisioned infrastructure meets security standards | Service Catalog with pre-approved products |
| Enforce mandatory tags on all Service Catalog launches | TagOption Library |

---

## AWS Resource Access Manager (RAM)

**What it is:** Share AWS resources across accounts within an Organization without copying them.

### Shareable Resources (Key for Exam)
| Resource | Use Case |
|----------|----------|
| **VPC Subnets** | Share a subnet with other accounts — resources in those accounts can launch into it |
| **Transit Gateway** | Share central TGW across accounts — no need for peering |
| **Route 53 Resolver Rules** | Share DNS forwarding rules to all accounts |
| **Prefix Lists** | Share managed IP prefix lists for use in SG rules |
| **AWS Private CA** | Share a Private CA to issue certs in other accounts |
| **License Manager Configurations** | Share software licenses |

### Key Points
- Resource stays in owner's account — other accounts get a reference.
- Works **within AWS Organizations** or with specific external account IDs.
- **No data copying** — the resource itself is shared, not duplicated.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| One VPC subnet used by EC2 in multiple accounts (hub model) | RAM — share subnet to member accounts |
| One Transit Gateway for all 50 accounts | RAM — share TGW |
| Share Private CA to issue certs across org | RAM — share Private CA |

---

## AWS Audit Manager

**What it is:** Continuously collect **evidence** for compliance audits — automates manual evidence-gathering tasks.

### How It Works
- **Frameworks** — pre-built or custom: HIPAA, PCI DSS, CIS, GDPR, NIST, SOC 2.
- **Controls** — each framework has controls (e.g., "MFA enabled on all IAM users").
- **Evidence** — auto-collected from Config, CloudTrail, Security Hub, IAM Access Analyzer.
- **Assessment Reports** — generate audit-ready reports with evidence attachments.
- Works across accounts with **delegated admin in Organizations**.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Continuously collect evidence for PCI DSS audit | AWS Audit Manager with PCI DSS framework |
| Automate compliance evidence collection to reduce audit prep time | Audit Manager |

---

## AWS Artifact

**What it is:** On-demand access to AWS compliance reports and agreements — NOT a monitoring tool.

### Two Components
| Component | Description |
|-----------|-------------|
| **Artifact Reports** | Download AWS compliance reports: SOC 1/2/3, PCI DSS AOC, ISO 27001/27017/27018, FedRAMP, GDPR DPA |
| **Artifact Agreements** | Review and accept legal agreements: BAA (HIPAA), GDPR DPA, NDA |

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Auditor needs AWS SOC 2 Type II report | Download from AWS Artifact Reports |
| Company needs to sign HIPAA BAA with AWS | AWS Artifact Agreements |
| Where to get AWS ISO 27001 certification | AWS Artifact |

---

## AWS Well-Architected Tool — Security Pillar

**What it is:** Evaluate workloads against AWS best practices across six pillars — Security pillar is most relevant for SCS-C03.

### Security Pillar — Six Design Principles
1. **Implement a strong identity foundation** — least privilege, roles, no long-term keys
2. **Enable traceability** — CloudTrail, CloudWatch, Access Analyzer, alerts on changes
3. **Apply security at all layers** — VPC, subnets, EC2, OS, application, data
4. **Automate security best practices** — IaC for security controls, automated responses
5. **Protect data in transit and at rest** — TLS, KMS, envelope encryption
6. **Keep people away from data** — minimize direct access; use tools and automation
7. **Prepare for security events** — incident response plans, simulations (FIS), runbooks

### Well-Architected Review Process
1. Define workload in WAT.
2. Answer questions in each pillar.
3. Receive **High Risk (HRI)** and **Medium Risk (MRI)** findings.
4. Generate improvement plan.
5. Milestone snapshots track progress over time.

### Exam Key Points
| Scenario | Answer |
|----------|--------|
| Assess a workload against AWS security best practices | AWS Well-Architected Tool Security Pillar review |
| Track security posture improvement over time | Well-Architected Tool milestones |

---

# ⚡ SCS-C03 EDGE CASES — REAL EXAM EXPERIENCE (Community-Reported)

> Sources: r/AWSCertifications exam reports, A Cloud Guru forums, TechStudySlack, and personal exam debriefs shared publicly. These are the **questions people got wrong** — not the ones they expected.

---

## KMS Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| AWS managed key cross-account | "I'll just share the key" | ❌ AWS managed keys (`aws/s3`, `aws/ebs`) **cannot** be used cross-account. Must use a customer-managed CMK |
| Key rotation changes key ID | Panic about re-encryption | ❌ Automatic rotation retains the same key ID and ARN. New backing material is added silently. Existing ciphertext remains decryptable |
| Both key policy AND IAM policy required | IAM policy alone should work | ❌ Both must allow the action. IAM policy alone = denied unless key policy delegates control to root account |
| `GenerateDataKey` vs `GenerateDataKeyWithoutPlaintext` | Same thing? | `GenerateDataKey` returns both encrypted + plaintext key (encrypt now). `GenerateDataKeyWithoutPlaintext` returns only encrypted key (decrypt later). Exam uses "pre-generate keys for offline encryption" → `WithoutPlaintext` |
| S3 Bucket Key changes encryption context | Didn't know it changes anything | When S3 Bucket Key is enabled, the encryption context changes from object-level to bucket-level. Applications validating encryption context must account for this |
| Imported key material rotation | Same as CMK rotation? | ❌ Keys with **imported key material cannot use automatic rotation**. Must manually rotate by creating a new CMK |
| Key deletion waiting period | 7 days minimum | Minimum is 7 days, default is 30 days. Cannot be set to 0. Cannot be cancelled after the period expires |
| Multi-region keys vs CRR | "It replicates the encrypted data" | Multi-region keys replicate **key material only** — not the encrypted data. You still need to copy the data separately |

---

## IAM & SCP Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| Permission Boundary grants permissions | "I added the boundary so now they can do it" | ❌ Permission boundaries **only restrict** — they never grant permissions. Identity policy + boundary must both allow |
| SCPs apply to management account | "SCPs govern everything in the org" | ❌ SCPs attached to the root OU or management account **do NOT apply to the management account**. Only member accounts |
| SCPs apply to service-linked roles | "SCPs block everything" | ❌ SCPs **do not restrict service-linked roles (SLRs)**. GuardDuty, Config, etc. use SLRs that bypass SCPs |
| `iam:PassRole` is often the missing permission | "The user has all the right permissions" | When a user creates a Lambda, EC2, or ECS task, they need `iam:PassRole` to assign a role to that resource — this is the most commonly missed permission |
| `aws:PrincipalOrgID` in resource policies | "Need cross-account roles for org-wide access" | Resource policies (S3, KMS, SQS) with `aws:PrincipalOrgID` condition allow any principal in the entire org without explicit cross-account role setup |
| `ExternalId` condition on AssumeRole | "That's for MFA" | `ExternalId` is the **confused deputy protection** for third-party access. If a vendor assumes a role in your account, `ExternalId` prevents other vendors from doing the same |
| `aws:MultiFactorAuthAge` vs `aws:MultiFactorAuthPresent` | Used interchangeably | `Present` = boolean (was MFA used?). `Age` = seconds since MFA authentication. Use `Age` to enforce re-authentication after N seconds |
| Deny in SCP overrides Allow in identity policy | "The identity policy allows it" | Evaluation order: Explicit Deny anywhere = final Deny. SCP Deny cannot be overridden by IAM Allow |

---

## GuardDuty Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| GuardDuty reads CloudTrail independently | "Need to configure CloudTrail for GuardDuty" | GuardDuty pulls CloudTrail management events directly — you do **not** need CloudTrail enabled separately for GuardDuty to work |
| S3 / EKS / Lambda protection off by default | "GuardDuty covers everything once enabled" | GuardDuty core covers EC2/IAM threats. **S3 Protection, EKS Audit Log Monitoring, EKS Runtime Monitoring, Lambda Protection, RDS Protection are separate features that must be explicitly enabled** |
| Suppression rules vs Trusted IP lists | Same thing? | Trusted IP list = never generate findings for this IP. Suppression rule = generate the finding but auto-archive it. Exam uses "stop alerting on known scanner" → Trusted IP list |
| GuardDuty in single region | "Multi-region is needed" | GuardDuty is **regional**. Each region must be enabled independently. Organizations can enable it via delegated admin for all regions at once |
| GuardDuty finding retention | "Findings persist indefinitely" | GuardDuty retains findings for **90 days**. Archive to S3 via EventBridge → Firehose for long-term storage |

---

## CloudTrail Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| Data events are off by default | "CloudTrail records everything" | ❌ Management events are on by default. **S3 object-level (GetObject, PutObject) and Lambda invoke events are data events — disabled by default. Must explicitly enable** |
| CloudTrail does NOT record all events | "It captures everything in AWS" | CloudTrail records **API calls only**. Does not capture: OS-level events, in-instance activity, stdout/stderr, application logs |
| CloudTrail log file integrity | "Logs are automatically tamper-proof" | Must explicitly enable **log file validation** to get digest files. Without it, you cannot detect log tampering |
| CloudTrail Lake vs S3 + Athena | Both seem equivalent | CloudTrail Lake = lowest operational overhead for SQL querying (no S3 bucket, no Glue catalog needed). S3 + Athena = more flexible, requires more setup |
| CloudTrail across regions | "One trail covers all regions" | A trail can be configured as multi-region via `--is-multi-region-trail`. Without this flag, it only covers the region it was created in |
| CloudTrail for S3 server access logging | Same as S3 data events? | CloudTrail S3 data events = per-object API call logging (who called GetObject). S3 server access logs = HTTP-level request logging. Different granularity, different use cases |

---

## S3 Security Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| S3 Block Public Access overrides bucket policy | "The bucket policy allows public, so it works" | S3 Block Public Access (at account or bucket level) **overrides** all bucket policies and ACLs. It is the highest-priority control |
| Pre-signed URL expiry with STS (role) credentials | "URL is valid until its stated expiry time" | ❌ For role-based pre-signed URLs, the URL expires when the **STS session expires** — even if the URL's stated expiry is later. Permissions are evaluated at **access time** (revoking the session or the access key invalidates the URL). To force-revoke before expiry, attach an explicit Deny with `aws:TokenIssueTime` condition on the role |
| Object Lock Compliance vs Governance mode | Same thing? | **Governance:** users with `s3:BypassGovernanceRetention` can delete. **Compliance:** **nobody** — including root — can delete or shorten the retention period. More restrictive |
| CRR does not replicate delete markers by default | "Replication copies everything including deletes" | CRR replicates new objects. Delete markers are **not replicated by default** — must configure `DeleteMarkerReplication`. Deletions on source do not auto-delete replicas |
| SSE-C key not stored by AWS | "AWS manages the key" | With SSE-C, the customer provides the encryption key with every request. **AWS does not store the key** — if you lose it, the data is unrecoverable |
| S3 Access Points restrict access per VPC | "Bucket policy does the same thing" | S3 Access Points allow per-application access policies and can be restricted to a specific VPC endpoint only — easier to manage than one complex bucket policy |

---

## VPC & Network Security Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| VPC Flow Logs miss some traffic | "Flow logs = full packet capture" | VPC Flow Logs **do NOT capture**: traffic to/from 169.254.169.254 (metadata service), DHCP traffic, DNS queries to the VPC resolver, Windows license activation traffic |
| PrivateLink is one-directional | "Like VPC peering — both sides can talk" | PrivateLink (Interface endpoint) is **consumer → provider only**. The provider cannot initiate connections back. No route table update needed |
| Interface endpoints vs Gateway endpoints | Same thing, just different syntax | **Gateway endpoints** (S3, DynamoDB): free, route table entry. **Interface endpoints** (most other services): cost $0.01/hr/AZ, use DNS |
| Network Firewall vs WAF vs Security Groups vs NACLs | All do "firewall stuff" | SG = per-instance stateful. NACL = per-subnet stateless. WAF = Layer 7 HTTP filtering. Network Firewall = deep packet inspection, IDS/IPS, FQDN filtering at VPC perimeter |
| Shield Advanced does NOT auto-block attacks | "Shield protects everything automatically" | Shield Standard auto-blocks common DDoS. **Shield Advanced provides detection, cost protection, 24/7 DRT access, and enhanced reports — but does not automatically block without WAF or Network Firewall** |

---

## AWS Config Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| Config records state, does NOT prevent changes | "Config will block non-compliant resources" | ❌ Config is **detective**, not preventive. It records configuration history and evaluates rules but **cannot block API calls**. Use SCPs or IAM to prevent |
| Config rules evaluate periodically OR on change | "Config is real-time" | Change-triggered rules run when a resource changes. Periodic rules run on a schedule (1h, 3h, 6h, 12h, 24h). Not all rules are truly real-time |
| Config remediation requires SSM Automation | "Config remediates by itself" | Automated remediation in Config uses **SSM Automation documents**. Config triggers the document — the actual fix is done by SSM |
| Config Aggregator shows findings but can't remediate | "Aggregator = control plane for all accounts" | Config Aggregator is **read-only** — it aggregates data from source accounts for visibility only. Remediation must be done in each source account |

---

## Inspector v2 Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| Inspector v2 is completely different from v1 | "Same service, newer version" | Inspector v2 is a **rewrite**: always-on, no assessment targets/templates, automated, integrates with Security Hub. v1 is legacy. Exam tests v2 |
| Inspector needs SSM Agent for EC2 | "Inspector scans EC2 directly" | EC2 vulnerability scanning in Inspector v2 requires the **SSM Agent** to be installed and the instance to be managed by Systems Manager |
| Inspector does NOT scan S3 | "Inspector scans all storage for issues" | ❌ Inspector scans EC2, ECR container images, and Lambda functions. **S3 sensitive data scanning = Macie** |
| Inspector scans Lambda code | Didn't know | Inspector v2 scans **Lambda function code** for software vulnerabilities (not just packages) — this is a commonly tested newer feature |

---

## Security Hub Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| Security Hub does NOT detect threats | "Security Hub = GuardDuty alternative" | Security Hub **aggregates and scores findings from other services** (GuardDuty, Inspector, Macie, Config, Firewall Manager). It does not independently detect threats |
| Findings use ASFF format | "Just JSON" | All findings in Security Hub use **Amazon Security Finding Format (ASFF)**. Cross-service integrations depend on this — exam may ask about format compatibility |
| Security Hub requires Config | "Security Hub works standalone" | Security Hub's **compliance standards** (CIS, PCI, FSBP) require AWS Config to be enabled. Without Config, the compliance checks cannot run |
| Delegated administrator receives all findings | "Each account manages its own" | With a delegated admin, **all member account findings are visible in the admin account**. Individual accounts can still see their own findings but not others |

---

## Secrets Manager & Parameter Store Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| First rotation runs immediately | "Rotation will start on the schedule" | When you enable rotation for the first time, **rotation runs immediately** — if your app is mid-connection, it may break. Use the rotation window to mitigate |
| Rotation Lambda needs network access | "Lambda auto-connects to the database" | The rotation Lambda must have **network access to the secret's target** (e.g., the RDS instance). If the DB is in a private subnet, Lambda must be in the same VPC |
| Parameter Store cross-account | "Same as Secrets Manager" | ❌ Parameter Store does **not support cross-account access**. Secrets Manager does via resource-based policy |
| Secrets Manager resource policy for cross-account | "Use AssumeRole for cross-account secrets" | Secrets Manager supports a **resource policy** that allows specific external accounts to call `GetSecretValue` directly — no role assumption needed |

---

## Cognito Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| User Pool vs Identity Pool confusion | Used interchangeably | **User Pool = authentication** (creates user identities, issues JWT tokens). **Identity Pool = authorization** (exchanges tokens for temporary AWS credentials via STS) |
| Identity Pool unauthenticated role | "Unauthenticated = no access" | Identity Pools can assign an **unauthenticated IAM role** — giving limited AWS access to guest users without sign-in. Exam tests whether you know this is configurable |
| Cognito hosted UI certificate must be in us-east-1 | Didn't know | Cognito custom domain (hosted UI) requires the ACM certificate to be in **us-east-1** regardless of the Cognito pool region |
| Cognito User Pool as IdP to User Pool | "Can't federate to another User Pool" | A Cognito User Pool can federate with external SAML/OIDC IdPs — and a second Cognito User Pool can act as an OIDC IdP for federation |

---

## Firewall Manager Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| Firewall Manager is the "future accounts" answer | "Use SCP to auto-apply WAF" | When the question asks "automatically apply WAF rules to all current and **future** accounts in the org" → **Firewall Manager**, not SCP. Firewall Manager auto-remediates new accounts |
| Requires Organizations with all features | "Works with consolidated billing only" | Firewall Manager requires **AWS Organizations with all features enabled** (not just consolidated billing) and a designated Firewall Manager admin account |
| Manages multiple security services | "Just for WAF" | Firewall Manager manages: WAF, Shield Advanced, Security Groups, Network Firewall, Route 53 Resolver DNS Firewall, and VPC endpoint policies |

---

## ACM Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| ACM public certificates cannot be exported | "I'll export the cert and install it on-prem" | ❌ ACM public certificates are **not exportable** — the private key never leaves ACM. For on-prem/non-AWS use, use ACM Private CA (certificates from it can be exported) |
| CloudFront requires ACM cert in us-east-1 | "Cert can be in any region" | ACM certificates for **CloudFront must be in us-east-1** regardless of distribution origin region. ALB certificates must be in the same region as the ALB |
| ACM auto-renews 60 days before expiry | "Manual renewal needed" | ACM auto-renews public certificates. But if **DNS validation** is used, the CNAME record must still exist. If it was deleted, renewal fails silently |

---

## Amazon Detective Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| Detective ≠ GuardDuty | "Detective detects threats" | Detective is for **investigation and root cause analysis** — not detection. GuardDuty detects → Security Hub aggregates → **Detective investigates** |
| Detective requires GuardDuty | "Standalone service" | Detective **requires GuardDuty to be enabled** in the account/region as its data source |
| Detective uses graph model | "It's just dashboards" | Detective builds a **behavior graph** from CloudTrail, VPC Flow Logs, and GuardDuty findings — used to visualize lateral movement, affected resources, and timelines |

---

## Commonly Misidentified Service Pairs (Real Exam Traps)

| Scenario | Wrong answer (common) | Correct answer |
|----------|----------------------|----------------|
| Detect sensitive data in S3 | Inspector | **Macie** |
| Scan EC2 for OS vulnerabilities | GuardDuty | **Inspector v2** |
| Investigate root cause of a GuardDuty finding | Security Hub | **Detective** |
| Aggregate compliance findings across 100 accounts | Config per account | **Security Hub** (delegated admin) |
| "Comprehensive security score" across standards | GuardDuty | **Security Hub** (security score) |
| Auto-apply WAF to future accounts in org | SCP | **Firewall Manager** |
| Detect unusual API call volume | CloudWatch alarm | **CloudTrail Insights** |
| SQL query against CloudTrail logs (lowest overhead) | Athena + S3 | **CloudTrail Lake** |
| Org-wide S3 public access prevention without SCP | IAM policy | **S3 Block Public Access at account level** (or enforced via Config rule + SCP) |
| Quarantine compromised EC2 without deleting it | Terminate + redeploy | **Isolate: remove SG, revoke IAM role, snapshot for forensics** |
| Restrict what external principals can do to org resources | SCP | **RCP (Resource Control Policy)** |
| Enforce IMDSv2 on all EC2 regardless of launch parameters | SCP deny IMDSv1 | **Declarative Policy** (enforces configuration state, not just API) |
| Fine-grained authorization decisions externalized from app code | IAM | **Amazon Verified Permissions (Cedar)** |
| On-premises servers accessing AWS without long-term keys | IAM access keys | **IAM Roles Anywhere** (X.509 certs + temporary credentials) |
| Validate CloudFormation templates against security rules in CI/CD | Manual review | **cfn-guard** (policy-as-code) |
| Check CloudFormation template for syntax correct before deploy | cfn-guard | **cfn-lint** (syntax + correctness) |
| Encrypt Direct Connect link at Layer 2 | Site-to-Site VPN | **MACsec on dedicated Direct Connect** |
| Cross-account resource sharing without copying (Transit Gateway) | VPC peering | **AWS RAM** |
| Collect audit evidence continuously for compliance frameworks | Manual evidence gathering | **AWS Audit Manager** |
| Download AWS SOC 2 / PCI DSS reports for auditors | AWS Security Hub | **AWS Artifact** |

---

## Governance Edge Cases

| Edge Case | What Trips People | Correct Understanding |
|-----------|------------------|-----------------------|
| RCP vs SCP confusion | "SCP prevents external access to resources" | ❌ SCP restricts **principals**. RCP restricts **resources** — only RCPs can block external principals from accessing org resources |
| RCP applies to management account | "New policy, probably same as SCP" | ❌ Like SCPs, RCPs do **not apply to the management account** |
| Declarative Policy vs SCP for IMDSv2 | Same effect? | Different mechanism: SCP Deny blocks the API call if wrong value; Declarative Policy **enforces** the attribute state at provisioning time — stronger guarantee |
| Control Tower proactive controls block what | "Same as detective controls but proactive" | Proactive controls (CloudFormation Hooks) reject CloudFormation resource creation BEFORE it happens — not just detect after |
| Service Catalog launch constraint | "Need to grant CloudFormation permissions to devs" | ❌ Launch constraint IAM role handles deployment — devs only need Service Catalog permissions, not CloudFormation or IAM permissions |
| RAM shared subnet ownership | "My account owns resources in the shared subnet" | You launch instances in the shared subnet — but the **subnet itself belongs to the owner account**. You pay for your instances, owner pays for the subnet |

---

## Numbers & Limits That Appear in Exam Scenarios

| Item | Value | Why It Matters |
|------|-------|---------------|
| KMS direct encrypt max | 4 KB | Reason to use `GenerateDataKey` (envelope encryption) |
| KMS key deletion minimum waiting period | 7 days | Cannot skip or reduce below 7 days; imported keys can delete immediately |
| GuardDuty finding retention | 90 days | Archive to S3 for longer retention |
| Parameter Store Standard max size | 4 KB | Larger values need Advanced tier (8 KB) |
| Secrets Manager max secret size | 64 KB | Rarely a constraint but tested |
| CloudTrail log delivery delay | ~15 minutes (typical) | Not real-time — use EventBridge for near-real-time |
| S3 Object Lock Compliance mode | No one can delete | Even root cannot override Compliance mode retention |
| ACM auto-renewal lead time | 60 days before expiry | Fails silently if DNS validation CNAME was removed |
| SCP management account exemption | Always | Management account root is never restricted by SCPs |
| Session Manager session logging | Full keystroke log | Captured to CloudTrail + optional S3/CloudWatch |
| IAM Roles Anywhere max session duration | 12 hours | Same as standard role assumption max |
| CloudFront ACM cert region | us-east-1 only | Must be in us-east-1 regardless of origin region |

---

# ✅ FINAL EXAM CHECK

Run this checklist before submitting every answer:

| Question | Check |
|----------|-------|
| Is this **visibility** or **processing**? | Visibility → CloudWatch/CloudTrail. Processing → Kinesis. |
| Did I follow keyword mapping? | Match the verb in the question to the service table above. |
| Am I choosing the most AWS-native managed solution? | Prefer built-in rotation, delegated admin, managed rules over custom Lambda. |
| Am I overengineering? | If one service solves it natively, don't add a second. |
| Does the SCP have an escape hatch? | Blanket denies with no exception = likely wrong answer. |
| Is KMS cross-account using a CMK (not AWS managed key)? | AWS managed keys cannot be shared cross-account. |
| Is it detection or prevention being asked for? | GuardDuty/Inspector = detect. SCP/Config rules = enforce. |
| Is it restricting principals or resources? | Principals → SCP. Resources org-wide → RCP. |
| Does the question involve on-prem workloads needing AWS access? | Consider IAM Roles Anywhere (X.509) not access keys. |
| Is there a "policy-as-code" in CI/CD requirement? | cfn-guard (security rules) not cfn-lint (syntax check). |

> **Golden rule:** If two answers look correct, pick the one whose service name matches the keyword used in the question stem — not your preferred architecture.

---