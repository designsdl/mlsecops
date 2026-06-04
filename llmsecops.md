# LLMSecOps Lifecycle

A security operations model for Large Language Model (LLM) applications, mirroring the
[MLSecOps Lifecycle](mlsecops.md) but tailored to the realities of foundation models,
prompt/context engineering, Retrieval-Augmented Generation (RAG), agentic systems, and
the Model Context Protocol (MCP).

Each phase lists **Key Activities**, **Risks**, and **Risk Mitigation**. Where applicable,
risks and mitigations are mapped to MITRE ATLAS

---

## 1. Use Case Definition & Data Sourcing
### Key Activities
- **Use Case Scoping**: Define the task, sensitivity tier, and acceptable-use boundaries before any build.
- **Foundation Model Sourcing**: Select base/foundation models (hosted API or open-weights).
- **Corpus Aggregation**: Collect fine-tuning data, RAG knowledge bases, and evaluation sets.
- **Data Wrangling & Labeling**: Clean, deduplicate, and annotate instruction/preference data securely.
- **Provenance Capture**: Record source, license, and modification history for every dataset and model.
- **Feature/Embedding Stores**: Maintain consistent embeddings between indexing and inference time.

### Risks
1. **Poisoned Foundation Models**: Open-weights model carries a backdoor or trojan (`AML.T0058 Publish Poisoned Models`, `AML.CS0019 PoisonGPT`).
2. **Supply Chain Compromise**: Malicious model/dataset/dependency pulled from a registry (`AML.T0010 ML Supply Chain Compromise`, `AML.CS0015 Compromised PyTorch Dependency Chain`).
3. **Web-Scale Data Poisoning**: Adversary controls expired domains or split-view content in scraped corpora (`AML.T0008.002 Domains`, `AML.CS0025 Split-View Poisoning`).
4. **Data Confidentiality & Compliance**: Sensitive/PII/regulated data ingested into training or RAG corpora (GDPR/HIPAA).
5. **License & Provenance Gaps**: Unknown lineage prevents rapid response to disclosed model/data vulnerabilities.

### Risk Mitigation
- **Verify ML Artifacts** (`AML.M0014`): Cryptographically verify model/dataset hashes and signatures before use.
- **AI Bill of Materials** (`AML.M0023`) & **Maintain AI Dataset Provenance** (`AML.M0025`): Track every artifact and its full modification history.
- **Sanitize Training Data** (`AML.M0007`): Detect and remove poisoned, duplicated, or out-of-policy records.
- **Encrypt Sensitive Information** (`AML.M0012`) and anonymize/tokenize PII during aggregation and storage.
- **Vulnerability Scanning** (`AML.M0016`): Scan model files (enforce safe formats such as `safetensors` over unsafe pickle) and dependencies.
- **Compliance Tooling**: Classify and gate regulated data (GDPR/HIPAA) before it enters any corpus.
---

## 2. Model Selection, Fine-Tuning & Alignment
### Key Activities
- **Base Model Evaluation**: Benchmark candidate models for capability, safety, and cost.
- **Fine-Tuning / Adapters**: SFT, LoRA/PEFT, or full fine-tuning on curated data.
- **Alignment**: RLHF / RLAIF / Constitutional methods and safety context distillation.
- **Safety & Red-Team Evaluation**: Jailbreak, prompt-injection, and harmful-content testing.
- **Artifact Lineage**: Version models, adapters, datasets, and eval results.

### Risks
1. **Alignment Erosion**: Fine-tuning strips built-in safety behaviors, re-enabling jailbreaks (`AML.T0054 LLM Jailbreak`).
2. **Backdoored / Trojan Models**: Trigger-conditioned malicious behavior inserted during training (`AML.T0018 Backdoor ML Model`, `AML.T0043.004 Insert Backdoor Trigger`).
3. **Training Data Memorization**: Model regurgitates secrets/PII at inference (`AML.T0057 LLM Data Leakage`, `AML.T0024.000 Infer Training Data Membership`).
4. **Reproducibility Gaps**: Inability to reconstruct a model build for audit or rollback.
5. **Weak Safety Evaluation**: Unmeasured susceptibility to adversarial prompts before release.

### Risk Mitigation
- **Generative AI Model Alignment** (`AML.M0022`): Use SFT, RLHF/RLAIF, and targeted safety context distillation; re-validate safety **after** every fine-tune.
- **Model Hardening** (`AML.M0003`) and **Validate ML Model** (`AML.M0008`): Adversarial training and pre-release validation on held-out adversarial sets.
- **Verify ML Artifacts** (`AML.M0014`) + **Code Signing** (`AML.M0013`): Sign models and adapters; verify provenance to detect trojans.
- **Experiment Tracking**: Log builds (e.g., MLflow) for reproducibility and rollback.
- **Privacy-Preserving Training**: Deduplicate/scrub secrets; apply differential privacy where memorization is a concern.
---

