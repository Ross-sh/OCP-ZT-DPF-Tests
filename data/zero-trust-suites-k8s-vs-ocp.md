---
title: "Zero-Trust E2E Suites — Kubernetes vs OpenShift (OCP)"
---

[TOC]

# Zero-Trust E2E Suites — Kubernetes vs OpenShift (OCP)

Two lists of the Zero-Trust (ZT) e2e test suites: one for **vanilla Kubernetes**
and one for **OpenShift (OCP)**. There is no separate ZT-for-k8s vs ZT-for-OCP
source tree — it is one set of specs, selected by Ginkgo labels, with the spec
bodies adapting their behaviour when the `OCP` label is present.

## How selection works

ZT coverage is driven by two additive `Domain` labels (defined in
`test/e2e/utils.go`):

* `Domain.ZeroTrust` — the DPFOperatorConfig is deployed in `DeploymentMode=ZeroTrust`
  during `BeforeSuite`. Selected by the vanilla ZT CI job (e.g. `e2e-zt-physical-bf4`)
  with `--label-filter="ZeroTrust"`.
* `Domain.OCP` — run against an **already-provisioned** OpenShift cluster in *reuse
  mode*. Selected by the OCP CI job (e.g. `e2e-ocp-bf3`, config
  `config-ocp-physical.yaml`) with `--label-filter="OCP"`.

Because `OCP` is additive, a spec tagged `{ZeroTrust, OCP}` runs in **both** jobs;
its body branches on `isGinkgoLabelApplied(Domain.OCP)` to adapt.

| | Vanilla Kubernetes (ZT) | OpenShift (OCP) |
|---|---|---|
| Label filter | `ZeroTrust` | `OCP` |
| BeforeSuite | `SystemSetupBeforeSuite` — creates DPFOperatorConfig (`DeploymentMode=ZeroTrust`), BFB, DPUFlavor, DPUSet, DPUCluster, then provisions | `OCPReuseBeforeSuite` — reuses the live cluster; verifies operator + config ready and binds DPUCluster clients only (`e2e_suite_test.go:389`) |
| Operator | built from HEAD | pinned release operator |
| Teardown | deletes the operator config | leaves the existing DPFOperatorConfig in place (`e2e_suite_test.go:547`) |
| Config file | `config-provisioning-zt-physical-bf4*.yaml` | `config-ocp-physical.yaml` |

Legend used below:
* **(both)** — spec carries `{ZeroTrust, OCP}` and runs in both lists.
* ⚠️ — selected in OCP but `Skip()`s at runtime in reuse mode (reason noted).

## List 1 — Vanilla Kubernetes (Zero-Trust) · `--label-filter="ZeroTrust"`

### Dedicated suites (own CI jobs)

| Suite | File | Labels |
|---|---|---|
| DPF System tests - Provisioning (DMA SF consumable) | `provisioning_test.go:34` | `DPFSystem, ZeroTrust` |
| Weave Physical testcases (v4/v6 virtual networks, flow metrics, ib_write_bw) | `weave_physical_test.go:37` | `WeavePhysical, ZeroTrust` |
| Zero-trust provisioning with mock DPUs (BF3/BF4 in kind) | `mock_dpuagent_test.go:44` | `MockDPU` |
| DPF Upgrade ZT (previous-GA → main in ZT mode) | `upgrade_test.go:86` | — |

### Core suite — "DPF System tests - Core" (`system_test.go:481`)

1. Encrypted Secrets — `{ZeroTrust}`
2. DPU Deployment — `{ZeroTrust}`
3. DPU Service generic IPAM fields — `{ZeroTrust}`
4. DPU Service IPAM — **(both)**
5. DPU Service Chain — **(both)**
6. DPU Service Credential Request — **(both)**
7. DPU Service (metrics / cleanup / ImagePullSecrets sync) — `{ZeroTrust}`
8. DPU Service Template — **(both)**
9. Validate General DPF Metrics — **(both)**
10. VAP Deprecation Warnings — **(both)**
11. Cluster DNS — `{ZeroTrust}`
12. Observability — `{Observability, ZeroTrust}`; sub-suites: Metrics Collection **(both)**, Node Problem Detector **(both)**, Logging / Trace / Metrics flow `{ZeroTrust}`
13. DPU Agent — **(both)**
14. BMC Factory Reset — `{ZeroTrust, RequiresNodes}`
15. BMC Server Certificate Rotation — `{ZeroTrust, RequiresNodes}`
16. Validate DPU Operator Config — overrides, MTU, podCIDR, PrivilegedPodEnforcement VAP (`{ZeroTrust}`); the MTU-1500 and MaxDPUParallelInstallations specs are **(both)**
17. Validate DPUDeployment full creation — readiness + disruptive upgrades of standard DPUServices / DPUServiceChain are **(both)**
18. Etcd Encryption at Rest — `{ZeroTrust}`
19. Validate DPFOperatorConfig deletion — `{ZeroTrust}`

