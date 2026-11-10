---
title: "ZT Gap Tests vs. HBN-OVNK Regression Plan (docx)"
---

# Zero-Trust Gap Tests vs. HBN-OVNK Regression Plan

Comparison of **`zero-trust-ocp-gap-tests.md`** (ZT behaviours that today run only
on K8s and are proposed to be added on OCP) against **`DPF QA Test Plan_ RedHat
Integration (OCP) - HBN-OVNK Regression.docx`** (manual regression plan, `TC-*`).

> **Context:** the docx is scoped to **Trust Mode = Host-Trusted (Trusted BM)**.
> The question for each gap test is: does the docx already have an OCP test case
> covering the same behaviour? Where it does, the behaviour has host-trusted OCP
> coverage and the ZT gap is "add the ZT variant"; where it doesn't, the gap is
> net-new on OCP for *any* trust mode.

Legend:
- ✅ **Covered** — docx has an equivalent OCP test case.
- 🟡 **Partial** — docx touches part of it (narrower / adjacent scope).
- ❌ **Not covered** — no docx test case; true gap on OCP.
- 🔒 **ZT-only** — zero-trust security posture; by design absent from the host-trusted docx.

---

## 1. Summary

| Bucket | Gap tests | ✅ | 🟡 | ❌ / 🔒 |
|---|---|---|---|---|
| Deployment / IPAM / DPUService | 10 | 2 | 6 | 2 |
| Cluster DNS wiring | 4 | 0 | 1 | 3 |
| Observability (OTel / logs / traces / metrics flow) | 11 | 0 | 1 | 10 |
| OperatorConfig | 5 | 2 | 1 | 2 |
| Security posture (secrets / etcd / BMC) | 4 | 0 | 0 | 4 🔒 |
| Misc (operatorconfig deletion, DMA SF) | 2 | 0 | 1 | 1 |
| **Total** | **36** | **6** | **10** | **20** |

Headline: of 36 ZT gap tests, only **6** have a direct docx (host-trusted OCP)
counterpart and **10** are partially adjacent; **20** have no OCP coverage at all
(14 functional/observability gaps + 4 ZT-only security + misc).

---

## 2. Detailed matrix (gap test → docx coverage)

### Deployment / IPAM / DPUService
| Gap test | Status | Docx TC | Notes |
|---|---|---|---|
| dpu_deployment_create | ✅ | TC-INST-001, TC-DPUD-001 | DPUDeployment created & reconciled / recreated |
| dpuservice_delete_cleanup | ✅ | TC-SVC-009, TC-DPUD-003 | Delete DPUService / system cleanup after deletion |
| dpu_deployment_metrics | 🟡 | TC-SVC-010/011 | docx covers DTS→Prometheus metrics, not DPUDeployment/DPUServiceInterface metrics specifically |
| dpuservice_metrics | 🟡 | TC-SVC-010/011 | As above |
| dpu_deployment_bad_param_delete | 🟡 | TC-SVCUPG-002/005, TC-UPG-004 | docx has disruptive upgrade + prereq error handling; bad-param-never-ready delete not explicit |
| ipam_render_ippools | 🟡 | TC-SVC-004 | docx edits DPUServiceIPAM; render of IPv4/IPv6 IPPools from subnet config not explicit |
| ipam_render_cidrpools | 🟡 | TC-SVC-004 | As above, CIDRPools from network config |
| ipam_workload_alloc_v6 | 🟡 | TC-NET-001 | docx verifies connectivity/service; IPv6 workload allocation not explicit |
| ipam_workload_alloc_v4v6 | ❌ | — | Dual-stack (v4+v6) allocation not in docx |
| imagepullsecrets_sync | ❌ | — | No docx case for ImagePullSecret sync/cleanup to DPU cluster |