## 3. Prompt & Context Engineering
### Key Activities
- **System Prompt Design**: Define role, goals, voice, and safety/security parameters.
- **Guideline Injection**: Append safety instructions to user prompts or system context.
- **Context Assembly**: Compose system + retrieved + user content with clear trust boundaries and delimiters.
- **Secret Management**: Keep credentials and keys out of prompts and configuration files.

### Risks
1. **Direct Prompt Injection** (`AML.T0051.000`): User overrides instructions to misuse the model.
2. **System Prompt Extraction** (`AML.T0056 Extract LLM System Prompt`): IP/guardrail disclosure via injection or config exposure.
3. **System Information Discovery** (`AML.T0069`): Leaking delimiters (`AML.T0069.000`), function keywords (`AML.T0069.001`), or the full system prompt (`AML.T0069.002`) used to craft attacks.
4. **Unsecured Credentials in Prompts/Config** (`AML.T0055`): Secrets embedded in prompt templates or env files.
5. **Trust-Boundary Confusion**: Retrieved or user content treated as privileged instructions.

### Risk Mitigation
- **Generative AI Guidelines** (`AML.M0021`): Encode goal, role, voice, and safety bounds in the system prompt; instruct refusal of unsafe inputs.
- **Generative AI Guardrails** (`AML.M0020`): Filter/validate inputs and outputs to block injection and meta-prompt extraction.
- **Strong Delimiting & Trust Tiers**: Structurally separate system, retrieved, and user content; never grant retrieved/user text instruction-level trust.
- **Encrypt Sensitive Information** (`AML.M0012`): Externalize secrets to a vault; keep them out of prompts/config.
- **Least-Privilege System Prompts**: Avoid embedding exploitable capability hints; minimize disclosed keywords.
---

## 4. RAG Security (Retrieval-Augmented Generation)
### Key Activities
- **Ingestion Pipeline**: Crawl/ingest documents, chunk, embed, and index into a vector store.
- **Data Lineage & Tracking**: Record source, owner, ingestion time, and checksum for every chunk.
- **Retrieval & Re-ranking**: Fetch top-k context for a query and assemble it into the prompt.
- **Poisoning Detection**: Classify and screen ingested content for malicious or anomalous entries.
- **Access-Scoped Retrieval**: Filter the index by the requesting user's authorization.

### Risks
1. **RAG Poisoning** (`AML.T0070`): Malicious documents indexed to surface for targeted queries.
2. **False RAG Entry Injection** (`AML.T0071`): Content disguised as a trusted RAG result, bypassing monitoring and resisting deletion.
3. **Indirect Prompt Injection** (`AML.T0051.001`): Hidden instructions in retrieved content (`AML.CS0020 Bing Chat Data Pirate`).
4. **RAG-Indexed Target Reconnaissance** (`AML.T0064`) & **Retrieval Content Crafting** (`AML.T0066`): Attacker maps and tailors content to the index.
5. **RAG-Based Worms**: Self-replicating injected prompts that propagate through shared knowledge bases (`AML.T0061 Prompt Self-Replication`, `AML.CS0024 Morris II Worm`).
6. **Cross-Tenant Leakage**: Retrieval ignores per-user authorization and surfaces another tenant's data.

### Risk Mitigation
- **RAG Data Lineage & Tracking**: Maintain provenance per chunk (source, author, timestamp, checksum); make every retrieved span auditable back to a trusted origin — extends **Maintain AI Dataset Provenance** (`AML.M0025`) and **AI BOM** (`AML.M0023`).
- **Data-Poisoning Detection Classifiers**: Screen ingested documents with classifiers/anomaly detection for injection patterns, obfuscation, and policy violations before indexing; re-scan on update.
- **Generative AI Guardrails** (`AML.M0020`): Apply input/output guardrails to retrieved context, not just the user turn, to catch indirect injection.
- **Ingestion Authorization & Trusted Sources**: Restrict who/what can write to the index; allowlist ingestion sources to mitigate `AML.T0066`/`AML.T0070`.
- **Access-Scoped Retrieval**: Enforce document-level ACLs at query time so retrieval honors the user's permissions (prevents cross-tenant leakage).
- **Sanitize & Normalize Content**: Strip hidden HTML, zero-width/invisible characters, and styling abused for **LLM Prompt Obfuscation** (`AML.T0068`).
- **Provenance-Aware Prompting**: Tag retrieved content as untrusted data (never instructions) and surface source attribution to the user.
---

