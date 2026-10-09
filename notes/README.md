# Infra, SRE, AI & System Design — Interview Notes

Senior/staff-level notes for DevOps, SRE, AI engineers, network engineers, DevSecOps and architects. Each file: TL;DR, one section per topic ID, mermaid diagrams, AWS ↔ Azure mapping where infra is non-trivial, hands-on (bash / Docker / Terraform only), cross-links and official sources. Last verified 2026-10.

Source curriculum: [`../infra-architecture-curriculum.md`](../infra-architecture-curriculum.md) · Authoring rules: [`_AUTHORING_GUIDE.md`](_AUTHORING_GUIDE.md)

## Suggested study order

A → F → H → B → G → C → D → E, with I alongside related sections; then J (SRE), K (AI infra), L (privacy & AI security), M (data platforms), N (CI/CD & platform), O (observability tools), P (security platforms & identity), Q (healthcare & payments), R (support & communication). Language refreshers: [../languages/](../languages/README.md).

## Sections

### [A. Operating Systems](A-operating-systems/README.md)

- [A1 Why an OS?](A-operating-systems/A1-why-an-os.md) — 2 topics
- [A2 The Anatomy of a Process](A-operating-systems/A2-the-anatomy-of-a-process.md) — 7 topics
- [A3 Memory Management](A-operating-systems/A3-memory-management.md) — 5 topics
- [A4 Inside the CPU](A-operating-systems/A4-inside-the-cpu.md) — 4 topics
- [A5 Process Management](A-operating-systems/A5-process-management.md) — 5 topics
- [A6 Storage Management](A-operating-systems/A6-storage-management.md) — 4 topics
- [A7 Socket Management](A-operating-systems/A7-socket-management.md) — 4 topics
- [A8 More OS Concepts](A-operating-systems/A8-more-os-concepts.md) — 3 topics
- [A9 Bonus](A-operating-systems/A9-bonus.md) — 5 topics

### [F. Network Engineering](F-network-engineering/README.md)

- [F1 Fundamentals of Networking](F-network-engineering/F1-fundamentals-of-networking.md) — 3 topics
- [F2 Internet Protocol (IP)](F-network-engineering/F2-internet-protocol.md) — 7 topics
- [F3 User Datagram Protocol (UDP)](F-network-engineering/F3-user-datagram-protocol.md) — 4 topics
- [F4 Transmission Control Protocol (TCP)](F-network-engineering/F4-transmission-control-protocol.md) — 10 topics
- [F5 Overview of Popular Networking Protocols](F-network-engineering/F5-popular-networking-protocols.md) — 3 topics
- [F6 Network Performance](F-network-engineering/F6-network-performance.md) — 10 topics
- [F7 Network Routing](F-network-engineering/F7-network-routing.md) — 2 topics
- [F8 Analyzing Protocols with Wireshark](F-network-engineering/F8-analyzing-protocols-with-wireshark.md) — 5 topics
- [F9 Answering your Questions](F-network-engineering/F9-answering-your-questions.md) — 2 topics

### [H. Full-Stack Troubleshooting](H-full-stack-troubleshooting/README.md)

- [H1 Linux Network Diagnostics](H-full-stack-troubleshooting/H1-linux-network-diagnostics.md) — 5 topics
- [H2 Troubleshooting Your Network](H-full-stack-troubleshooting/H2-troubleshooting-your-network.md) — 7 topics
- [H3 Domain Name System (DNS)](H-full-stack-troubleshooting/H3-domain-name-system.md) — 5 topics
- [H4 Transport Layer Security (SSL/TLS)](H-full-stack-troubleshooting/H4-transport-layer-security.md) — 7 topics
- [H5 Network Performance Deep Dive](H-full-stack-troubleshooting/H5-network-performance-deep-dive.md) — 8 topics
- [H6 Web Application Architecture](H-full-stack-troubleshooting/H6-web-application-architecture.md) — 13 topics

### [B. Database Engineering](B-database-engineering/README.md)

