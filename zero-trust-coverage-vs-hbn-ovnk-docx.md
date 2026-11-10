---
title: "Zero-Trust Coverage vs. HBN-OVNK Regression Plan (docx)"
---

# Zero-Trust Coverage vs. HBN-OVNK Regression Plan

Comparison of **`zero-trust-ocp-test-plan.md`** (the ZeroTrust OCP automated e2e
plan, cases `OCP-ZT-0xx`) against **`DPF QA Test Plan_ RedHat Integration (OCP) -
HBN-OVNK Regression.docx`** (the manual regression plan, cases `TC-*`).

> **Important context:** the docx is explicitly scoped to **Trust Mode =
> Host-Trusted (Trusted BM)** (see its *Test Configuration* table). It is *not* a
> zero-trust plan. The matrix below marks, for each docx test case, whether the
> ZT automated plan covers the same behaviour on OpenShift in ZeroTrust mode.

Legend:
- ✅ **Covered** — a ZT plan case (`OCP-ZT-*`) exercises the same behaviour.
- 🟡 **Partial** — ZT plan touches part of it (platform-adapted / narrower scope).
- ❌ **Not covered** — no corresponding ZT case; gap for ZT (or out of ZT scope).
- 🚫 **N/A to ZT** — explicitly Host-Trusted-only; does not apply to ZeroTrust.

---

## 1. Coverage summary

| Area | Docx cases | ZT-covered | Partial | Not covered / N/A |
|---|---|---|---|---|
| Install & upgrade | 4 | 1 | 0 | 3 |
| Network | 4 | 0 | 1 | 3 |
| DPUDeployment lifecycle | 6 | 1 | 2 | 3 |
| DPUService operations | 12 | 2 | 3 | 7 |
| DPUService upgrades | 7 | 1 | 0 | 6 (2 Trusted-only) |
| DPF release upgrade | 4 | 0 | 0 | 4 |
| DPFOperatorConfig | 2 | 0 | 1 | 1 |
| MTU | 5 | 0 | 2 | 3 |
| NodeEffect | 2 | 0 | 0 | 2 |
| Critical services | 6 | 0 | 0 | 6 |
| Parallel provisioning | 4 | 1 | 1 | 2 |
| Service dependencies | 6 | 0 | 0 | 6 |
| Resiliency & recovery | 17 | 0 | 1 | 16 |
| Uninstall | 2 | 0 | 0 | 2 |
| Stability | 6 | 0 | 0 | 6 |
| BFB | 4 | 0 | 0 | 4 |
| Log collection | 3 | 0 | 0 | 3 |
| Conformance / Kubevirt / HCP / Kata | 4 | 0 | 0 | 4 |

ZT plan cases with **no docx counterpart** (ZT-only, mostly security posture):
`OCP-ZT-012` (credential request), `OCP-ZT-013` (service template),
`OCP-ZT-023` (NodeSRIOVDevicePlugin webhook), `OCP-ZT-030` (etcd encryption),
`OCP-ZT-031` (encrypted secrets / OpenBao), `OCP-ZT-032` (BMC factory reset),
`OCP-ZT-033` (BMC server-cert rotation).

---

## 2. Detailed matrix (docx case → ZT coverage)

### Section 1 — Installation & Upgrade
| Docx TC | Status | ZT case | Notes |
|---|---|---|---|
| TC-INST-001 Install on OCP 4.22.z | ✅ | OCP-ZT-001/002/003/004 | Suite lifecycle & health assert operator Ready, config Ready, DPUCluster Ready, DPU-agent status |
| TC-INST-002 Host-Trusted Mode | 🚫 | — | ZT equivalent is OCP-ZT-002 asserting `DeploymentMode=ZeroTrust`; host-trusted mode itself is N/A to ZT |
| TC-INST-003 Mixed DPU / non-DPU workers | ❌ | — | No ZT case |
| TC-INST-004 OCP 4.22.x→4.22.z upgrade | ❌ | — | OCP/platform upgrade out of ZT plan scope |

### Section 2 — Network
| Docx TC | Status | ZT case | Notes |
|---|---|---|---|
| TC-NET-001 Connectivity & service verification | 🟡 | OCP-ZT-020 | ZT covers DNS resolution of a Service from a DPU pod (`dns-default`); broader connectivity not covered |
| TC-NET-002 Secondary CNI NAD traffic | ❌ | — | NAD cross-ns attach is an *expected skip* in ZT (Multus namespaceIsolation) |
| TC-NET-003 iperf3 offload | ❌ | — | No ZT case |
| TC-NET-004 RDMA offload | ❌ | — | No ZT case |