## List 2 — OpenShift (OCP, reuse mode) · `--label-filter="OCP"`

### Core suite contexts selected by the `OCP` label (all **(both)**)

1. DPU Service IPAM
2. DPU Service Chain
3. DPU Service Credential Request
4. DPU Service Template
5. Validate General DPF Metrics — kube-state-metrics name & PromQL assertions skipped
6. VAP Deprecation Warnings — ⚠️ skipped (pinned operator ships no deprecation VAPs)
7. Observability → Metrics Collection; Node Problem Detector — ⚠️ skipped (pinned operator reports a different node condition set)
8. DPU Agent — OCP-specific expected condition set (`system_test.go:815`)
9. Validate DPU Operator Config → flannel MTU-1500 and MaxDPUParallelInstallations specs
10. Validate DPUDeployment full creation → becomes-ready + disruptive upgrades of DPUServices / DPUServiceChain (OCP records & reuses the existing DPUDeployment, `dpudeployment.go:410`)

### OCP-only suites (carry `OCP` but **not** `ZeroTrust`)

These are **not** in List 1 — on vanilla k8s they run under the DPFSystem job instead.

11. Cluster DNS → "resolve a Service name from a DPU cluster Pod" — `{RequiresNodes, OCP}`
12. NodeSRIOVDevicePluginController — webhook validation — `{OCP}`
13. NodeSRIOVDevicePluginController Managed Pods — `{RequiresNodes, OCP}`
14. DPU reprovisioning (host-trusted OnDelete reprovision) — `{RequiresNodes, OCP}` (`reprovision.go:43`)

## OCP reuse-mode behavioural differences (reference)

Where shared ZT specs adapt when the `OCP` label is present:

* **Metrics names** — pinned operator emits kube-state-metrics without the `dpf_`
  prefix → name assertions skipped (`skipMetricNamesInOCPReuse`, `utils.go:480`).
* **Prometheus** — OpenShift serves metrics via thanos-querier, unreachable by the
  test client → PromQL assertions skipped (`skipPrometheusInOCPReuse`, `utils.go:490`).
* **Host KSM service** — `dpf-host-kube-state-metrics` on OCP vs `kube-state-metrics`
  (`e2e_suite_test.go:322`, `utils.go:69`).
* **DPU-cluster system pods** — OCP provides its own multus/CNI, so DPF-managed
  `cni-installer`/`kube-multus` are not expected → `ocpSystemPodsToVerify`
  (`utils.go:497`).
* **Cluster DNS** — targets `dns-default` in `openshift-dns` (`coredns.go:243`).
* **DPUServiceIPAM** — OCP drops physical-only node selectors and static allocations
  (`dpuserviceipam.go:327`).
* **NAD** — cross-namespace attach skipped; OpenShift Multus `namespaceIsolation`
  rejects it (`dpuservicenad.go:51`).
* **Kata containers** — OCP checks whether the kata DPUService is enabled and skips
  if not (`dpuservice_kata_container.go:53`).
* **in-cluster DPUService disruptive upgrade** — skipped; `example-in-cluster` is not
  created and nginx cannot run under the `restricted-v2` SCC (`dpudeployment.go:1553`).
* **NodeSRIOVDevicePlugin** — dedicated OCP path
  `validateOCPNodeSRIOVDevicePluginManagement` (`nodesriovdeviceplugin.go:152`).

> Line numbers reflect the state of the tree when this doc was written; treat them
> as pointers, not guarantees.
