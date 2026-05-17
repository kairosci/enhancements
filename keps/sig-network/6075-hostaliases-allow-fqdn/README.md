# KEP-6075: Allow FQDN with trailing dot in HostAliases

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Validation](#validation)
  - [Storage: trailing dot is preserved](#storage-trailing-dot-is-preserved)
  - [Interaction with existing HostAliases semantics](#interaction-with-existing-hostaliases-semantics)
  - [Feature gate](#feature-gate)
  - [Test Plan](#test-plan)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
<!-- /toc -->

## Summary

Allow a trailing dot in hostnames under `spec.hostAliases[].hostnames`. The trailing dot denotes a Fully Qualified Domain Name (FQDN) as defined in RFC 1034, preventing DNS resolvers from appending the search path.

## Motivation

`HostAliases` hostnames are validated with `ValidateDNS1123Subdomain`, which rejects trailing dots. A trailing dot is a standard DNS convention that marks a name as absolute. Applications that resolve absolute names (e.g. `http://example.com.:8080`) depend on this to bypass search path expansion.

Without this change, users must either accept incorrect DNS behavior or use privileged init containers to rewrite `/etc/hosts`.

### Goals

- Allow a trailing dot in `HostAliases` hostnames for FQDN entries.

### Non-Goals

- Change any other DNS validation or behavior.
- Add new API fields or types.

## Proposal

Modify `ValidateHostAliases` to unconditionally strip a trailing dot before calling `ValidateDNS1123Subdomain`. Add a gate check in `ValidatePod` and `ValidatePodUpdate` to reject trailing dots when the feature gate is disabled.

The validation itself always succeeds for valid subdomains regardless of gate state — the gate only controls whether trailing dots are permitted at all.

### User Stories

#### Story 1

A pod runs an application that resolves `example.com.` (with trailing dot, as an FQDN). The user adds a HostAlias to pin that name to a specific IP:

```yaml
spec:
  hostAliases:
  - ip: "10.10.10.10"
    hostnames:
    - "example.com."
```

The trailing dot makes `gethostbyname` skip the search list. Without this KEP the pod is rejected by validation.

### Risks and Mitigations

Downgrade risk: pods with trailing dot hostnames cannot be updated through an older API server that does not strip trailing dots. Mitigated by ratcheting (see Upgrade/Downgrade Strategy).

## Design Details

### Validation

Current code at `pkg/apis/core/validation/validation.go`:

```go
func ValidateHostAliases(hostAliases []core.HostAlias, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}
    for i, hostAlias := range hostAliases {
        allErrs = append(allErrs, IsValidIPForLegacyField(fldPath.Index(i).Child("ip"), hostAlias.IP, nil)...)
        for j, hostname := range hostAlias.Hostnames {
            allErrs = append(allErrs, ValidateDNS1123Subdomain(hostname, fldPath.Index(i).Child("hostnames").Index(j))...)
        }
    }
    return allErrs
}
```

Proposed change: strip a single trailing dot before DNS subdomain validation:

```go
func ValidateHostAliases(hostAliases []core.HostAlias, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}
    for i, hostAlias := range hostAliases {
        allErrs = append(allErrs, IsValidIPForLegacyField(fldPath.Index(i).Child("ip"), hostAlias.IP, nil)...)
        for j, hostname := range hostAlias.Hostnames {
            allErrs = append(allErrs, ValidateDNS1123Subdomain(strings.TrimSuffix(hostname, "."), fldPath.Index(i).Child("hostnames").Index(j))...)
        }
    }
    return allErrs
}
```

`strings.TrimSuffix` removes only a single trailing dot and only when present:
- `"example.com."` → `"example.com"` → valid subdomain
- `"localhost."` → `"localhost"` → valid label
- `"my-server."` → `"my-server"` → valid label (single-label FQDNs are accepted)
- `"example.com.."` → `"example.com."` → rejected by `ValidateDNS1123Subdomain` (label ends with dot)
- `"."` → `""` → rejected by `ValidateDNS1123Subdomain` (empty string)

### Storage: trailing dot is preserved

The trailing dot is stored verbatim in etcd. The API server does not normalize or strip it after validation.

This is required because the kubelet writes hostnames from `HostAliases` directly into `/etc/hosts` via `hostsEntriesFromHostAliases`:

```go
func hostsEntriesFromHostAliases(hostAliases []v1.HostAlias) []byte {
    var buffer bytes.Buffer
    buffer.WriteString("\n")
    buffer.WriteString("# Entries added by HostAliases.\n")
    for _, hostAlias := range hostAliases {
        buffer.WriteString(fmt.Sprintf("%s\t%s\n", hostAlias.IP, strings.Join(hostAlias.Hostnames, "\t")))
    }
    return buffer.Bytes()
}
```

If the trailing dot were stripped, the entry in `/etc/hosts` would lack it and FQDN resolution would not match.

### Interaction with existing HostAliases semantics

The existing behavior of `HostAliases` is unchanged for hostnames without a trailing dot. Reserved names (e.g. `localhost`, pod hostname, node name) are already handled by the existing HostAliases logic — the kubelet writes whatever hostnames are in the field into `/etc/hosts` without filtering. This KEP does not add any special filtering: a trailing dot simply passes through the same code path.

Users cannot override the pod's own hostname or localhost via HostAliases in any meaningful way beyond what `/etc/hosts` already allows. This is unchanged — the trailing dot does not introduce any new ability to break local resolution beyond what already exists with non-FQDN entries.

### Feature gate

The feature gate check is added at the caller level in `ValidatePod` and `ValidatePodUpdate`:

```go
for i, hostAlias := range pod.Spec.HostAliases {
    for j, hostname := range hostAlias.Hostnames {
        if strings.HasSuffix(hostname, ".") && hostname != "." {
            allErrs = append(allErrs, field.Forbidden(field.NewPath("spec", "hostAliases").Index(i).Child("hostnames").Index(j),
                "trailing dot requires feature gate HostAliasesAllowFQDN"))
        }
    }
}
```

**Gate enabled**: trailing dots are allowed on create and update.

**Gate disabled**: trailing dots are rejected on create. Updates are not additionally restricted because `ValidateHostAliases` always accepts the trimmed value — ratcheting is automatic: existing hostnames with trailing dots remain updatable after the gate is disabled.

### Test Plan

##### Unit tests

| Input | Gate state | Expected |
|---|---|---|
| `"example.com."` | enabled | accepted |
| `"example.com."` | disabled | rejected |
| `"example.com"` | either | accepted (unchanged) |
| `"."` | either | rejected (bare dot) |
| `"example.com.."` | either | rejected (multi-dot) |
| `".."` | either | rejected |
| `"localhost."` | enabled | accepted |
| `"my-server."` | enabled | accepted (single-label FQDN) |

##### Integration tests

- create pod with trailing dot — accepted with gate enabled, rejected with gate disabled
- update pod that has trailing dot (created with gate on) after gate is disabled — accepted (ratcheting)

##### e2e tests

- create pod with trailing dot in hostAliases, verify `/etc/hosts` contains the trailing dot entry and FQDN resolution works from within the pod

### Graduation Criteria

#### Alpha

- Feature gate `HostAliasesAllowFQDN`, disabled by default.
- Unit and integration tests.

#### Beta

- Gate enabled by default.
- e2e tests.

#### GA

- Feature gate removed (locked to true).

### Upgrade / Downgrade Strategy

`ValidateHostAliases` performs `strings.TrimSuffix(hostname, ".")` before DNS validation irrespective of the gate. This means a trailing dot never causes a DNS validation error — it only fails the explicit gate check. When the gate is disabled on update, the per-hostname gate check is skipped entirely, so stored objects with trailing dots remain updatable.

Downgrade safety: if a cluster is downgraded to a release without this change, pods with trailing dots cannot be updated. The risk is limited to pods that explicitly use the feature.

### Version Skew Strategy

Validation is confined to the API server. The kubelet writes hostnames verbatim into `/etc/hosts` without validation. No version skew concerns.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate
  - Name: HostAliasesAllowFQDN
  - Components: kube-apiserver

###### Does enabling the feature change any default behavior?

No. Opt-in validation relaxation.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Existing objects with trailing dots remain updatable via automatic ratcheting (the DNS validation always strips the trailing dot).

###### What happens if we reenable the feature if it was previously rolled back?

Relaxed validation becomes available again.

###### Are there any tests for feature enablement/disablement?

Yes — unit and integration tests cover enabling, disabling, and re-enabling.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

No impact on running workloads. Rollout failure limited to unrecognized feature gate name (standard behavior).

###### What specific metrics should inform a rollback?

`apiserver_request_total{code=422, resource=pods, verb=POST}`

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Manual testing planned for alpha.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

```sh
kubectl get pods -A -o json | jq '.items[] |
  select(.spec.hostAliases != null) |
  select(.spec.hostAliases[].hostnames[] | endswith(".")) |
  "\(.metadata.namespace)/\(.metadata.name)"'
```

###### How can someone using this feature know that it is working for their instance?

Pod creation succeeds and `/etc/hosts` contains the trailing dot entry.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

N/A. Validation change, no runtime impact.

### Dependencies

No dependencies.

### Scalability

No new API calls, types, or resource increases. One additional byte per hostname.

### Troubleshooting

N/A. Validation change within the API server.

## Implementation History

- 2026-05-14: Initial KEP draft

## Drawbacks

Increases the valid value surface for `HostAliases` hostnames. Trailing dots are a standard DNS convention and valid in `/etc/hosts` — minimal risk.

## Alternatives

1. **No feature gate**: Simpler but bypasses controlled rollout.
2. **Strip trailing dot silently**: Would prevent users from observing it in `/etc/hosts`, defeating the purpose.
3. **New `fqdn` field in `HostAlias`**: Unnecessary complexity for a simple validation relaxation.
