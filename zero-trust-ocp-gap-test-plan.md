---
title: "Test Plan — Zero-Trust on OCP: Coverage Gaps vs Kubernetes"
---

[TOC]

# Test Plan — Zero-Trust on OCP: Coverage Gaps vs Kubernetes

Close the OpenShift (OCP) coverage gap for Zero-Trust (ZT): add, on OCP, the ZT
specs that currently run **only** on vanilla Kubernetes. This plan enumerates
every missing spec, classifies it by portability, and defines the OCP test case
(prerequisites, steps, expected result, adaptation needed) to add coverage.

Related:
* [Zero-Trust E2E Suites — K8s vs OCP](./zero-trust-suites-k8s-vs-ocp.md)
* [Test Plan — Zero-Trust on OCP](./zero-trust-ocp-test-plan.md)

---

## 1. Objective

For every spec selected by the vanilla ZT job (`--label-filter="ZeroTrust"`) that
is **not** selected by the OCP job (`--label-filter="OCP"`), establish equivalent
validation on an OCP cluster running in `DeploymentMode=ZeroTrust` — or record a
justified decision not to.

## 2. How the gap was derived

A spec runs on OCP only if it carries `Domain.OCP` (additive label). The gap is
therefore every spec with `Domain.ZeroTrust` and **without** `Domain.OCP`:

```bash
grep -nE 'Describe\(|Context\(|It\(' system_test.go provisioning_test.go \
  | grep ZeroTrust | grep -v Domain.OCP
```

## 3. Gap inventory