### Cluster DNS wiring
| Gap test | Status | Docx TC | Notes |
|---|---|---|---|
| cluster_dns_host_coredns | 🟡 | TC-NET-001 | docx verifies service resolution generally; OpenShift `dns-default` running assumed, not asserted as a test |
| cluster_dns_coredns_disabled | ❌ | — | Kamaji CoreDNS addon disabled when host serves DNS — not in docx |
| cluster_dns_dpu_points_to_host | ❌ | — | DPU-cluster DNS Service → host CoreDNS — not in docx |
| cluster_dns_own_dns | ❌ | — | Non-host-served DPUCluster keeps own DNS — not in docx |

### Observability — OTel / logs / traces / metrics flow
| Gap test | Status | Notes |
|---|---|---|
| otel_daemonsets_host | ❌ | No OpenTelemetry coverage in docx |
| otel_daemonsets_dpu | ❌ | No OTel coverage |
| otel_dpu_collector_config | ❌ | No OTel coverage |
| otel_user_collector | ❌ | No OTel coverage |
| log_flow_mgmt_to_loki | ❌ | docx log section (TC-LOG-*) is SOS/dpfctl/must-gather collection, not Loki forwarding |
| log_flow_dpu_to_loki | ❌ | As above |
| log_flow_kamaji_audit | ❌ | No Kamaji audit-log forwarding in docx |
| metrics_flow_dpu_to_host | 🟡 | Adjacent to TC-SVC-010/011 (Prometheus metrics); host-collector forwarding not asserted |
| trace_flow_controller_to_tempo | ❌ | No tracing/Tempo coverage in docx |
| trace_flow_sfc_to_tempo | ❌ | No tracing coverage |
| trace_flow_dpu_single_trace | ❌ | No tracing coverage |

### OperatorConfig
| Gap test | Status | Docx TC | Notes |
|---|---|---|---|
| opcfg_system_overrides | ✅ | TC-OPCFG-001/002 | CPU/memory + component image overrides |
| opcfg_mtu_update | ✅ | TC-MTU-001/002/003 | Changing MTUs in operatorConfig updates DPU clusters |
| opcfg_overrides_path | 🟡 | TC-OPCFG-002 | Component override exists; "overrides path for system DPUServices" not explicit |
| opcfg_flannel_podcidr | ❌ | — | No flannel podCIDR change test in docx |
| opcfg_privileged_pod_enforcement | ❌ | — | No PrivilegedPodEnforcement / VAP-lifecycle test in docx |

### Security posture (zero-trust-only)
| Gap test | Status | Notes |
|---|---|---|
| encrypted_secrets | 🔒 | OpenBao/External Secrets — absent from host-trusted docx |
| etcd_encryption_at_rest | 🔒 | Kamaji etcd encryption — absent from docx |
| bmc_factory_reset | 🔒 | BMC factory reset + password hardening — absent from docx |
| bmc_cert_rotation | 🔒 | BMC mTLS server-cert rotation — absent from docx |

### Misc
| Gap test | Status | Docx TC | Notes |
|---|---|---|---|
| operatorconfig_deletion | 🟡 | TC-UNINST-001 | DPF uninstall removes objects; precise "deletion removes expected objects" overlaps |
| dma_sf_consumable | ❌ | — | DMA SF (created by dpu-agent) consumable — not in docx |

---

## 3. Takeaways

- **Overlap with docx (host-trusted OCP):** deployment create, DPUService
  delete/cleanup, operator-config overrides, and MTU updates (✅); IPAM render,
  service/deployment metrics, and bad-param upgrade are adjacent (🟡). These are
  "port the existing OCP behaviour to a ZT run" gaps.
- **Observability is the largest hole:** none of the OTel / Loki / Tempo / trace /
  metrics-flow gap tests have *any* docx counterpart — net-new on OCP regardless
  of trust mode.
- **DNS wiring and advanced IPAM (dual-stack, pool render)** are mostly uncovered
  by the docx.
- **Security posture (🔒)** — encrypted secrets, etcd encryption, BMC factory
  reset, BMC cert rotation — is intentionally absent from the host-trusted docx and
  is pure zero-trust coverage to add.