- [B1 ACID](B-database-engineering/B1-acid.md) — 8 topics
- [B2 Understanding Database Internals](B-database-engineering/B2-database-internals.md) — 4 topics
- [B3 Database Indexing](B-database-engineering/B3-database-indexing.md) — 13 topics
- [B4 B-Tree vs B+Tree in Production Database Systems](B-database-engineering/B4-btree-vs-bplustree.md) — 7 topics
- [B5 Database Partitioning](B-database-engineering/B5-database-partitioning.md) — 8 topics
- [B6 Database Sharding](B-database-engineering/B6-database-sharding.md) — 8 topics
- [B7 Concurrency Control](B-database-engineering/B7-concurrency-control.md) — 7 topics
- [B8 Database Replication](B-database-engineering/B8-database-replication.md) — 4 topics
- [B9 Database System Design](B-database-engineering/B9-database-system-design.md) — 2 topics
- [B10 Database Engines](B-database-engineering/B10-database-engines.md) — 11 topics
- [B11 Database Cursors](B-database-engineering/B11-database-cursors.md) — 5 topics
- [B12 Database Security](B-database-engineering/B12-database-security.md) — 6 topics
- [B13 Homomorphic Encryption: Performing Database Queries on Encrypted Data](B-database-engineering/B13-homomorphic-encryption.md) — 5 topics
- [B14 Database Discussions](B-database-engineering/B14-database-discussions.md) — 8 topics

### [G. Cloud Network Architecture (AWS ↔ Azure)](G-cloud-network-architecture/README.md)

- [G1 Virtual Network Fundamentals](G-cloud-network-architecture/G1-virtual-network-fundamentals.md) — 14 topics
- [G2 Additional Virtual Network Features](G-cloud-network-architecture/G2-additional-virtual-network-features.md) — 4 topics
- [G3 Network DNS and DHCP](G-cloud-network-architecture/G3-network-dns-and-dhcp.md) — 6 topics
- [G4 Network Performance and Optimization](G-cloud-network-architecture/G4-network-performance-and-optimization.md) — 6 topics
- [G5 Traffic Monitoring, Troubleshooting & Analysis](G-cloud-network-architecture/G5-traffic-monitoring-troubleshooting.md) — 4 topics
- [G6 Private Connectivity: Peering](G-cloud-network-architecture/G6-private-connectivity-peering.md) — 4 topics
- [G7 Private Connectivity: Service Endpoints & Private Link](G-cloud-network-architecture/G7-service-endpoints-private-link.md) — 14 topics
- [G8 Transit Hub](G-cloud-network-architecture/G8-transit-hub.md) — 18 topics
- [G9 Hybrid Network Basics](G-cloud-network-architecture/G9-hybrid-network-basics.md) — 4 topics
- [G10 Site-to-Site VPN](G-cloud-network-architecture/G10-site-to-site-vpn.md) — 13 topics
- [G11 Client VPN](G-cloud-network-architecture/G11-client-vpn.md) — 2 topics
- [G12 Dedicated Interconnect](G-cloud-network-architecture/G12-dedicated-interconnect.md) — 31 topics
- [G13 Managed Global WAN](G-cloud-network-architecture/G13-managed-global-wan.md) — 5 topics
- [G14 Service-to-Service Application Networking](G-cloud-network-architecture/G14-service-to-service-networking.md) — 6 topics
- [G15 Additional course topics](G-cloud-network-architecture/G15-additional-course-topics.md) — 8 topics

### [C. Large-Scale Architecture](C-large-scale-architecture/README.md)

- [C1 Performance](C-large-scale-architecture/C1-performance.md) — 30 topics
- [C2 Scalability](C-large-scale-architecture/C2-scalability.md) — 38 topics
- [C3 Reliability](C-large-scale-architecture/C3-reliability.md) — 32 topics
- [C4 Security](C-large-scale-architecture/C4-security.md) — 34 topics
- [C5 Deployment](C-large-scale-architecture/C5-deployment.md) — 30 topics
- [C6 Technology Stack](C-large-scale-architecture/C6-technology-stack.md) — 51 topics

### [D. System Design](D-system-design/README.md)

- [D1 System Design Basics](D-system-design/D1-system-design-basics.md) — 25 topics
- [D2 Reusable Parts of System Design](D-system-design/D2-reusable-parts-of-system-design.md) — 16 topics
- [D3 System Design of Modern Applications](D-system-design/D3-system-design-of-modern-applications.md) — 13 topics

### [E. AI System Design](E-ai-system-design/README.md)