### Section 3 — DPUDeployment Lifecycle
| Docx TC | Status | ZT case | Notes |
|---|---|---|---|
| TC-DPUD-001 Delete & recreate DPUDeployment | ✅ | OCP-ZT-014 | New deployment becomes Ready; existing reused |
| TC-DPUD-002 Change BFB | ❌ | — | Reprovision/BFB path not driven (reuse mode) |
| TC-DPUD-003 System cleanup after deletion | 🟡 | OCP-ZT-011 | Chain Sets cleaned up on delete; full DPUDeployment cleanup not asserted |
| TC-DPUD-004 Recovery after child-object deletion | 🟡 | OCP-ZT-042 | DPUSet reprovisions deleted DPU CRs; other child objects not covered |
| TC-DPUD-005 Child-object immutability | ❌ | — | No ZT case |
| TC-DPUD-006 Scale DPUs up/down by selector | ❌ | — | No ZT case |

### Section 4 — DPUService Operations
| Docx TC | Status | ZT case | Notes |
|---|---|---|---|
| TC-SVC-001 Update HBN DPUService | 🟡 | OCP-ZT-014 | Disruptive upgrade of standard DPUServices covered generically |
| TC-SVC-002 Update OVN DPUService | 🟡 | OCP-ZT-014 | As above |
| TC-SVC-003 DPUServiceChain update | 🟡 | OCP-ZT-011 | Chain mirrored to each cluster; update path partially |
| TC-SVC-004 Edit DPUServiceIPAM | ✅ | OCP-ZT-010 | IPAM reconcile + webhook validation |
| TC-SVC-005 Delete DPUServiceIPAM w/ active DPUs | ✅ | OCP-ZT-010 | Covered under IPAM suite |
| TC-SVC-006 NVUE via NodePort | ❌ | — | No ZT case |
| TC-SVC-007..009 DTS install/update/delete | ❌ | — | DTS lifecycle not in ZT plan |
| TC-SVC-010 DTS metrics to DPU Prometheus | 🟡 | OCP-ZT-022 | Metrics collection reachable host+DPU; DTS-specific not asserted |
| TC-SVC-011 DTS metrics via Prometheus | 🟡 | OCP-ZT-021 | PromQL assertions are an *expected skip* on OCP (thanos-querier) |
| TC-SVC-012 DPU services running (ignition) | ❌ | — | No ZT case |

### Section 5 — DPUService Upgrades
| Docx TC | Status | ZT case | Notes |
|---|---|---|---|
| TC-SVCUPG-001 Non-disruptive upgrade | ❌ | — | No ZT case |
| TC-SVCUPG-002 Disruptive upgrade HBN (Trusted only) | 🚫 | OCP-ZT-014 (host-trusted) | Marked Trusted-only in docx; ZT-014 note covers disruptive upgrade of standard services |
| TC-SVCUPG-003 Disruptive InCluster (Trusted only) | 🚫 | — | In-cluster disruptive upgrade is an *expected skip* in ZT (restricted-v2 SCC) |
| TC-SVCUPG-004 Add service to deployment | ❌ | — | No ZT case |
| TC-SVCUPG-005 Sequential disruptive upgrade | ❌ | — | No ZT case |
| TC-SVCUPG-006 Upgrade DOCA-version annotated service | 🟡 | OCP-ZT-013 | Template status/versions reflect chart annotations |
| TC-SVCUPG-007 Upgrade while upgrade in progress | ❌ | — | No ZT case |

### Section 6 — DPF Release Upgrade
| Docx TC | Status | Notes |
|---|---|---|
| TC-UPG-001..004 | ❌ | DPF upgrade explicitly out of scope of the ZT plan (covered by upgrade suites) |

### Section 7 — DPFOperatorConfig
| Docx TC | Status | ZT case | Notes |
|---|---|---|---|
| TC-OPCFG-001 CPU/memory override | ❌ | — | No ZT case |
| TC-OPCFG-002 Component image override | 🟡 | OCP-ZT-040/041 | ZT covers flannel-MTU and MaxDPUParallelInstallations operator-config changes, not image override |

### Section 8 — MTU
| Docx TC | Status | ZT case | Notes |
|---|---|---|---|
| TC-MTU-001 HBN DPUServiceChain MTU | 🟡 | OCP-ZT-040 | ZT asserts flannel ConfigMap MTU==1500 |
| TC-MTU-002 Deployed-service MTU | ❌ | — | No ZT case |
| TC-MTU-003 ControlplaneMTU before deployment | 🟡 | OCP-ZT-040 | Partial via flannel MTU operator-config |
| TC-MTU-004 Delete highspeedMTU fallback | ❌ | — | No ZT case |
| TC-MTU-005 Negative: service MTU > highspeedMTU | ❌ | — | No ZT case |

