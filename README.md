# ACM Policies

## Introduction

This repository contains an exercise that is using Red Hat ACM policies framework to enforce the state of configuration and inform about the status of a subset of CIS Benchmark controls.
ArgoCD is loaded with the policy generator plugin that deploys some of the policies using a GitOps aproach.

### GitHub Repository Organization
The Git repository used for this demo has the following structure:

- **argocd:** ArgoCD ApplicationSet definitions.
- **hub:** Hub cluster configuration only.
- **operators:** One subfolder per operator to install on managed clusters Each follows the same structure as `policies/` (kustomization + PolicyGenerator + placement + manifests). Operators are Deployed via the `operators-appset.yaml` ApplicationSet.
- **policies:** One subfolder per audit/compliance policy domain. 
- **auxiliar:** contains manifests, for example policies, that will be used in the test exercices.

---

## Deployment Guide

### Overview

The deployment has three phases:

1. **Hub cluster groups:** Manually apply ACM namespace, RBAC, ManagedClusterSets, and ManagedClusterSetBindings to the hub cluster.
2. **GitOps operator install in:** Apply an ACM `OperatorPolicy` to the hub. ACM installs the GitOps operator, configures ArgoCD with the PolicyGenerator plugin and ApplicationSet controller, and registers ArgoCD with ACM.
3. **Policy and operator deployment:** Deploy ACM policies and operator installs to managed clusters via PolicyGenerator and ArgoCD (two ApplicationSets).

### Prerequisites

Before starting, ensure the following:

- **Red Hat ACM** is installed on the hub cluster and the `MultiCluster Engine` is installed and running.
- **At least one managed (spoke) cluster** with the name `cluster1`. Verify with: `oc get managedclusters`.
- **oc CLI** is available and authenticated to the hub cluster with `cluster-admin` privileges.
- **Git repository** is accessible from the hub cluster (the ArgoCD repo-server must be able to clone it).

- Clone the repository

  You should clone the repository for a git account that you own, so that you are allowed to push to the repository changes.

  ```bash
  git clone https://github.com/luisevm/acm-policies-01.git
  cd acm-policies-01
  ```

- Login to the ACM HUB

  ```bash
  oc login -u admin -p <password>  https://<api>:6443
  ```

- Apply Hub Cluster Groups

  Apply all cluster group resources to the hub. These create the `acm-policies` namespace, RBAC for the ArgoCD controller, ManagedClusterSets, and ManagedClusterSetBindings. The `acm-policies` namespace must exist before the OperatorPolicy can be applied in the next step.

  ```bash
  oc apply -f hub/clustergroups/
  ```

- Install GitOps Operator to the ACM HUB

  Generate and apply the GitOps policy to the hub. ACM will install the OpenShift GitOps operator, and configure the ArgoCD instance with the PolicyGenerator plugin.

  **Note:** You must configure the policy generator plugin in your laptop before running this command.

  ```bash
  kustomize build hub/gitops --enable-alpha-plugins | oc apply -f -
  ```

  Monitor the policy compliance:

  ```bash
  oc get policy gitops-operator -n acm-policies -w
  ```

  > **Note:** The ConfigurationPolicy for the ArgoCD patch will initially be non-compliant while the operator is installing (the ArgoCD instance does not exist yet). ACM retries automatically — once the operator creates the ArgoCD instance, the patch is applied. This is expected behavior.

- Deploy ACM Policies

  The policies are created using GitOps.
  Apply both ApplicationSets. Each auto-discovers folders under its respective directory and creates an ArgoCD Application per domain.

  ```bash
  oc apply -f argocd/operators-appset.yaml
  oc apply -f argocd/policies-appset.yaml
  ```

  Verify the generated ApplicationSets and Applications:

  ```bash
  oc get applicationset.argoproj.io -n openshift-gitops
  oc get app.argoproj.io -n openshift-gitops
  ```

- Verify Policies on Managed Clusters

  Check that the policies are distributed and evaluated:

  ```bash
  # List all policies in the acm-policies namespace
  oc get policies -n acm-policies

  # Check compliance status
  oc get policies -n acm-policies -o custom-columns=NAME:.metadata.name,COMPLIANCE:.status.compliant
  ```

---

## Hub Configuration Details

### Cluster Groups (`hub/clustergroups/`)