Feasibility classes:
* **A — Portable:** behaviour is platform-neutral; add `Domain.OCP` (plus a skip
  helper only if a sub-assertion can't hold on OCP).
* **B — Portable with adaptation:** needs an OCP-specific branch (platform DNS,
  observability stack, reuse-mode object handling).
* **C — Infeasible / unsafe in reuse mode:** mutates or deletes shared live state;
  needs an isolated environment or stays out of scope.

| ID | Spec (K8s ZT) | Location | K8s labels | Class | Gap reason |
|---|---|---|---|---|---|
| G-01 | Encrypted Secrets (OpenBao + External Secrets) | `system_test.go:503` | `ZeroTrust` | A | No `OCP` label |
| G-02 | DPU Deployment — create + metrics + bad-param deletion | `system_test.go:509` | `ZeroTrust` | B | Reuse mode already has a live DPUDeployment |
| G-03 | DPU Service generic IPAM fields — render v4/v6 IPPools & CIDRPools | `system_test.go:522` | `ZeroTrust` | A | No `OCP` label |
| G-03a | ↳ DPUService workload address allocation (v4/v6) | `system_test.go:529` | `ZeroTrust, RequiresNodes` | A | No `OCP` label |
| G-04 | DPU Service — metrics | `system_test.go:593` | `ZeroTrust` | A | No `OCP`; metric-name part needs `skipMetricNamesInOCPReuse` |
| G-05 | DPU Service — delete & cleanup | `system_test.go:596` | `ZeroTrust` | A | No `OCP` label |
| G-06 | DPU Service — ImagePullSecrets synced & cleaned | `system_test.go:599` | `ZeroTrust, ImagePullSecretsSync` | A | No `OCP` label |
| G-07 | Cluster DNS — Kamaji CoreDNS disabled / host CoreDNS / DPU DNS points at host | `system_test.go:628` | `ZeroTrust` | B | OpenShift DNS model differs (`dns-default`/`openshift-dns`) |
| G-07a | ↳ each DPUCluster not served by host keeps own DNS | `system_test.go:651` | `ZeroTrust, RequiresNodes` | B | Same |
| G-08 | Observability → Logging Infra (OTel DaemonSets, collector config, Log/Metrics/Trace flow to Loki/Tempo) | `system_test.go:669` (Logging block `:701`–`:783`) | `Observability, ZeroTrust` | B | Conditional on the DPF observability stack being deployed on the reused cluster |
| G-09 | BMC Factory Reset + password hardening | `system_test.go:845` | `ZeroTrust, RequiresNodes` | A | No `OCP`; needs reachable BMCs |
| G-10 | BMC Server Certificate Rotation (mTLS) | `system_test.go:857` | `ZeroTrust, RequiresNodes` | A | No `OCP`; needs reachable BMCs |
| G-11 | Operator config — verify system component overrides | `system_test.go:898` | `ZeroTrust` | C | Reads/validates live shared config |
| G-12 | Operator config — change MTUs, verify DPU Clusters updated | `system_test.go:904` | `ZeroTrust` | C | Mutates shared live config |
| G-13 | Operator config — overrides path for system DPUServices | `system_test.go:907` | `ZeroTrust` | C | Mutates shared live config |
| G-14 | Operator config — change flannel podCIDR | `system_test.go:913` | `ZeroTrust` | C | Mutates shared live config |
| G-15 | Operator config — toggle PrivilegedPodEnforcement, VAP lifecycle | `system_test.go:916` | `ZeroTrust` | C | Mutates shared live config |
| G-16 | Etcd Encryption at Rest (Kamaji data encrypted) | `system_test.go:982` | `ZeroTrust` | A | No `OCP` label |
| G-17 | Validate DPFOperatorConfig deletion | `system_test.go:990` | `ZeroTrust` | C | Deletion destroys the reused cluster |
| G-18 | Provisioning — DMA SF created by dpu-agent is consumable | `provisioning_test.go:34` | `DPFSystem, ZeroTrust, RequiresNodes` | A | No `OCP`; read-only post-provision check |

Broader ZT suites also absent on OCP but tracked separately (own domains/jobs,
not part of this gap plan): Weave Physical (`WeavePhysical`), mock-DPU (`MockDPU`),
DPF Upgrade ZT.

## 4. Test cases to add

### 4.1 Class A — portable (add `Domain.OCP`, optional skip helper)

For each: prerequisite is the OCP reuse-mode pre-flight (operator + ZT
DPFOperatorConfig Ready, `doca` DPUCluster Ready, DPUs provisioned).

| TC | From | Steps | Expected on OCP | Adaptation |
|---|---|---|---|---|
| OCP-GAP-01 | G-16 Etcd Encryption | Inspect Kamaji DPUCluster etcd data | Payload encrypted at rest | none |
| OCP-GAP-02 | G-01 Encrypted Secrets | Create ExternalSecret backed by OpenBao; update source | Secret data replicated and refreshed | OpenBao/ESO must be present on the reused cluster |
| OCP-GAP-03 | G-03 generic IPAM render | Apply subnet/network IPAM configs | v4/v6 IPPools & CIDRPools rendered | reuse existing `buildTestDPUServiceIPAM` OCP branch |
| OCP-GAP-04 | G-03a workload alloc | Schedule workload pods requesting v4/v6 | Addresses assigned from pools | `RequiresNodes` |
| OCP-GAP-05 | G-04/G-05 DPU Service metrics + cleanup | Verify metrics; delete DPUServices | Apps cleaned up; metrics present | wrap metric-name asserts in `skipMetricNamesInOCPReuse` |
| OCP-GAP-06 | G-06 ImagePullSecrets | Verify sync + cleanup | Secrets synced to DPU cluster and removed | `ImagePullSecretsSync` |
| OCP-GAP-07 | G-09 BMC Factory Reset | Trigger bootstrap reset; check password hardening | BMC reset to factory; passwords hardened | reachable BMCs; `RequiresNodes` |
| OCP-GAP-08 | G-10 BMC Cert Rotation | Rotate mTLS server cert per DPUDevice | New cert active, old rejected | reachable BMCs; `Serial` |
| OCP-GAP-09 | G-18 DMA SF | Query SF created by dpu-agent; consume it | SF present and usable | read-only; `RequiresNodes` |

### 4.2 Class B — portable with adaptation (needs an OCP branch)

| TC | From | OCP-specific expectation | Adaptation design |
|---|---|---|---|
| OCP-GAP-10 | G-02 DPU Deployment | Create a *second* DPUDeployment (or reuse the live one) without colliding with the pre-existing object | Follow the `ValidateDPUDeploymentFullCreation` pattern (`dpudeployment.go:410`): record/reuse the existing deployment rather than assuming none |
| OCP-GAP-11 | G-07 Cluster DNS | Kamaji CoreDNS addon disabled; DPU cluster DNS resolves via the OpenShift host DNS | Branch on `Domain.OCP` to target `dns-default`/`openshift-dns` (as `coredns.go:243` already does for the resolve-from-pod spec) |
| OCP-GAP-12 | G-08 Observability logging | OTel DaemonSets present; logs→Loki, metrics→host collector, traces→Tempo | Gate on observability stack presence; `Skip()` with a documented reason if the reused cluster/pinned operator does not ship it |

### 4.3 Class C — mutating / destructive (do NOT add to the shared OCP reuse job)

G-11…G-15 (operator-config mutation) and G-17 (config deletion) mutate or delete
the single shared live DPFOperatorConfig that reuse mode depends on. Running them
in the reuse job would corrupt the environment for all other specs.

Options (decision required — see §7):
* **C-opt-1 (recommended):** run these against a **dedicated, disposable OCP
  cluster** in a non-reuse ZT lane (provision fresh, allow mutation/deletion),
  rather than the shared reuse job.
* **C-opt-2:** adapt each to snapshot → mutate → assert → restore, marked
  `Serial`, and accept the residual risk on a shared cluster.
* **C-opt-3:** keep as K8s-only; document the OCP gap as accepted.

## 5. Execution

Class A/B once labelled:

```bash
E2E_TEST_ARGS='-ginkgo.label-filter="OCP" -e2e.config=./config-ocp-physical.yaml' make test-e2e
```

Class C (C-opt-1) on a disposable cluster, ZT non-reuse:

```bash
E2E_TEST_ARGS='-ginkgo.label-filter="ZeroTrust && !OCP && <operator-config specs>" ...' make test-e2e
```

## 6. Acceptance criteria

* Every Class A item carries `Domain.OCP` and passes in the OCP reuse job (metric
  sub-assertions skip with documented reasons where applicable).
* Every Class B item has an OCP branch and either passes or `Skip()`s with a clear
  reason tied to a platform constraint.
* Every Class C item has a recorded decision (C-opt-1/2/3) and, if C-opt-1, runs
  green in the dedicated ZT lane.
* No change regresses the existing vanilla ZT K8s coverage.

## 7. Open decisions

1. **Class C placement** — dedicated disposable OCP ZT cluster (C-opt-1) vs
   snapshot/restore on shared (C-opt-2) vs accept gap (C-opt-3)?
2. **Observability (G-08)** — is the DPF logging/tracing stack (OTel/Loki/Tempo)
   deployed on the target OCP cluster / shipped by the pinned operator? Drives
   whether OCP-GAP-12 runs or skips.
3. **BMC access (G-09/G-10)** — are BMCs reachable and credentialed from the OCP
   lab environment?
4. **OpenBao/ESO (G-01)** — present on the reused OCP cluster?

> Line/label references reflect the tree when this plan was written; treat them as
> pointers, not guarantees.
