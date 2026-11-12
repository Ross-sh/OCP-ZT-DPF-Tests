---
title: "Zero-Trust on OCP — Gap Tests Grouped by Test Scenario"
---

# Zero-Trust on OCP — Gap Tests Grouped by Test Scenario

The same gap tests from [zero-trust-ocp-gap-tests.md](./zero-trust-ocp-gap-tests.md),
grouped by test scenario — the kind of behavior each test exercises.

---

## S-01 — Resource Creation & Reconciliation

| test_name | what is going to be tested |
|---|---|
| dpu_deployment_create | DPUDeployment and its dependent objects are created and reconciled |

## S-02 — IPAM Rendering & Address Allocation

| test_name | what is going to be tested |
|---|---|
| ipam_render_ippools | IPv4/IPv6 IPPools render from subnet configurations |
| ipam_render_cidrpools | IPv4/IPv6 CIDRPools render from network configurations |
| ipam_workload_alloc_v6 | A workload is assigned an IPv6 address from the pool |
| ipam_workload_alloc_v4v6 | A workload is assigned IPv4 and IPv6 addresses from the pools |

## S-03 — Metrics Exposure

| test_name | what is going to be tested |
|---|---|
| dpu_deployment_metrics | DPUDeployment and DPUServiceInterface metrics are exposed |
| dpuservice_metrics | DPUService and DPUServiceInterface metrics are exposed |

## S-04 — Deletion & Cleanup

| test_name | what is going to be tested |
|---|---|
| dpu_deployment_bad_param_delete | Deletion on a disruptive upgrade with bad parameters (up-to-date DPUService never becomes ready) |
| dpuservice_delete_cleanup | Deleting DPUServices cleans up the deployed applications |
| operatorconfig_deletion | DPFOperatorConfig deletion removes the expected objects |

## S-05 — Secret Replication & Sync

| test_name | what is going to be tested |
|---|---|
| encrypted_secrets | OpenBao secret data replicates and refreshes into the cluster via External Secrets |
| imagepullsecrets_sync | ImagePullSecrets are synced to the DPU cluster and cleaned up |

## S-06 — Security Hardening & Encryption at Rest

| test_name | what is going to be tested |
|---|---|
| etcd_encryption_at_rest | Kamaji DPUCluster data is encrypted in etcd |
| bmc_factory_reset | BMC is reset to factory defaults on bootstrap and passwords are hardened |
| bmc_cert_rotation | BMC mTLS server certificate is rotated for each DPUDevice |

## S-07 — DNS & Networking Resolution

| test_name | what is going to be tested |
|---|---|
| cluster_dns_coredns_disabled | Kamaji CoreDNS addon is disabled when host serves DNS |
| cluster_dns_host_coredns | Host-cluster CoreDNS (OpenShift dns-default) is running |
| cluster_dns_dpu_points_to_host | DPU cluster DNS Service points at the host cluster CoreDNS |
| cluster_dns_own_dns | Each DPUCluster not served by the host keeps its own DNS |

## S-08 — Observability Pipelines (Collectors, Logs, Metrics, Traces)

| test_name | what is going to be tested |
|---|---|
| otel_daemonsets_host | OpenTelemetry Collector DaemonSets run in the host cluster |
| otel_daemonsets_dpu | OpenTelemetry Collector DaemonSets run in the DPU cluster |
| otel_dpu_collector_config | DPU cluster collector configuration is correct |
| otel_user_collector | An additional collector from a user DPUService runs |
| log_flow_mgmt_to_loki | Management-cluster logs are collected and forwarded to Loki |
| log_flow_dpu_to_loki | DPU-cluster logs are collected and forwarded to Loki |
| log_flow_kamaji_audit | Kamaji audit logs are forwarded to Loki |
| metrics_flow_dpu_to_host | DPU-cluster metrics are forwarded to the host collector |
| trace_flow_controller_to_tempo | Controller reconcile spans are exported to Tempo |
| trace_flow_sfc_to_tempo | sfc-controller spans from the DPU cluster reach Tempo |
| trace_flow_dpu_single_trace | A provisioned DPU is followed from its DPUDevice in one trace |

## S-09 — Operator Configuration Changes

| test_name | what is going to be tested |
|---|---|
| opcfg_system_overrides | System component overrides are applied |
| opcfg_mtu_update | Changing MTUs in operatorConfig updates the DPU Clusters |
| opcfg_overrides_path | Overrides path setting for system DPUServices is honored |
| opcfg_flannel_podcidr | Changing the flannel podCIDR in operatorConfig takes effect |
| opcfg_privileged_pod_enforcement | Toggling PrivilegedPodEnforcement drives the VAP lifecycle in the DPU cluster |

## S-10 — Provisioning Readiness

| test_name | what is going to be tested |
|---|---|
| dma_sf_consumable | The DMA SF created by the dpu-agent is consumable |