These resources are applied manually with `oc apply -f hub/clustergroups/` and establish the foundational ACM configuration on the hub cluster. Files are prefixed with numbers to indicate logical ordering (namespace first, then RBAC, then cluster sets, then bindings), though `oc apply` processes them declaratively.

#### Labelling Managed Clusters

After importing managed clusters into ACM, you can assign them to cluster sets and add environment labels. This enables policy placements to target specific groups of clusters — for example, applying stricter policies to production or different operator configurations to development.

**Command structure:**

```bash
oc label managedcluster <cluster-name> <label-key>=<label-value> --overwrite
```

---

## Created Policies

Once tests from a following chapter are worked, the following is the list of  policies created.

### Governance Metadata

All policies carry three ACM metadata fields used for filtering and grouping in the Governance dashboard:


| Domain     | Policy                               | Remediation | Severity | Target                   | Source                           | What it does                                                                                                          |
| ---------- | ------------------------------------ | ----------- | -------- | ------------------------ | -------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| RBAC       | `cm-rbac-exceptions-exists`          | enforce     | low      | All OpenShift clusters   | `policies/rbac/`                 | Deploys the `rbac-policy-exceptions` ConfigMap to `acm-policies` on all managed clusters                              |
| RBAC       | `cis-cluster-admin`                  | inform      | critical | All OpenShift clusters   | `policies/rbac/`                 | Flags ClusterRoleBindings to `cluster-admin` where any subject is a non-platform ServiceAccount, User, or Group       |
| RBAC       | `sa-token-restriction`               | inform      | medium   | All OpenShift clusters   | `policies/rbac/`                 | Flags default ServiceAccounts in user namespaces that lack `automountServiceAccountToken: false`                      |
| RBAC       | `detect-anonymous-and-wildcard-rbac` | inform      | high     | All OpenShift clusters   | `policies/rbac/`                 | Flags CRBs granting access to unauthenticated/anonymous users and wildcard Roles in user namespaces (PCI DSS 7.2/7.3) |
| Registries | `allowed-registries`                 | enforce     | high     | `registry-enforce=true`  | `auxiliar/registries/`           | Patches `image.config.openshift.io/cluster` to add approved registry configuration                                    |
| Kubeadmin  | `kubeadmin-remove-enforce`           | enforce     | critical | `kubeadmin-enforce=true` | `policies/kubeadmin/`            | Deletes the `kubeadmin` secret on opted-in clusters                                                                   |
| Gatekeeper | `gatekeeper-operator-install`        | enforce     | high     | All OpenShift clusters   | `auxiliar/gatekeeper/`           | Installs the Red Hat Gatekeeper Operator and creates the Gatekeeper CR                                                |
| Gatekeeper | `gk-rbac-clusteradmin`               | inform      | high     | All OpenShift clusters   | `auxiliar/gatekeeper/`           | Flags CRBs/RBs referencing the `cluster-admin` ClusterRole                                                            |
| Gatekeeper | `gk-rbac-wildcard-roles`             | inform      | high     | All OpenShift clusters   | `auxiliar/gatekeeper/`           | Flags bindings whose referenced ClusterRole/Role has wildcard permissions                                             |
| GitOps     | `gitops-operator`                    | enforce     | critical | `local-cluster` (hub)    | `hub/gitops/`                    | Installs OpenShift GitOps and configures ArgoCD with PolicyGenerator plugin                                           |
| Compliance | `compliance-operator`                | enforce     | medium   | All OpenShift clusters   | `operators/compliance-operator/` | Installs the Compliance Operator (namespace, OperatorPolicy, OperatorGroup)                                           |
| Compliance | `compliance-cis-scan`                | enforce     | high     | All OpenShift clusters   | `operators/compliance-operator/` | Deploys CIS ScanSetting + ScanSettingBinding                                                                          |
| Compliance | `compliance-cis-results`             | inform      | high     | All OpenShift clusters   | `operators/compliance-operator/` | Reports failed CIS ComplianceCheckResults — non-compliant when checks fail                                            |


---

### Handling False Positives (Exception Workflow)

When a policy flags a resource that is intentionally configured (e.g., a trusted user who should have `cluster-admin`), you can suppress the violation. RBAC policies use two different exception mechanisms depending on the policy:

**Method 1 — Inline trusted list** (used by `detect-anonymous-and-wildcard-rbac`)