- [E1 System Design Fundamentals](E-ai-system-design/E1-system-design-fundamentals.md) — 10 topics
- [E2 Google CTR Prediction System Case Study](E-ai-system-design/E2-google-ctr-prediction-case-study.md) — 4 topics
- [E3 HubSpot User Clustering Case Study](E-ai-system-design/E3-hubspot-user-clustering-case-study.md) — 3 topics
- [E4 Facebook Content Moderation Case Study](E-ai-system-design/E4-facebook-content-moderation-case-study.md) — 3 topics
- [E5 Smart Car Parking SaaS Case Study (computer vision)](E-ai-system-design/E5-smart-car-parking-case-study.md) — 2 topics
- [E6 AI Grammar Checker SaaS Case Study](E-ai-system-design/E6-ai-grammar-checker-case-study.md) — 3 topics
- [E7 AI Interview Chatbot Case Study](E-ai-system-design/E7-ai-interview-chatbot-case-study.md) — 3 topics
- [E8 Deep Research Agent Case Study](E-ai-system-design/E8-deep-research-agent-case-study.md) — 2 topics

### [I. DNS, TLS & Acceleration Gaps](I-dns-tls-acceleration-gaps/README.md)

- [I1 DNS](I-dns-tls-acceleration-gaps/I1-dns.md) — 4 topics
- [I2 TLS and Certificates](I-dns-tls-acceleration-gaps/I2-tls-and-certificates.md) — 4 topics
- [I3 Acceleration](I-dns-tls-acceleration-gaps/I3-acceleration.md) — 1 topics

### [J. SRE](J-sre/README.md)

- [J1 SLIs, SLOs & Error Budgets](J-sre/J1-slis-slos-error-budgets.md) — 8 topics
- [J2 Monitoring and Alerting](J-sre/J2-monitoring-and-alerting.md) — 11 topics
- [J3 Observability](J-sre/J3-observability.md) — 9 topics
- [J4 Incident Response & Postmortems](J-sre/J4-incident-response-postmortems.md) — 10 topics
- [J5 Capacity Planning & Load Testing](J-sre/J5-capacity-planning-load-testing.md) — 10 topics
- [J6 Toil, Automation & Release Engineering](J-sre/J6-toil-release-engineering.md) — 9 topics
- [J7 Chaos Engineering & Resilience Testing](J-sre/J7-chaos-engineering.md) — 9 topics

### [K. AI Infra & LLM Systems](K-ai-infra-llm/README.md)

- [K1 LLM Fundamentals for Infra](K-ai-infra-llm/K1-llm-fundamentals-for-infra.md) — 13 topics
- [K2 Embeddings & Vector Databases](K-ai-infra-llm/K2-embeddings-vector-databases.md) — 15 topics
- [K3 RAG Pipelines (production engineering)](K-ai-infra-llm/K3-rag-pipelines.md) — 11 topics
- [K4 LLM Serving & Inference Infrastructure](K-ai-infra-llm/K4-llm-serving-inference.md) — 11 topics
- [K5 Training & Fine-tuning Infrastructure](K-ai-infra-llm/K5-training-fine-tuning.md) — 10 topics
- [K6 Managed model platforms](K-ai-infra-llm/K6-managed-model-platforms.md) — 9 topics
- [K7 AI Gateways, Caching and Cost](K-ai-infra-llm/K7-ai-gateways-caching-cost.md) — 9 topics
- [K8 Agents, Tool Use and MCP](K-ai-infra-llm/K8-agents-tool-use-mcp.md) — 15 topics
- [K9 LLMOps, Evals and Guardrails](K-ai-infra-llm/K9-llmops-evals-guardrails.md) — 11 topics
- [K10 ML Fundamentals for Infra, SRE and AI Engineers](K-ai-infra-llm/K10-ml-fundamentals.md) — 16 topics

### [L. Data Privacy & AI Security](L-data-privacy-ai-security/README.md)

- [L1 Data classification & PII protection](L-data-privacy-ai-security/L1-data-classification-pii.md) — 8 topics
- [L2 Encryption & Key Management](L-data-privacy-ai-security/L2-encryption-key-management.md) — 10 topics
- [L3 Data residency, sovereignty & compliance frameworks](L-data-privacy-ai-security/L3-residency-compliance.md) — 14 topics
- [L4 AI security threats & defenses](L-data-privacy-ai-security/L4-ai-security-threats.md) — 14 topics
- [L5 Model & data governance](L-data-privacy-ai-security/L5-model-data-governance.md) — 10 topics
- [L6 Secrets management & software supply chain security](L-data-privacy-ai-security/L6-secrets-supply-chain.md) — 13 topics
- [L7 Zero trust & workload identity](L-data-privacy-ai-security/L7-zero-trust-workload-identity.md) — 10 topics

### [M. Data Platforms](M-data-platforms/README.md)

