---
title: "Test Plan — Zero-Trust on OpenShift (OCP)"
---

[TOC]

# Test Plan — Zero-Trust on OpenShift (OCP)

Validate DPF running in **Zero-Trust (ZT) deployment mode** on an **OpenShift
(OCP)** cluster. The e2e automation exercises this through the `OCP` Ginkgo label
in *reuse mode* against a pre-provisioned cluster.

Related docs:
* [Zero-Trust E2E Suites — Kubernetes vs OCP](./zero-trust-suites-k8s-vs-ocp.md)
* [Zero-Trust State Machine — E2E Coverage & Retry Gaps](./zero-trust-e2e-state-machine.md)
* [E2E Testing quick start guide](./README.md)

---

## 1. Objective

Confirm that a DPF deployment configured in `DeploymentMode=ZeroTrust` on OpenShift:

* reaches and stays in a healthy, reconciled state (operator + DPFOperatorConfig
  ready, DPUClusters ready, DPUs provisioned);
* exposes the same DPUService / DPUServiceChain / IPAM / credential / template
  behaviour as the host-trusted path, adapted for OpenShift platform services
  (Multus/CNI, DNS, metrics stack, SCCs);
* satisfies the ZT security posture validated by the suite (etcd encryption at
  rest, encrypted secrets, BMC hardening) where those specs are in scope.

## 2. Scope

### In scope

* Automated e2e specs selected by `--label-filter="OCP"` against a live OCP
  cluster (see §6 for the mapped test cases).
* OCP reuse-mode suite lifecycle (`OCPReuseBeforeSuite`): verify, bind, reuse —
  never create/delete the operator config or DPUCluster.
* OpenShift-specific adaptations of shared ZT specs (metrics stack, DNS, CNI,
  SCC, namespace isolation).

### Out of scope / known limitations

* **ZT provisioning & Redfish reboot state machine is NOT driven here.** Reuse
  mode consumes an already-provisioned cluster, so the BMC/Redfish reboot path,
  `nodeRebootConfigMap`, `PatchDPUNodesForScriptReboot` and
  `PatchDPUDevicesForZeroTrust` are not exercised by the OCP job. The OCP job
  runs with the `OCP` label (not `ZeroTrust`), so the ZT env-var gates in
  `validateFlags()` do not apply. Provisioning/reboot ZT coverage is validated
  separately by the vanilla ZT jobs (`e2e-zt-physical-bf4`, mock-DPU) and the
  state-machine doc above.
* Specs skipped at runtime in OCP reuse mode (pinned operator / platform
  constraints) — see §7.
* Upgrade in ZT mode on OCP (covered by the upgrade suites, not this plan).
* Performance/scale.

## 3. Test environment

| Item | Requirement |
|---|---|
| Platform | OpenShift (OCP) cluster, DPF operator already installed |
| DPF mode | DPFOperatorConfig live and `DeploymentMode=ZeroTrust` |
| Operator | Pinned release operator (reuse mode asserts against its behaviour) |
| DPUCluster | Existing `doca` DPUCluster, Ready; referenced read-only by the suite |
| DPU nodes | `numberOfDPUNodes: 2`, `numberOfDPUsPerNode: 1` (per `config-ocp-physical.yaml`); DPUs provisioned and `HostNetworkReady=True` |
| BMC/Redfish | Reachable per DPUDevice (ZT contract: BMC factory reset + mTLS server-cert rotation specs) |
| Platform services | OpenShift Multus/CNI, `openshift-dns` (`dns-default`), monitoring stack (thanos-querier), SCCs (`restricted-v2`) |
| Kata | `kata-containers` DPUService present only if that feature is enabled |

## 4. Prerequisites (pre-flight checklist)

1. OCP cluster reachable; kubeconfig exported for the test client.
2. `dpf-operator-controller-manager` Deployment Ready in `dpf-operator-system`.
3. DPFOperatorConfig exists, is **ZeroTrust mode**, and reports Ready.
4. `doca` DPUCluster exists and is Ready; name/namespace match
   `../objects/infrastructure/dpucluster-doca.yaml`.
5. DPUs provisioned; DPU-agent reporting status on all DPUs.
6. Static manifests under `../objects/` match the live cluster
   (`operatorconfig-ocp.yaml` mirrors reality — it is **not** applied).
7. BMC credentials/reachability available for the BMC specs (factory reset, cert
   rotation).

## 5. Configuration & execution

Config: `test/e2e/config-ocp-physical.yaml` (reuse mode; `bfb`/`dpuSet`/
`dpuFlavor` entries satisfy shared validation but are not applied).

```bash
# From test/e2e
E2E_TEST_ARGS='-ginkgo.label-filter="OCP" -e2e.config=./config-ocp-physical.yaml' \
  make test-e2e
```

Narrow the run with compound filters, e.g.:
* `"OCP && RequiresNodes"` — only specs that need provisioned DPUs.
* `"OCP && !RequiresNodes"` — control-plane/API specs only.

Lifecycle: `OCPReuseBeforeSuite` ensures the operator + DPFOperatorConfig are
ready and binds DPUCluster clients; teardown leaves the DPFOperatorConfig in
place.

## 6. Test cases

> TC IDs are for this plan; each maps to a Ginkgo spec selected by `OCP`.
> "(both)" = also runs in the vanilla ZT job. Priority: P1 core/security,
> P2 functional, P3 platform/observability.

### 6.1 Suite lifecycle & health (P1)