The `rbac-no-unauth-access` and `rbac-no-wildcard-roles` ConfigurationPolicies each contain an inline trusted-entity list directly in `policies/rbac/manifests/cis-rbac-controls.yaml`. Add entries to suppress false positives:

- `$allowedCRBs` — CRB names to skip in `rbac-no-unauth-access` (e.g. `"my-trusted-crb"`)
- `$allowedRoles` — `"namespace/rolename"` entries to skip in `rbac-no-wildcard-roles`

Edit the file, commit and push. ArgoCD syncs the change and the violation clears on the next evaluation cycle.

**Method 2 — Exceptions ConfigMap** (used by `cis-cluster-admin`)

The `cis-cluster-admin` policy reads exceptions from a ConfigMap (`rbac-policy-exceptions` in `acm-policies` namespace) deployed to all managed clusters by the `cm-rbac-exceptions-exists` enforce policy. Each key holds a newline-separated list of resource names to skip.

Edit `policies/rbac/manifests/cm-rbac-exceptions.yaml` and add the resource name:

```yaml
data:
  cis-cluster-admin: |
    cluster-admin-0
    my-trusted-admin-binding
```

Commit and push. ArgoCD syncs the ConfigMap to all clusters and the violation disappears on the next evaluation cycle.

| Method | Policy | File to edit | What to add |
| --- | --- | --- | --- |
| Inline `$allowedCRBs` | `rbac-no-unauth-access` | `policies/rbac/manifests/cis-rbac-controls.yaml` | CRB name |
| Inline `$allowedRoles` | `rbac-no-wildcard-roles` | `policies/rbac/manifests/cis-rbac-controls.yaml` | `namespace/rolename` |
| ConfigMap key `cis-cluster-admin` | `cis-cluster-admin` | `policies/rbac/manifests/cm-rbac-exceptions.yaml` | CRB name |

> **Note:** Policy `sa-token-restriction` does not use either mechanism. To exclude namespaces from this policy, add them to `namespaceSelector.exclude` in the policy manifest directly.

---

## Testing

### PrePreparation

- Clone the repository
  ```bash
  git clone https://github.com/luisevm/acm-policies-01.git
  cd acm-policies-01
  ```
- Login to the ACM HUB
  ```bash
  oc login -u admin -p <password> <api_fqdn:6443>
  ```
- Extract secret of one managed cluster
  ```bash
  CLUSTER_NAME=cluster1
  SECRET_NAME=$(oc get secret -n $CLUSTER_NAME -o name | grep 'admin-kubeconfig')
  oc extract $SECRET_NAME -n $CLUSTER_NAME --keys=kubeconfig --to=- > /tmp/${CLUSTER_NAME}-kubeconfig
  ```
- Review the controls results for the Compliance Operator and the controls that must be manually checked.
  Go to ACM -> Governance, select the `CIS OpenShift Container Platform 4 Benchmark` start and you will find the Policies related with this standart.
  - The policy `compliance-cis-results` contains the results of the Compliance Operator scan against the `ocp4-cis`.
  - The rest of the policies are related with manual controls, that the Compliance Operator doenst classify, as these have to be manually checked.

---

### Test 1: Fix Allowed Registries

**Objectives:**

- Verify that the violation `ocp4-cis-ocp-allowed-registries-for-import` is flagged by the `compliance-cis-results` policy.
- Use the ACM configurationmanager policy to fix this violation flaged by the Compliance operator scan.

**Test Procedure**

1. Go to ACM -> Governance, select the `CIS OpenShift Container Platform 4 Benchmark` standart and select the policy `compliance-cis-results`. Under this Policy you will find the template `cis-results`, that shows the violations raised against each cluster.
  - Select `cis-results`
  - on the Cluster, template `cis-results`, press on the `View Details` textfield, this will show the Violations raised by `cis-ocp4`.
  - one of such violations is `ocp4-cis-ocp-allowed-registries`, press on top of this violation and a new window will open with the recommendation to fix it.
  - Lets fix this Violation, by allowing the applicationSet to create the Policy, the Placement is configured with a label of `vendor=OpenShift`, so this configuration is enforced in all the clusters of the fleat, including the hub cluster.

2. Create a Policy and reThis policy will be applied to all clusters.quired files under policies path
  - Copy the policy to the operators path, this will result that the applicationSet will create the Policy on the Hub

    ```bash
    cp -r auxiliar/registries policies/
    ```