- [M1 Lakehouse & Open Table Formats](M-data-platforms/M1-lakehouse-table-formats.md) — 13 topics
- [M2 Spark at scale](M-data-platforms/M2-spark-at-scale.md) — 16 topics
- [M3 Databricks platform](M-data-platforms/M3-databricks-platform.md) — 11 topics
- [M4 Kafka at scale](M-data-platforms/M4-kafka-at-scale.md) — 14 topics
- [M5 Stream processing](M-data-platforms/M5-stream-processing.md) — 13 topics
- [M6 Orchestration & ETL/ELT](M-data-platforms/M6-orchestration-etl.md) — 11 topics
- [M7 Data Warehouses (Cloud OLAP)](M-data-platforms/M7-data-warehouses.md) — 13 topics

### [N. CI/CD & Platform Engineering](N-cicd-platform-engineering/README.md)

- [N1 GitHub Actions](N-cicd-platform-engineering/N1-github-actions.md) — 12 topics
- [N2 GitLab CI/CD and Jenkins](N-cicd-platform-engineering/N2-gitlab-ci-jenkins.md) — 16 topics
- [N3 Azure DevOps & AWS CodePipeline (cloud-native CI/CD)](N-cicd-platform-engineering/N3-azure-devops-aws-codepipeline.md) — 15 topics
- [N4 GitOps with Argo CD and Flux](N-cicd-platform-engineering/N4-gitops-argocd-flux.md) — 17 topics
- [N5 IaC Pipelines & Policy as Code](N-cicd-platform-engineering/N5-iac-pipelines-policy-as-code.md) — 15 topics
- [N6 Internal Developer Platforms (Platform Engineering, Backstage, Golden Paths)](N-cicd-platform-engineering/N6-internal-developer-platforms.md) — 13 topics
- [N7 OpenShift](N-cicd-platform-engineering/N7-openshift.md) — 17 topics

### [O. Observability Tooling](O-observability-tooling/README.md)

- [O1 Prometheus (deep dive)](O-observability-tooling/O1-prometheus.md) — 11 topics
- [O2 Grafana and the LGTM stack (Loki, Grafana, Tempo, Mimir + Alloy, Pyroscope)](O-observability-tooling/O2-grafana-lgtm-stack.md) — 12 topics
- [O3 Elastic Stack (Elasticsearch, Kibana, Agent/Fleet, APM) for observability and security](O-observability-tooling/O3-elastic-stack.md) — 14 topics
- [O4 Datadog](O-observability-tooling/O4-datadog.md) — 13 topics
- [O5 Zabbix](O-observability-tooling/O5-zabbix.md) — 15 topics

### [P. Security Platforms & Identity](P-security-platforms-identity/README.md)

- [P1 Wiz & CNAPP (Cloud-Native Application Protection)](P-security-platforms-identity/P1-wiz-cnapp.md) — 13 topics
- [P2 CrowdStrike, EDR & XDR](P-security-platforms-identity/P2-crowdstrike-edr-xdr.md) — 12 topics
- [P3 SOC 2 / ISO 27001 compliance operations](P-security-platforms-identity/P3-soc2-iso-compliance-operations.md) — 11 topics
- [P4 Identity providers (IdPs): SAML, OIDC, SCIM, Okta, Entra ID, Keycloak, and federating the cloud](P-security-platforms-identity/P4-identity-providers.md) — 15 topics

### [Q. Industry Domains (Healthcare, Payments)](Q-industry-domains/README.md)

- [Q1 Healthcare interoperability](Q-industry-domains/Q1-healthcare-interoperability.md) — 14 topics
- [Q2 Healthcare cloud & HIPAA engineering](Q-industry-domains/Q2-healthcare-cloud-hipaa-engineering.md) — 13 topics
- [Q3 Payment processing integration](Q-industry-domains/Q3-payment-processing-integration.md) — 19 topics

### [R. Support & Communication](R-support-communication/README.md)

- [R1 Incident Communication](R-support-communication/R1-incident-communication.md) — 11 topics
- [R2 Ticket handling etiquette & Jira (support engineering)](R-support-communication/R2-ticket-handling-jira.md) — 11 topics
- [R3 Writing for Audiences (technical and non-technical)](R-support-communication/R3-writing-for-audiences.md) — 14 topics

## Cross-cutting topics (where each is covered)

