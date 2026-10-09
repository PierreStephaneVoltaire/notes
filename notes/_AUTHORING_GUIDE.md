# Authoring guide for subsection notes (read fully before writing)

Curriculum source: `/home/sirsimpalot/Downloads/learn/infra-architecture-curriculum.md`
(read its "Data caveats" and "Overlaps" tables too). Today is 2026-10-08.

## Audience and purpose
System-design interview notes for **senior/staff** DevOps, SRE, AI engineers, network engineers,
DevSecOps and architects. Include facts that could land in a senior/staff interview or an
AWS/Azure professional or specialty certification. No fluff, no lecture narration, no intro filler.

## Research (mandatory)
- Use WebSearch and WebFetch. Prefer official docs: docs.aws.amazon.com, aws.amazon.com/blogs,
  learn.microsoft.com, kubernetes.io, man7.org / kernel.org, postgresql.org, dev.mysql.com,
  docs.databricks.com, kafka.apache.org, spark.apache.org, developers.cloudflare.com,
  docs.claude.com / docs.anthropic.com, ai.google.dev / cloud.google.com, sre.google, RFCs (rfc-editor.org),
  docs.docker.com, developer.hashicorp.com/terraform.
- Verify anything version-, limit-, quota-, pricing-model- or default-sensitive against current docs.
  Mention renamed/retired services (e.g. "Azure AI Studio → Azure AI Foundry", "as of 2026 …").
  If something could not be verified, mark it `(unverified)`.
- Do a reasonable number of lookups (roughly 8–25 depending on scope); do not stall.
- **WebSearch quota is shared and may be exhausted.** Try WebSearch at most once; if it errors or is
  rate-limited, switch to WebFetch on known official URLs (doc landing pages, "what's new"/release-notes
  pages, limits/quotas pages, FAQ pages) and follow links from there.

## File format (exact skeleton)
```
# <SubsectionID> <Title>
> Last verified: 2026-10 · Audience: senior/staff DevOps, SRE, AI eng, NetEng, DevSecOps, architects

## TL;DR
- 5–8 bullets: what an interviewer expects you to say about this whole subsection

## <ID> <Topic title>        <- one `##` heading PER curriculum ID, exactly prefixed with the ID (e.g. "## C1.27 Caching for performance")
- **How it works:** bullets, concrete numbers, defaults, limits
- **Trade-offs / when to use:** bullets
- **Interview angles:** "If asked X → say Y", common follow-ups, pitfalls/anti-patterns
(Use `###` sub-headings for the curriculum's nested sub-bullets where useful. Keep each ID tight;
trivial IDs can be 3–5 bullets. Merge-by-reference rather than repeating: if a topic is fully
covered in another ID listed in the Overlaps table, give 2–3 bullets and cross-link.)

