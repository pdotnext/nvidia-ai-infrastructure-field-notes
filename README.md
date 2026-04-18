# NVIDIA AI Infrastructure — Field Notes

> **Infrastructure Architect → AI Solutions Architect**
> 18 years enterprise data centre. Now building the infrastructure that runs AI.

---

## What This Repository Is

This repository documents a structured, source-verified learning journey into NVIDIA AI infrastructure — built for an Solutions Architect role in the DACH industrial sector.

Every file here comes from a primary source: NVIDIA documentation, SuperPOD design guides, official specifications. Where a spec is unverified or from secondary sources, it is flagged explicitly. Nothing is stated from memory alone.

This is operational knowledge, not exam notes.

---

## Certification

| Credential | Score | Date |
|---|---|---|
| NVIDIA NCA-AIIO — AI Infrastructure & Operations Associate | AI Infra: 72% · AI Ops: 90% · Intro AI: 83% | April 2026 ✅ |
| NCP-AII — AI Infrastructure Professional | — | Target: Q3 2026 |

---

## Repository Structure

```
nvidia-ai-infrastructure-field-notes/
│
├── gpu-health/
│   ├── xid-error-reference.md          # XID codes, meaning, response sequence
│   ├── daily-monitoring-checklist.md   # nvidia-smi + DCGM daily sequence
│   └── dcgm-diagnostic-guide.md        # dcgmi diag levels, ECC counters
│
├── dc-power/
│   ├── dgx-h100-power-architecture.md  # 3-feed requirement, PSU redundancy
│   ├── pue-and-efficiency-metrics.md   # PUE formula, EU mandate, tokens/joule
│   └── voltage-stability-gpu-training.md
│
├── benchmarking/
│   ├── hpl-validation-guide.md         # HPL methodology, targets, failure modes
│   └── nccl-allreduce-guide.md         # busbw vs algbw, single/multi-node targets
│
├── kubernetes/
│   ├── gpu-operator-architecture.md    # DaemonSet stack, dependency chain
│   ├── cluster-policy-operations.md    # ClusterPolicy, drift detection
│   └── mig-partitioning-strategy.md   # MIG profiles, workload design
│
└── networking/
    ├── nvlink-topology.md              # NVLink vs NVSwitch, topology matrix
    ├── infiniband-health-checks.md     # ibstat, perfquery, ib_write_bw
    └── nccl-fabric-validation.md       # Multi-node NCCL, IB fabric verification
```

*Folders populate as modules complete. Active updates through July 2026.*

---

## Background

18 years enterprise infrastructure at VMware/Broadcom, Credit Suisse, and technology companies across Stuttgart, Abu Dhabi, and Singapore.

Core competencies transferring directly into AI infrastructure:
- Infrastructure design at scale (2,000+ ESXi hosts, VCF, vSAN)
- Network virtualisation (NSX-T, large-scale ACI → NSX-T migration)
- Control plane operations (vCenter, BCM patterns, cluster lifecycle)
- Automation (Ansible, Python, REST API)
- Customer architecture — pre-sales, regulated industries, DACH enterprise

Gaps being closed in this repository: GPU-specific diagnostics, HBM memory behaviour, NVLink/NVSwitch fabric management, InfiniBand at scale, HPL/NCCL benchmarking methodology, liquid cooling for Blackwell-generation hardware.

---

## 90-Day Gap Closing Plan (April → July 2026)

**Phase 1 — GPU Stack and Kubernetes (April–May)**
MIG partitioning, GPU Operator on OpenShift, time-slicing vs MPS, BCM/Slurm/Run:ai, NIM/Triton inference stack

**Phase 2 — Hardware, Networking, DC Infrastructure (May–June)**
InfiniBand fat-tree topology, RoCE vs IB decision framework, BlueField DPU, GPUDirect RDMA/Storage, DLC for Blackwell, HPL/NCCL burn-in methodology

**Phase 3 — Portfolio and DACH Market (June–July)**
Three running portfolio projects, EU AI Act positioning, automotive/manufacturing AI workload patterns, on-prem vs DGX Cloud decision framework

---

## Portfolio Projects (Phase 3 — June 2026)

| Project | Stack | Purpose |
|---|---|---|
| GPU Cluster Monitor Dashboard | Python · DCGM · Prometheus · Grafana | Operational monitoring for DGX clusters |
| AI Infrastructure Sizing Calculator | Python · Streamlit | Pre-sales GPU sizing for DACH industrial workloads |
| DGX Deployment Scenario Playbook | Markdown · Architecture diagrams | 5 customer scenarios, Stuttgart/Frankfurt/Munich context |

---

## Contact

- LinkedIn: [linkedin.com/in/preetam-ai-dev](https://www.linkedin.com/in/preetam-ai-dev)
- Location: Stuttgart, Germany
- Open to: AI Infrastructure Solutions Architect roles, DACH region

---

*Building in public. Every file is a verified learning session.*
*Active updates through July 2026.*