3. Push changes to GitHub
    ```bash
    git add policies/registries/*
    git commit -m "added allowed registries"
    git push
    ```
4. A new policy will be added to ACM HUB. This policy will be applied to all clusters.

    ```bash
    oc -n acm-policies get policy allowed-registries
    ```

---

### Test 2: Fix Kubeadmin

> **WARNING — This deletion is permanent.** The kubeadmin secret cannot be restored without reinstalling the cluster. Before labelling a cluster, verify that an identity provider is configured and at least one user can log in with `cluster-admin` privileges: `oc login -u <user>`, or you have a valid `kubeconfig` file.

**Objectives:**

- Use the ACM configuration manager policy to fix the violation raised by the Compliance operator about the presence of a kubeadmin secret. 
- This policy will aloso demonstrate how the labels are used to control the placement of the policies. In this case the policy is enforced on the clusters labeled with `kubeadmin-enforce=true`

**Test Procedure**

1. Go to ACM -> Governance, select the `CIS OpenShift Container Platform 4 Benchmark` standart and select the policy `compliance-cis-results`. Under this Policy you will find the template `cis-results`, that shows the violations for each cluster.
  - Select `cis-results`
  - on the Cluster, template `cis-results`, press on the `View Details` textfield, this will show the Violations raised by `cis-ocp4`.
  - one of such violations is `ocp4-cis-kubeadmin-removed`, press on top of this violation and a new window will open with the recommendation to fix it.
  - Confirm the secret exists on a target cluster before enforcing, for example in cluster1:
    ```bash
    # Set your target cluster name
    CLUSTER_NAME="cluster1"

    # 1. Dynamically find the secret and extract the kubeconfig
    SECRET_NAME=$(oc get secret -n $CLUSTER_NAME -o name | grep 'admin-kubeconfig')
    oc extract $SECRET_NAME -n $CLUSTER_NAME --keys=kubeconfig --to=- > /tmp/${CLUSTER_NAME}-kubeconfig

    # 2. Execute your command against the managed cluster
    oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get secret kubeadmin -n kube-system
    ```

2. Create a Policy and required files under policies path

  - Copy the policy to the operators path, once changes are pushed to Git, this will result that the applicationSet will create the Policy on the Hub
    ```bash
    cp -r auxiliar/kubeadmin policies/
    ```

3. Push changes to GitHub
  ```bash
  git add policies/kubeadmin/*
  git commit -m "policy kubeadmin"
  git push
  ```
4. A new policy will be added to ACM HUB

  ```bash
  oc -n acm-policies get policy kubeadmin-remove-enforce
  ```

5. Fix this violation on the selected clusters by labelling the clusters where this policy should be placed.

  To decide the clusters where this policy will be placed one must set a label into the clusters where it should be enforced.

  - Label a the clusters to enforce the kubeadmin removal:
    ```bash
    oc label managedcluster cluster1 kubeadmin-enforce=true --overwrite
    ```
  - Confirm both policies now show Compliant for that cluster:
    Wait for ArgoCD to sync and the policy to propagate, then verify the secret was deleted.

    ```bash
    oc get policy kubeadmin-remove-enforce -n acm-policies -o jsonpath='{range .status.status[*]}{.clustername}: {.compliant}{"\n"}{end}'
    ```

---

### Test 3: Run the Compliance Operator OCP CIS Benchmark again and check that the fixed violations are cleared

**Objectives:**
- Run the Compliance operator scan, and verify that the previous checks were fixed. The scan may take several minutes.

1. Scan all clusters

  ```bash
  oc annotate compliancescan/ocp4-cis -n openshift-compliance --all compliance.openshift.io/rescan=

  oc annotate compliancescans/ocp4-cis compliance.openshift.io/rescan= -n openshift-compliance
  oc --kubeconfig=/tmp/cluster1-kubeconfig annotate compliancescans/ocp4-cis compliance.openshift.io/rescan= -n openshift-compliance
  ```

2. check the state of the new scan

  ```bash
  oc get ComplianceSuite -n openshift-compliance
  oc --kubeconfig=/tmp/cluster1-kubeconfig get ComplianceSuite -n openshift-compliance
  ```