### Section 9 — NodeEffect
| Docx TC | Status | Notes |
|---|---|---|
| TC-NE-001/002 | ❌ | No ZT case |

### Section 10 — Critical DPUServices Label
| Docx TC | Status | Notes |
|---|---|---|
| TC-CRIT-001..006 | ❌ | No ZT case |

### Section 11 — Parallel Provisioning
| Docx TC | Status | ZT case | Notes |
|---|---|---|---|
| TC-PAR-001 Multi-DPU same host | ❌ | — | No ZT case |
| TC-PAR-002 Multiple node effects | ❌ | — | No ZT case |
| TC-PAR-003 maxDPUParallelInstallations=1 | ✅ | OCP-ZT-041 | Provisioning controller restarts on change |
| TC-PAR-004 Increase during provisioning | 🟡 | OCP-ZT-041 | Change behaviour covered; mid-provisioning increase not asserted |

### Section 12 — Service Dependencies
| Docx TC | Status | Notes |
|---|---|---|
| TC-DEP-001..006 | ❌ | No ZT case |

### Sections 13–15 — Resiliency & Recovery
| Docx TC | Status | ZT case | Notes |
|---|---|---|---|
| TC-RES-001..015, 017 | ❌ | — | Host/DPU/cluster reboot & power paths out of ZT reuse-mode scope |
| TC-RES-016 Reprovision — delete DPUSet | 🟡 | OCP-ZT-042 | DPUSet reprovisions deleted DPU CRs (host-trusted OnDelete) |

### Section 16 — Uninstall
| Docx TC | Status | Notes |
|---|---|---|
| TC-UNINST-001/002 | ❌ | ZT plan leaves config in place on teardown; uninstall not driven |

### Section 17 — Stability
| Docx TC | Status | Notes |
|---|---|---|
| TC-STAB-001..006 | ❌ | No ZT case (stability/soak out of scope) |

### Section 18 — BFB
| Docx TC | Status | Notes |
|---|---|---|
| TC-BFB-001 SecureBoot provisioning | ❌ | Provisioning/Redfish path not driven in ZT reuse mode |
| TC-BFB-002..004 | ❌ | No ZT case |

### Sections 19–23 — Logs / Conformance / Kubevirt / HCP / Kata
| Docx TC | Status | Notes |
|---|---|---|
| TC-LOG-001..003 | ❌ | No ZT case |
| TC-CONF-001 Conformance | ❌ | No ZT case |
| TC-VIRT-001 Kubevirt + UDN | ❌ | No ZT case |
| TC-HCP-001 HCP provisioner | ❌ | No ZT case |
| Kata containers | 🟡 | ZT runs Kata RuntimeClass spec only if `kata-containers` enabled (expected skip otherwise) |

---

## 3. ZeroTrust-specific coverage NOT in the docx

These ZT plan cases have **no docx counterpart** — the docx is Host-Trusted, so
its security posture is not exercised there. They are the net-new ZT coverage:

| ZT case | Covers |
|---|---|
| OCP-ZT-012 | DPUService credential request create/delete |
| OCP-ZT-013 | DPUService template status/versions from chart annotations |
| OCP-ZT-023 | NodeSRIOVDevicePlugin webhook + managed pods (OCP path) |
| OCP-ZT-030 | **Etcd encryption at rest** (Kamaji DPUCluster data) `{ZeroTrust}` |
| OCP-ZT-031 | **Encrypted secrets** via OpenBao / External Secrets `{ZeroTrust}` |
| OCP-ZT-032 | **BMC factory reset** + password hardening |
| OCP-ZT-033 | **BMC server-certificate (mTLS) rotation** per DPUDevice |

---

## 4. Key gaps (docx behaviour with no ZT automation)

High-value regression areas in the docx that the ZT plan does **not** cover and
would be gaps for a ZeroTrust regression pass:

- **Resiliency/recovery** (TC-RES-*): host/DPU/cluster reboot, power cycle, drain,
  fabric down-up — none driven (ZT runs reuse mode; Redfish/reboot state machine
  out of scope).
- **BFB provisioning incl. SecureBoot** (TC-BFB-*): not driven in reuse mode.
- **DPF release upgrades** (TC-UPG-*) and most **service upgrades** (TC-SVCUPG-*).
- **Critical-services labeling** (TC-CRIT-*) and **service dependencies** (TC-DEP-*).
- **NodeEffect** (TC-NE-*), **stability/soak** (TC-STAB-*), **uninstall**
  (TC-UNINST-*), **network offload** (iperf3/RDMA), **DTS lifecycle**.