## 5. Agentic Security (Tool-Using & Autonomous Agents)
### Key Activities
- **Agent Identity**: Issue each agent a distinct, attestable workload identity (not a shared human credential).
- **Authentication & Authorization (AuthN/AuthZ)**: Authenticate the agent and authorize each action against a policy.
- **On-Behalf-Of (OBO) Delegation**: Scope agent permissions to the **invoking user's role** so the agent can never exceed the user's own rights.
- **Tool / Action Governance**: Gate tool calls, code execution, and external API access behind allowlists and approvals.
- **Observability, Logging & Tracing**: Record every prompt, tool call, parameter, and result across multi-step plans.
- **Human-in-the-Loop**: Require confirmation for high-impact or irreversible actions.

### Risks
1. **Plugin / Tool Compromise** (`AML.T0053 LLM Plugin Compromise`): Injected instructions drive privileged API/code execution (`AML.CS0016 MathGPT RCE`, `AML.CS0018 Code Execution via Colab`).
2. **Privilege Escalation via the Agent**: Agent acts beyond the user's authorization or with broad service credentials (`AML.TA0012 Privilege Escalation`).
3. **Confused-Deputy / Insider Abuse**: Trusted agent is steered to act against the user (`AML.CS0026 Financial Transaction Hijacking with M365 Copilot as an Insider`).
4. **Injection-Driven Exfiltration**: Agent leaks data through tools/links (`AML.T0057 LLM Data Leakage`, `AML.CS0021 ChatGPT Plugin Privacy Leak`).
5. **Trusted Output Manipulation** (`AML.T0067`, incl. `AML.T0067.000 Citations`): Agent makes malicious actions/links appear trustworthy.
6. **Self-Propagating Agentic Worms** (`AML.T0061`, `AML.CS0024 Morris II`): Injected prompts spread agent-to-agent.

