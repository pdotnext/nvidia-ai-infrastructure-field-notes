# NVIDIA AI Infrastructure — Field Notes

## About this repository

Last 18 years I have been in the Enterprise Infrastructure,
currently working as Sr Technical Adoption Manager at Broadcom with
rich experience — VxRail, VCF, NSX-T, vSAN and large-scale VMware deployments.

In 2025 I made a important decision to upskill into AI GPU
infrastructure. This repository documents that learning journey —
operationally, not theoretically.

Everything here is built from primary sources: NVIDIA documentation,
SuperPOD design guides, and hands-on lab sessions. Where I am
uncertain, I say so. Where I have verified a spec from official
documentation, I link the source.

**NVIDIA AIIO Associate — in progress** [comming soon]
---

## Folder structure brief

- `/gpu-health/` — GPU monitoring runbooks, XID error reference,
  DCGM diagnostic sequences
- `/dc-power/` — DGX H100 power architecture, 3-feed requirement,
  PUE and efficiency metrics
- `/benchmarking/` — HPL and NCCL validation guides with
  expected outputs
- `/kubernetes/` — GPU Operator stack, DaemonSet architecture,
  device plugin
- `/networking/` — NVLink topology, InfiniBand health checks,
  NCCL allreduce validation

---

## Background

Coming from VMware/Broadcom infrastructure, I found that roughly
70% of the operational discipline transfers directly —
Infrastructure design thinking, control plane management, large-scale deployment
patterns, BMC/IPMI automation.

Goal here is to fill the gaps focused on AI esp GPU-specific diagnostics, HBM memory behaviour,
NVLink fabric management, HPL/NCCL benchmarking. This repo is
where I close those gaps in public.

---

## Contact

LinkedIn: https://www.linkedin.com/in/preetam-ai-dev/

Currently based in Stuttgart, Germany.

Open to AI infrastructure roles — NVIDIA SA track.