## Diagrams
- At least ONE ```mermaid block per file (flowchart / sequenceDiagram / stateDiagram / etc.).
  More where a flow, handshake or architecture benefits. Keep mermaid syntax valid:
  quote labels containing special chars, e.g. A["Client (browser)"]; no parentheses/colons unquoted.

## Cloud mapping: AWS vs Azure
(ONLY if the subsection involves non-trivial infra. Skip for pure theory like CPU pipelining —
then omit the heading entirely.)
| Capability | AWS | Azure | Role it plays | Key differences | Alternatives |
- Equivalents must be genuinely equivalent (e.g. Direct Connect ↔ ExpressRoute, Transit Gateway ↔
  Virtual WAN hub / hub-spoke VNet, PrivateLink ↔ Private Link/Private Endpoint, ElastiCache ↔
  Azure Cache for Redis / Azure Managed Redis, SQS ↔ Storage Queues/Service Bus, KMS ↔ Key Vault).
- Below the table: bullets explaining each service's role, the important AWS-vs-Azure differences
  (scope: regional vs global, zonal behaviour, SLA, limits, pricing model shape, gotchas), and
  alternatives. Alternatives may reference: Kubernetes, Cloudflare, Kafka/Confluent, Spark,
  Databricks, Claude, Gemini (GCP only when it's the canonical alternative).
- † items in the curriculum: cloud mapping is first-class. Section G is "neutralized": map every
  generic term back to its AWS AND Azure name.

## Hands-on (optional)
- Code is restricted to: **bash** (```bash), **Docker** (```dockerfile / docker compose ```yaml),
  **Terraform** (```hcl). NO Python, Go, JS, SQL-heavy programs, etc. (a one-line SQL statement
  shown inside a bash `psql -c` call is fine). Keep snippets short and correct.
- Tool configuration counts as config, not code, and is allowed: CI pipeline YAML (GitHub Actions /
  GitLab CI / Azure Pipelines), a short declarative Jenkinsfile, Prometheus/Alertmanager/OTel/Kubernetes
  YAML, JSON policies.

## Cross-links
- Relative markdown links to related subsection files, e.g.
  `[C1 Performance](../C-large-scale-architecture/C1-performance.md#c127-caching-for-performance)`.
  File naming convention: `<folder>/<SubsectionID>-<kebab-title>.md`. Folders:
  A-operating-systems, B-database-engineering, C-large-scale-architecture, D-system-design,
  E-ai-system-design, F-network-engineering, G-cloud-network-architecture,
  H-full-stack-troubleshooting, I-dns-tls-acceleration-gaps, J-sre, K-ai-infra-llm,
  L-data-privacy-ai-security, M-data-platforms, N-cicd-platform-engineering, O-observability-tooling,
  P-security-platforms-identity, Q-industry-domains, R-support-communication, ../languages/.
  If unsure of the exact target filename, link to the folder plus ID text, e.g. `../C-large-scale-architecture/ (C1.27)`.

## Sources
- Bullet list of the official URLs you actually consulted (≥3).

## Rules
- Write ONLY your assigned file (Write tool). Do not touch other files. No other subagents.
- Markdown only, bullets over prose. Bold key terms. Use tables for comparisons.
- When finished, reply with ONE short line: file path, number of IDs covered, and anything unverified.

## Exact filenames of all subsection files (use these for cross-links)
A-operating-systems: A1-why-an-os.md, A2-the-anatomy-of-a-process.md, A3-memory-management.md, A4-inside-the-cpu.md, A5-process-management.md, A6-storage-management.md, A7-socket-management.md, A8-more-os-concepts.md, A9-bonus.md
B-database-engineering: B1-acid.md, B2-database-internals.md, B3-database-indexing.md, B4-btree-vs-bplustree.md, B5-database-partitioning.md, B6-database-sharding.md, B7-concurrency-control.md, B8-database-replication.md, B9-database-system-design.md, B10-database-engines.md, B11-database-cursors.md, B12-database-security.md, B13-homomorphic-encryption.md, B14-database-discussions.md
C-large-scale-architecture: C1-performance.md, C2-scalability.md, C3-reliability.md, C4-security.md, C5-deployment.md, C6-technology-stack.md
D-system-design: D1-system-design-basics.md, D2-reusable-parts-of-system-design.md, D3-system-design-of-modern-applications.md
E-ai-system-design: E1-system-design-fundamentals.md, E2-google-ctr-prediction-case-study.md, E3-hubspot-user-clustering-case-study.md, E4-facebook-content-moderation-case-study.md, E5-smart-car-parking-case-study.md, E6-ai-grammar-checker-case-study.md, E7-ai-interview-chatbot-case-study.md, E8-deep-research-agent-case-study.md
F-network-engineering: F1-fundamentals-of-networking.md, F2-internet-protocol.md, F3-user-datagram-protocol.md, F4-transmission-control-protocol.md, F5-popular-networking-protocols.md, F6-network-performance.md, F7-network-routing.md, F8-analyzing-protocols-with-wireshark.md, F9-answering-your-questions.md
G-cloud-network-architecture: G1-virtual-network-fundamentals.md, G2-additional-virtual-network-features.md, G3-network-dns-and-dhcp.md, G4-network-performance-and-optimization.md, G5-traffic-monitoring-troubleshooting.md, G6-private-connectivity-peering.md, G7-service-endpoints-private-link.md, G8-transit-hub.md, G9-hybrid-network-basics.md, G10-site-to-site-vpn.md, G11-client-vpn.md, G12-dedicated-interconnect.md, G13-managed-global-wan.md, G14-service-to-service-networking.md, G15-additional-course-topics.md
H-full-stack-troubleshooting: H1-linux-network-diagnostics.md, H2-troubleshooting-your-network.md, H3-domain-name-system.md, H4-transport-layer-security.md, H5-network-performance-deep-dive.md, H6-web-application-architecture.md
I-dns-tls-acceleration-gaps: I1-dns.md, I2-tls-and-certificates.md, I3-acceleration.md
J-sre: J1-slis-slos-error-budgets.md, J2-monitoring-and-alerting.md, J3-observability.md, J4-incident-response-postmortems.md, J5-capacity-planning-load-testing.md, J6-toil-release-engineering.md, J7-chaos-engineering.md
K-ai-infra-llm: K1-llm-fundamentals-for-infra.md, K2-embeddings-vector-databases.md, K3-rag-pipelines.md, K4-llm-serving-inference.md, K5-training-fine-tuning.md, K6-managed-model-platforms.md, K7-ai-gateways-caching-cost.md, K8-agents-tool-use-mcp.md, K9-llmops-evals-guardrails.md
L-data-privacy-ai-security: L1-data-classification-pii.md, L2-encryption-key-management.md, L3-residency-compliance.md, L4-ai-security-threats.md, L5-model-data-governance.md, L6-secrets-supply-chain.md, L7-zero-trust-workload-identity.md
M-data-platforms: M1-lakehouse-table-formats.md, M2-spark-at-scale.md, M3-databricks-platform.md, M4-kafka-at-scale.md, M5-stream-processing.md, M6-orchestration-etl.md, M7-data-warehouses.md
K-ai-infra-llm (added): K10-ml-fundamentals.md
N-cicd-platform-engineering: N1-github-actions.md, N2-gitlab-ci-jenkins.md, N3-azure-devops-aws-codepipeline.md, N4-gitops-argocd-flux.md, N5-iac-pipelines-policy-as-code.md, N6-internal-developer-platforms.md, N7-openshift.md
O-observability-tooling: O1-prometheus.md, O2-grafana-lgtm-stack.md, O3-elastic-stack.md, O4-datadog.md, O5-zabbix.md
P-security-platforms-identity: P1-wiz-cnapp.md, P2-crowdstrike-edr-xdr.md, P3-soc2-iso-compliance-operations.md, P4-identity-providers.md
Q-industry-domains: Q1-healthcare-interoperability.md, Q2-healthcare-cloud-hipaa-engineering.md, Q3-payment-processing-integration.md
R-support-communication: R1-incident-communication.md, R2-ticket-handling-jira.md, R3-writing-for-audiences.md
Anchor format: GitHub-style lowercase slug of the heading, e.g. "## C1.27 Caching for performance" → #c127-caching-for-performance