### Risk Mitigation
- **Agent AuthN/AuthZ**: Give each agent its own identity; authenticate and authorize **every** action, not just session start.
- **On-Behalf-Of, Least Privilege**: Bind agent permissions to the user's role via OBO/token exchange and short-lived scoped credentials; the agent's effective rights = (agent policy ∩ user role). Never run agents with standing admin tokens.
- **Tool Allowlisting & Sandboxing**: Restrict callable tools; run code execution in isolated, egress-controlled sandboxes (**Restrict Library Loading** `AML.M0011`, **Control Access in Production** `AML.M0019`).
- **Human Approval Gates**: Require confirmation for irreversible/high-value actions (payments, deletes, external sends).
- **Generative AI Guardrails** (`AML.M0020`): Validate tool-call arguments and agent outputs; detect PII/secret egress.
- **AI Telemetry Logging** (`AML.M0024`): Full action-level audit trail (prompts, tool calls, args, results) for detection and forensics — see [Observability](#8-monitoring-observability--logging).
- **Output Trust Verification**: Validate and attribute links/citations/recommended actions to counter `AML.T0067`.
---

## 6. MCP Security (Model Context Protocol)
### Key Activities
- **Server Registration & Discovery**: Maintain a vetted registry of approved MCP servers and their tools/resources.
- **AuthN/AuthZ at the MCP Boundary**: Authenticate client↔server connections and authorize each tool/resource invocation.
- **Capability Scoping**: Declare and constrain the tools, resources, and prompts each server exposes.
- **Observability, Logging & Monitoring**: Trace every MCP request/response and tool invocation end-to-end.
- **Supply-Chain Vetting**: Review third-party MCP server code and updates before connecting.

### Risks
1. **Malicious / Compromised MCP Server**: Untrusted server exposes harmful tools or exfiltrates context (`AML.T0010 Supply Chain`, `AML.T0053 Plugin Compromise`).
2. **Tool Poisoning / Rug-Pull**: Tool descriptions or behavior change post-approval to inject instructions or alter actions (`AML.T0051.001 Indirect Injection`, `AML.T0067 Trusted Output Manipulation`).
3. **Over-Broad Capabilities**: Server granted more access than the task requires (`AML.TA0012 Privilege Escalation`).
4. **Token / Credential Theft**: OAuth tokens or secrets passed to MCP servers harvested (`AML.T0055 Unsecured Credentials`).
5. **Cross-Server Confused Deputy**: One server's tool result manipulates the agent into misusing another server.
6. **Unmonitored Channels**: Tool invocations occur with no audit trail.

### Risk Mitigation
- **MCP AuthN/AuthZ**: Mutually authenticate connections; authorize each tool/resource call against policy; prefer scoped, short-lived tokens via proper OAuth flows.
- **Server Allowlisting & Pinning**: Connect only to registered servers; pin versions/hashes and re-vet on update to counter rug-pulls (**Verify ML Artifacts** `AML.M0014`, **AI BOM** `AML.M0023`).
- **Least-Privilege Capability Scoping**: Grant each server only the tools/resources its task needs; isolate per-tenant.
- **Tool-Description Integrity**: Treat tool/prompt descriptions as untrusted data; validate and diff them; apply guardrails (`AML.M0020`) to tool inputs/outputs.
- **Credential Hygiene**: Vault secrets, never embed in server config; rotate tokens; scope per server (**Encrypt Sensitive Information** `AML.M0012`).
- **MCP Observability & Monitoring**: Log and trace every request/response and invocation (**AI Telemetry Logging** `AML.M0024`); alert on anomalous tool usage.
- **Egress Controls**: Sandbox MCP servers and restrict outbound network access to limit exfiltration.
---

## 7. Deployment & Serving
### Key Activities
- **Endpoint Deployment**: Serve the model/app behind an API gateway.
- **Inference Stack Hardening**: Secure the serving runtime, GPUs, orchestration, and dependencies.
- **Rate & Quota Controls**: Throttle per-user/per-key request volume.
- **Model Registry**: Store and version deployed models and configs with signatures.

### Risks
1. **Host & Infrastructure Compromise**: Exposed inference/orchestration infra (`AML.T0049 Exploit Public-Facing Application`, `AML.CS0023 ShadowRay`, `AML.CS0010 Azure Service Disruption`).
2. **Model Theft / Replication**: Weights or behavior extracted (`AML.T0048 ML IP Theft`, `AML.T0024.002 Extract ML Model`, `AML.CS0007 GPT-2 Replication`).
3. **Insecure Config Exposure**: System prompts/keys in deployable configs (`AML.T0056`, `AML.CS0006 ClearviewAI Misconfiguration`).
4. **Denial of Wallet / Service**: Cost-driving or resource-exhausting traffic (`AML.T0029 Denial of ML Service`, `AML.T0034 Cost Harvesting`).
5. **Unsafe Serialized Artifacts**: Malicious model files executing on load (`AML.T0011.000 Unsafe ML Artifacts`).

### Risk Mitigation
- **Control Access to Models & Data in Production** (`AML.M0019`) and **at Rest** (`AML.M0005`): IAM, network isolation, and secrets management around the serving stack.
- **API Gateway Hardening**: AuthN/AuthZ, **Restrict Number of ML Model Queries** (`AML.M0004`), rate limits, and quotas to blunt extraction and denial-of-wallet.
- **Code Signing** (`AML.M0013`) & **Verify ML Artifacts** (`AML.M0014`): Verify signed models before load; enforce safe serialization formats.
- **Passive Output Obfuscation** (`AML.M0002`): Withhold raw logits/confidence not needed by users (`AML.T0063 Discover AI Model Outputs`).
- **Vulnerability Scanning** (`AML.M0016`): Patch serving frameworks, orchestration (e.g., Ray), and dependencies.
---

## 8. Monitoring, Observability & Logging
### Key Activities
- **Prompt/Response Telemetry**: Log inputs, outputs, tool calls, and retrieved context.
- **Abuse & Threat Detection**: Detect injection, jailbreak, and exfiltration patterns in real time.
- **Drift & Quality Monitoring**: Track output quality, refusal rates, and behavior drift.
- **Cost & Rate Monitoring**: Watch token spend and traffic anomalies.
- **Alerting & Response**: Trigger alerts and automated mitigations on anomalies.

### Risks
1. **Undetected Injection/Jailbreak**: Attacks proceed without telemetry (`AML.T0051`, `AML.T0054`).
2. **Silent Data Exfiltration**: Leakage via API/tools goes unnoticed (`AML.T0024 Exfiltration via Inference API`, `AML.T0057`).
3. **Obfuscated Attacks** (`AML.T0068 LLM Prompt Obfuscation`): Hidden/invisible payloads evade naive logging.
4. **Evaluation/Telemetry Poisoning**: Attacker pollutes feedback loops used for retraining (`AML.T0002 Poison Training Data`, `AML.CS0009 Tay Poisoning`).
5. **Privacy in Logs**: Sensitive prompts/PII stored insecurely in telemetry.

### Risk Mitigation
- **AI Telemetry Logging** (`AML.M0024`): Comprehensive, tamper-evident logging of inputs/outputs/tool calls across agents, RAG, and MCP.
- **Adversarial Input Detection** (`AML.M0015`): Real-time classifiers for injection, jailbreak, and anomalous prompts; normalize input to defeat obfuscation.
- **Guardrail Telemetry**: Capture guardrail (`AML.M0020`) decisions (blocks, redactions) as first-class signals.
- **Feedback-Loop Integrity**: Quarantine and review user-supplied feedback before it influences fine-tuning (counters Tay-style poisoning).
- **Log Privacy**: Redact/tokenize PII in telemetry; **Encrypt Sensitive Information** (`AML.M0012`) and access-control logs.
- **Anomaly Alerting**: Baseline token/cost/refusal metrics; alert on deviations indicating extraction or denial-of-wallet.
---

## 9. Output & Downstream Risks
### Key Activities
- **Output Validation**: Verify, filter, and ground responses before downstream use.
- **Citation & Source Attribution**: Attach and verify provenance for claims and links.
- **Safe Rendering**: Sanitize outputs consumed by browsers, shells, or other systems.

### Risks
1. **Hallucination Exploitation**: Fabricated packages/URLs/entities weaponized (`AML.T0062 Discover LLM Hallucinations`, `AML.T0060 Publish Hallucinated Entities`, `AML.CS0022 ChatGPT Package Hallucination`).
2. **Trusted Output / Citation Manipulation** (`AML.T0067`, `AML.T0067.000`): Malicious links/actions framed as trustworthy.
3. **Insecure Output Handling**: Model output flows unsanitized into code/SQL/markup (downstream injection).
4. **Misinformation & Harmful Content**: Unsafe or non-compliant generations reach users.
5. **Output Provenance Gaps**: No traceability for what the system asserted or did.

### Risk Mitigation
- **Generative AI Guardrails** (`AML.M0020`): Output-side filters for harmful content, PII, code exploits, and SQL/command injection.
- **Hallucination Defenses**: Ground answers in retrieved sources; verify package/URL existence before suggesting; flag low-confidence claims (counters `AML.T0060`/`AML.T0062`).
- **Citation Verification**: Validate that cited sources exist and support the claim (counters `AML.T0067.000`).
- **Insecure Output Handling**: Treat model output as untrusted; escape/sanitize before execution or rendering downstream.
- **Output Provenance**: Retain response/action lineage for audit and incident response.
---

## 10. Governance & Secure Pipeline
- **AI Bill of Materials** (`AML.M0023`): Maintain a complete inventory of models, datasets, adapters, prompts, tools, and MCP servers for rapid vulnerability response.
- **Provenance Everywhere** (`AML.M0025`, `AML.M0014`): Sign and verify artifacts; track dataset/RAG/model lineage end-to-end.
- **Least-Privilege Pipelines**: Enforce access controls across data, training, serving, agentic, and MCP components so no stage can take unauthorized actions.
- **User Training** (`AML.M0018`): Educate users/developers on prompt injection, hallucination, and safe agent/tool use.
- **Generative AI Guidelines** (`AML.M0021`): Standardize safety/security system-prompt policy across applications.
- **Continuous Red-Teaming**: Routinely test injection, jailbreak, RAG poisoning, and agentic/MCP abuse against production guardrails.
- **Incident Response & Rollback**: Use signed registries and the AI BOM to revoke/rollback compromised models, adapters, or MCP servers quickly.

---

### Reference: ATLAS LLM Technique Map
| Domain | Key ATLAS Techniques | Case Studies |
|---|---|---|
| Prompt Injection | `AML.T0051` (`.000` Direct, `.001` Indirect), `AML.T0065` Prompt Crafting, `AML.T0068` Obfuscation | CS0016, CS0020 |
| Jailbreak & Discovery | `AML.T0054` Jailbreak, `AML.T0069` System Info (`.000`–`.002`), `AML.T0056` Extract System Prompt | CS0016 |
| RAG | `AML.T0064` Gather RAG Targets, `AML.T0066` Retrieval Content Crafting, `AML.T0070` RAG Poisoning, `AML.T0071` False RAG Entry | CS0024 |
| Agentic / Tools | `AML.T0053` Plugin Compromise, `AML.T0061` Prompt Self-Replication, `AML.T0067` Trusted Output Manipulation | CS0018, CS0021, CS0024, CS0026 |
| Exfiltration & Leakage | `AML.T0057` Data Leakage, `AML.T0024` Exfil via Inference API | CS0021 |
| Supply Chain & Hallucination | `AML.T0010` Supply Chain, `AML.T0058` Poisoned Models, `AML.T0060`/`AML.T0062` Hallucinated Entities | CS0015, CS0019, CS0022, CS0025 |