3. Once the scan is completed check the state of the controls

  - The `cluster1` is not violating the `ocp4-cis-kubeadmin-removed` nor the allowed `ocp4-cis-ocp-allowed-registries`. 
  - And the local-cluster will ot show the `ocp4-cis-ocp-allowed-registries` violation

  ```bash
  oc get compliancecheckresult ocp4-cis-ocp-allowed-registries -n openshift-compliance -o jsonpath='{.status}'

  oc --kubeconfig=/tmp/cluster1-kubeconfig get compliancecheckresult ocp4-cis-ocp-allowed-registries -n openshift-compliance -o jsonpath='{.status}'
  ```

---

### Test 4: Fix `rbac-no-unauth-access` unauthenticated CRB Detection

**Objectives:**

- Verify that a ClusterRoleBinding granting access to `system:unauthenticated` is flagged by the `rbac-no-unauth-access` ConfigurationPolicy (inside the `detect-anonymous-and-wildcard-rbac` parent policy).
- Verify how to whitelist CRB and CR, adding inline in the policy the trusted ones. This whitelist contains a list of trusted CBR and CR - `$allowedCRBs` and `$allowedRoles`

**Test Procedure**

1. Open the **ACM Console → Governance** dashboard. Select the `CIS OpenShift Container Platform 4 Benchmark` standart. Locate the policy `detect-anonymous-and-wildcard-rbac` and review the current violations for the `rbac-no-unauth-access` template. Note the existing violations (if any).
2. Create on the ACM Hub (local-cluster), a test ClusterRoleBinding that grants the `view` ClusterRole to `system:unauthenticated`:
  ```bash
   oc create clusterrolebinding test-unauth-access --clusterrole=view --group=system:unauthenticated
  ```
3. Wait for the next policy evaluation cycle (1–2 minutes) or check the policy status with:
  ```bash
   oc get policy detect-anonymous-and-wildcard-rbac -n acm-policies \
     -o jsonpath='{range .status.status[*]}{.clustername}: {.compliant}{"\n"}{end}'
  ```
   The policy should show **NonCompliant** and the violation detail in the ACM Governance UI should list `test-unauth-access` for the ACM HUB cluster.
4. Add a new the CRB name to the trusted list. Edit `policies/rbac/manifests/cis-rbac-controls.yaml` and add `"test-unauth-access"` to the `$allowedCRBs` list:
  ```yaml
   {{- $allowedCRBs := list
         "test-unauth-access"
   }}
  ```
5. Commit and push:
  ```bash
   git add policies/rbac/manifests/cis-rbac-controls.yaml
   git commit -m "test: add test-unauth-access to trusted CRBs"
   git push
  ```
6. Wait for ArgoCD to sync and the next policy evaluation cycle (1–2 minutes). Verify the violation for `test-unauth-access` is cleared in the ACM Governance UI.
7. **Clean up** — remove the test CRB and revert the trusted list:
  ```bash
   oc delete clusterrolebinding test-unauth-access
  ```
   Then remove `"test-unauth-access"` from `$allowedCRBs` in `cis-rbac-controls.yaml`, commit and push.

---

### Test 5: Fix Unauthorized CRB to cluster-admin CR

**Objectives:**

- test that a ClusterRoleBinding attaching a ServiceAccount to a cluster-admin of ClusterRole, will cause ACM to raise a violation in the Governance tab.  
- Adding the CRB to the list of trusted CRB will clear the violation.

**Test Procedure**

- Create a ServiceAccount in a user namespace and bind it to `cluster-admin`:
  ```bash
  oc new-project test-cis
  oc create sa test-sa -n test-cis
  oc adm policy add-cluster-role-to-user cluster-admin -z test-sa -n test-cis
  ```
  The result is that a new ClusterRoleBinding is created (binding the SA to the ClusterRole), referencing `cluster-admin` with a ServiceAccount in a non-platform namespace. Policy `cis-cluster-admin` will flag it as **Non-Compliant**.
- Configure the CRB to be trusted, by adding the CRB to the configmap cm-rbac-exceptions.yaml under "cis-cluster-admin: |" 
  ```bash
  vi /mnt/vm/Var/my_git-clone/acm-policies-01/policies/rbac/manifests/cm-rbac-exceptions.yaml

  `<crb-name>`
  ```
- Push changes to GitHub
  ```bash
  git add /mnt/vm/Var/my_git-clone/acm-policies-01/policies/rbac/manifests/cm-rbac-exceptions.yaml
  git commit -m "rbac cluster-admin text"
  git push
  ```