| Topic | IDs |
|---|---|
| Consistent hashing | [B6.2](B-database-engineering/B6-database-sharding.md), [D1.21](D-system-design/D1-system-design-basics.md) |
| Partitioning and sharding | [B5](B-database-engineering/B5-database-partitioning.md), [B6](B-database-engineering/B6-database-sharding.md), [C2.18](C-large-scale-architecture/C2-scalability.md)–C2.20, [D1.22](D-system-design/D1-system-design-basics.md) |
| Caching | [C1.26](C-large-scale-architecture/C1-performance.md)–C1.30, [C2.16](C-large-scale-architecture/C2-scalability.md), [C6.18](C-large-scale-architecture/C6-technology-stack.md)–C6.21, [D1.14](D-system-design/D1-system-design-basics.md), [D1.15](D-system-design/D1-system-design-basics.md), [H6.4](H-full-stack-troubleshooting/H6-web-application-architecture.md) |
| CDN | [C6.15](C-large-scale-architecture/C6-technology-stack.md), [D1.14](D-system-design/D1-system-design-basics.md), [H6.12](H-full-stack-troubleshooting/H6-web-application-architecture.md) |
| Replication | [B8](B-database-engineering/B8-database-replication.md), [C2.10](C-large-scale-architecture/C2-scalability.md), [C2.11](C-large-scale-architecture/C2-scalability.md) |
| Locking and concurrency | [B7](B-database-engineering/B7-concurrency-control.md), [C1.20](C-large-scale-architecture/C1-performance.md)–C1.24 |
| DR and standby | [C3.24](C-large-scale-architecture/C3-reliability.md)–C3.26, [D1.23](D-system-design/D1-system-design-basics.md), [D1.24](D-system-design/D1-system-design-basics.md) |
| TLS and certificates | [C4.5](C-large-scale-architecture/C4-security.md)–C4.11, [D2.11](D-system-design/D2-reusable-parts-of-system-design.md), [D2.12](D-system-design/D2-reusable-parts-of-system-design.md), [F5.2](F-network-engineering/F5-popular-networking-protocols.md), [F5.3](F-network-engineering/F5-popular-networking-protocols.md), [H4](H-full-stack-troubleshooting/H4-transport-layer-security.md) |
| Sockets and kernel queues | [A7.1](A-operating-systems/A7-socket-management.md), [F4.9](F-network-engineering/F4-transmission-control-protocol.md), [H6.7](H-full-stack-troubleshooting/H6-web-application-architecture.md) |
| L4 vs L7 load balancing and proxies | [C2.26](C-large-scale-architecture/C2-scalability.md), [D1.3](D-system-design/D1-system-design-basics.md)–D1.5, [F6.8](F-network-engineering/F6-network-performance.md), [F6.9](F-network-engineering/F6-network-performance.md), [H6.5](H-full-stack-troubleshooting/H6-web-application-architecture.md), [H6.6](H-full-stack-troubleshooting/H6-web-application-architecture.md) |
| Firewalls and ACLs | [C4.12](C-large-scale-architecture/C4-security.md), [C4.13](C-large-scale-architecture/C4-security.md), [D2.13](D-system-design/D2-reusable-parts-of-system-design.md), [G1.7](G-cloud-network-architecture/G1-virtual-network-fundamentals.md), [G1.8](G-cloud-network-architecture/G1-virtual-network-fundamentals.md), [H2.7](H-full-stack-troubleshooting/H2-troubleshooting-your-network.md) |
| MTU | [F6.1](F-network-engineering/F6-network-performance.md), [G4.1](G-cloud-network-architecture/G4-network-performance-and-optimization.md), [G12.27](G-cloud-network-architecture/G12-dedicated-interconnect.md), [H5.8](H-full-stack-troubleshooting/H5-network-performance-deep-dive.md) |
| DNS | [F5.1](F-network-engineering/F5-popular-networking-protocols.md), [G3](G-cloud-network-architecture/G3-network-dns-and-dhcp.md), [H3](H-full-stack-troubleshooting/H3-domain-name-system.md) |
| Latency | [C1.6](C-large-scale-architecture/C1-performance.md)–C1.15, [F6](F-network-engineering/F6-network-performance.md), [H5](H-full-stack-troubleshooting/H5-network-performance-deep-dive.md) |
| Diagnostic tools | [F2.3](F-network-engineering/F2-internet-protocol.md), [F2.5](F-network-engineering/F2-internet-protocol.md), [F8](F-network-engineering/F8-analyzing-protocols-with-wireshark.md), [H1](H-full-stack-troubleshooting/H1-linux-network-diagnostics.md), [H2](H-full-stack-troubleshooting/H2-troubleshooting-your-network.md) |

## Notes on accuracy

- Facts were checked against official docs where possible; anything not confirmed is tagged `(unverified)` in-line.
- Cloud services change quickly — re-check limits, prices and retirement dates before relying on them.