| TC | Description | Expected result |
|---|---|---|
| OCP-ZT-001 | Operator deployment ready | `dpf-operator-controller-manager` ReadyReplicas == replicas |
| OCP-ZT-002 | DPFOperatorConfig ready in ZT mode | Config reconciled Ready; `DeploymentMode=ZeroTrust` |
| OCP-ZT-003 | DPUCluster(s) ready & client bound | Existing `doca` DPUCluster Ready; clients connect |
| OCP-ZT-004 | DPU-agent status on all DPUs | Agent reports status on every provisioned DPU (OCP condition set) |

### 6.2 DPUService & networking (P2) — all (both)

| TC | Suite | Expected result |
|---|---|---|
| OCP-ZT-010 | DPU Service IPAM | Webhook rejects invalid IPAM; legacy v4 subnet/network reconcile to NVIPAM IPPool/CIDRPool; OCP drops physical-only node selectors / static allocations |
| OCP-ZT-011 | DPU Service Chain | DPUServiceInterface + DPUServiceChain mirrored to each cluster; Sets cleaned up on delete |
| OCP-ZT-012 | DPU Service Credential Request | Credentials created and deleted with the request |
| OCP-ZT-013 | DPU Service Template | Status/versions reflect chart annotations (templates labelled `CleanupScope.It`) |
| OCP-ZT-014 | DPUDeployment full creation | Existing DPUDeployment reused; new deployment becomes Ready; disruptive upgrade of standard DPUServices & DPUServiceChain succeeds |

### 6.3 Platform-adapted services (P3)

| TC | Suite | OCP adaptation |
|---|---|---|
| OCP-ZT-020 | Cluster DNS — resolve a Service from a DPU pod | Targets `dns-default` in `openshift-dns` |
| OCP-ZT-021 | Validate General DPF Metrics | Host KSM = `dpf-host-kube-state-metrics`; `dpf_`-prefix name & PromQL assertions skipped |
| OCP-ZT-022 | Observability → Metrics Collection | KSM reachable on host + DPU cluster |
| OCP-ZT-023 | NodeSRIOVDevicePlugin webhook + managed pods | OCP path `validateOCPNodeSRIOVDevicePluginManagement` |

### 6.4 ZT security posture (P1)

| TC | Suite | Expected result |
|---|---|---|
| OCP-ZT-030 | Etcd Encryption at Rest | Kamaji DPUCluster data encrypted in etcd |
| OCP-ZT-031 | Encrypted Secrets | Secret data replicated/refreshed from OpenBao via External Secrets |
| OCP-ZT-032 | BMC Factory Reset | BMC reset to factory on bootstrap; BMC passwords hardened |
| OCP-ZT-033 | BMC Server Certificate Rotation | BMC mTLS server cert rotated per DPUDevice |

> OCP-ZT-030/031 carry `{ZeroTrust}` only — include them when running the ZT
> security specs on OCP; they are not selected by a bare `OCP` filter. Run
> `"OCP || (ZeroTrust && ...)"` or a dedicated pass to cover them.

### 6.5 Operator config & reprovision (P2)

| TC | Suite | Expected result |
|---|---|---|
| OCP-ZT-040 | Operator config — flannel MTU 1500 | DPU-cluster flannel ConfigMap MTU == 1500 |
| OCP-ZT-041 | Operator config — MaxDPUParallelInstallations | Provisioning controller restarts on change |
| OCP-ZT-042 | DPU reprovisioning (host-trusted OnDelete) | DPUSet reprovisions deleted DPU CRs |

## 7. Expected skips (not failures)

The following are **selected** by the OCP filter but `Skip()` by design — verify
they skip with the stated reason rather than fail:

| Spec | Skip reason |
|---|---|
| VAP Deprecation Warnings | Pinned release operator ships no deprecation VAPs |
| Node Problem Detector | Pinned operator reports a different node condition set |
| KSM metric-name assertions | Pinned operator omits the `dpf_` prefix |
| Prometheus/PromQL assertions | OpenShift serves metrics via thanos-querier, unreachable by the test client |
| NAD cross-namespace attach | OpenShift Multus `namespaceIsolation` rejects it |
| in-cluster DPUService disruptive upgrade | `example-in-cluster` not created; nginx cannot run under `restricted-v2` SCC |
| Kata RuntimeClass | Skipped if `kata-containers` is not enabled on the cluster |

## 8. Pass / fail criteria

* **Pass:** all selected, non-skipped specs green; expected skips (§7) skip with
  their documented reasons; cluster left healthy (operator + config Ready,
  DPUCluster Ready) after teardown.
* **Fail:** any selected spec fails; an expected-skip spec runs and fails; the
  suite cannot reach the pre-flight checklist (§4); teardown leaves the cluster
  degraded.

## 9. Risks & notes

* Reuse mode asserts against a **pinned release operator**; mismatches between the
  live operator and the pinned expectations surface as skips or failures — keep
  `../objects/infrastructure/operatorconfig-ocp.yaml` in sync with reality.
* BMC-dependent specs (§6.4) need reachable, authenticated BMCs; flaky Redfish
  endpoints are the most likely source of intermittent failures.
* Because provisioning/reboot is out of scope here, a green OCP run does **not**
  by itself validate the ZT Redfish state machine — pair it with the vanilla ZT
  provisioning/mock-DPU jobs for full ZT coverage.

> Line/label references reflect the tree when this plan was written; treat them as
> pointers, not guarantees.