- Result is the violation is cleared.






### Test 6: Trigger `rbac-no-wildcard-roles` — Wildcard Role Detection

**Objectives:**

- Verify that a Role with wildcard verbs, resources, and apiGroups in a user namespace is flagged by the `rbac-no-wildcard-roles` ConfigurationPolicy (inside the `detect-anonymous-and-wildcard-rbac` parent policy).
- Verify that adding the Role to the trusted list (`$allowedRoles`) clears the violation.

**Test Procedure**

1. Open the **ACM Console → Governance** dashboard. Locate the policy `detect-anonymous-and-wildcard-rbac` and review the current violations for the `rbac-no-wildcard-roles` template. Note the existing violations (if any).
2. Create a test namespace and a Role with full wildcard permissions:
  ```bash
   oc new-project test-wildcard-role
   oc create role test-wildcard --verb='*' --resource='*.*' -n test-wildcard-role
  ```
   Verify the Role was created with the expected wildcard rules:
   Confirm the `rules` section contains `apiGroups: ["*"]`, `resources: ["*"]`, `verbs: ["*"]`.
3. Wait for the next policy evaluation cycle (1–2 minutes) or check the policy status with:
  ```bash
   oc get policy detect-anonymous-and-wildcard-rbac -n acm-policies \
     -o jsonpath='{range .status.status[*]}{.clustername}: {.compliant}{"\n"}{end}'
  ```
   The policy should show **NonCompliant** and the violation detail in the ACM Governance UI should list `test-wildcard` in namespace `test-wildcard-role`.
4. Add the Role to the trusted list. Edit `policies/rbac/manifests/cis-rbac-controls.yaml` and add `"test-wildcard-role/test-wildcard"` to the `$allowedRoles` list:
  ```yaml
   {{- $allowedRoles := list
         "test-wildcard-role/test-wildcard"
   }}
  ```
5. Commit and push:
  ```bash
   git add policies/rbac/manifests/cis-rbac-controls.yaml
   git commit -m "test: add test-wildcard-role/test-wildcard to trusted roles"
   git push
  ```
6. Wait for ArgoCD to sync and the next policy evaluation cycle (1–2 minutes). Verify the violation for `test-wildcard` is cleared in the ACM Governance UI.
7. **Clean up** — remove the test namespace and revert the trusted list:
  ```bash
   oc delete project test-wildcard-role
  ```
   Then remove `"test-wildcard-role/test-wildcard"` from `$allowedRoles` in `cis-rbac-controls.yaml`, commit and push.

### Test 3: Gatekeeper RBAC Constraint — Admission and Audit

- Verify that the Gatekeeper operator is installed and running on a managed cluster:
  ```bash
  oc get pods -n openshift-gatekeeper-system --context <cluster-name>
  ```
  You should see `audit-controller` and `controller-manager` pods in `Running` state.
- Check that both ConstraintTemplates and Constraints are deployed:
  ```bash
  oc get constrainttemplate k8sclusteradminbinding k8swildcardclusterrole --context <cluster-name>
  oc get k8sclusteradminbinding gk-clusteradmin-binding --context <cluster-name>
  oc get k8swildcardclusterrole gk-wildcard-clusterrole --context <cluster-name>
  ```
- View the Gatekeeper audit violations for each constraint:
  ```bash
  oc get k8sclusteradminbinding gk-clusteradmin-binding --context <cluster-name> -o json | jq '.status.violations'
  oc get k8swildcardclusterrole gk-wildcard-clusterrole --context <cluster-name> -o json | jq '.status.violations'
  ```
- Test 5a — cluster-admin detection. Create a CRB that binds a user to `cluster-admin`:
  ```bash
  oc create clusterrolebinding test-gk-admin --clusterrole=cluster-admin --user=test-gk-user
  ```
  Gatekeeper returns a warning (since `remediationAction: inform` maps to `warn`). The violation appears in the ACM Governance dashboard under `gk-rbac-clusteradmin`.
- Test 5b — wildcard ClusterRole detection. Create a ClusterRole with wildcard verbs/resources and bind it:
  ```bash
  oc create clusterrole test-gk-wildcard --verb='*' --resource='*'
  oc create clusterrolebinding test-gk-wildcard-binding --clusterrole=test-gk-wildcard --user=test-gk-user
  ```
  Gatekeeper flags this CRB under `gk-rbac-wildcard-roles` because the referenced ClusterRole has wildcard verbs and resources.
- Clean up:
  ```bash
  oc delete clusterrolebinding test-gk-admin test-gk-wildcard-binding
  oc delete clusterrole test-gk-wildcard
  ```

---

## References

- [Multicluster Role Assignment Operator](https://github.com/stolostron/multicluster-role-assignment)
- [RBAC Model Around Cluster Sets in RHACM](https://www.redhat.com/en/blog/rbac-model-around-cluster-sets-in-red-hat-advanced-cluster-management-for-kubernetes)
- [Fine-Grained RBAC in RHACM 2.14](https://medium.com/@yakovbeder/fine-grained-rbac-in-rhacm-2-14-because-all-or-nothing-was-so-last-season-fd24f7394ef4)
- [ACM Policy Samples — ACS Install](https://github.com/bry-tam/acm-policy-samples/tree/main/policies/operators/acs)

You can force a sync of the ArgoCD Application generated by the ApplicationSet using:

oc get applicationset -n openshift-gitops

argocd app sync  --server openshift-gitops-server-openshift-gitops.apps.

Or if you don't have the argocd CLI, you can trigger a sync via the oc CLI by patching the Application's operation field:

oc -n openshift-gitops patch applicationset  --type merge -p '{"operation":{"initiatedBy":{"username":"admin","automated":false},"sync":{"revision":"HEAD","prune":true}}}'
To find the exact application name for the compliance operator, run:

oc get applications -n openshift-gitops --no-headers | grep compliance
If you want to sync all applications from the ApplicationSet at once:

for app in $(oc get applications -n openshift-gitops -o name); do
  oc -n openshift-gitops patch "$app" --type merge -p '{"operation":{"initiatedBy":{"username":"admin","automated":false},"sync":{"revision":"HEAD","prune":true}}}'
done

# Auxiliary commands

### Step 4 — Verify Hub Resources

Confirm all hub resources were created correctly:

```bash
# Namespace
oc get namespace acm-policies

# RBAC
oc get clusterrole openshift-gitops-policy-admin
oc get clusterrolebinding openshift-gitops-policy-admin

# ManagedClusterSets
oc get managedclusterset mceprod mcedev

# ManagedClusterSetBindings
oc get managedclustersetbinding -n acm-policies

# GitOps operator
oc get csv -n openshift-gitops-operator

# PolicyGenerator plugin
oc get argocd openshift-gitops -n openshift-gitops -o jsonpath='{.spec.repo.initContainers[0].name}'

# ArgoCD server Route and SSO
oc get route openshift-gitops-server -n openshift-gitops
oc get pod -n openshift-gitops -l app.kubernetes.io/name=openshift-gitops-dex-server

# ApplicationSet controller
oc get pod -n openshift-gitops -l app.kubernetes.io/name=openshift-gitops-applicationset-controller

# ACM-ArgoCD integration
oc get gitopscluster -n openshift-gitops
```

### Step 5 — Patch ArgoCD TLS Certs (self-signed ingress only)

> **When is this needed?** Only when the OpenShift cluster uses self-signed certificates on its ingress router (the default for fresh installs). ArgoCD's Dex SSO connector calls the OpenShift OAuth server via the cluster Route. If the Route uses a self-signed CA, ArgoCD cannot verify the TLS certificate during the OIDC callback and SSO login fails. This step injects the router CA into ArgoCD's trust store. Skip this step if your cluster uses a publicly trusted certificate (e.g., Let's Encrypt).

```bash
ROUTE_HOST=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')
CA_CERT=$(oc get secret -n openshift-ingress-operator router-ca -o jsonpath='{.data.tls\.crt}' | base64 -d)

oc patch configmap argocd-tls-certs-cm -n openshift-gitops \
  --type merge \
  -p "{\"data\":{\"${ROUTE_HOST}\": $(echo "$CA_CERT" | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read()))')}}"

oc delete pod -n openshift-gitops -l app.kubernetes.io/name=openshift-gitops-server
```

The ArgoCD server pod restarts and picks up the ingress CA, allowing it to verify the OIDC token via the Route. See Red Hat KB [solutions/7128442](https://access.redhat.com/solutions/7128442) for details.