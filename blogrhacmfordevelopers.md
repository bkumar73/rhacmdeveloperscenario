### Leveraging RHACM for Developers: A Comprehensive Guide

One common question we receive is: *How can developers effectively use Red Hat Advanced Cluster Management (RHACM)?* In this post, we'll guide you through setting up RHACM to empower developers while maintaining necessary constraints.

#### Overview

Our goal is to set up a controlled environment where developers can:

- Work within their designated `preconfigured-namespace` on the hub.
- Deploy applications to clusters within their `developer-clusterset`.
- Edit `Placements` within their cluster set.
- Avoid modifying critical resources like `bindings` and `clustersets`.
- Control RBAC both on the Hub and the Managed-Clusters

Let’s walk through the process step by step.

#### Step-by-Step Implementation

1. **Create a Developer Namespace on the Hub**

  Define a namespace specifically for developers to isolate their activities.

 ```yaml
 ---
 apiVersion: v1
 kind: Namespace
 metadata:
   name: developer
   annotations:
   labels:
     kubernetes.io/metadata.name: developer
 ```
 #### Create a Developer Group on the Hub

 Set up a group for developers to manage permissions.

   ```yaml
   ---
   apiVersion: user.openshift.io/v1
   kind: Group
   metadata:
     name: developer
   users:
     - developer
   ```
  #### Grant Admin Role to the Developer Group

   Assign the admin role to the developer group within their namespace.

   ```yaml
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: group-admin-binding
     namespace: developer
   subjects:
     - kind: Group
       name: developer
       apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: Role
     name: admin
     apiGroup: rbac.authorization.k8s.io
   ```
 
  #### Create a Managed Cluster Set for Developers

   Define a Managed Cluster Set for developers to access.

   ```yaml
   ---
   apiVersion: cluster.open-cluster-management.io/v1beta2
   kind: ManagedClusterSet
  metadata:
    name: developer
  spec:
    clusterSelector:
      selectorType: ExclusiveClusterSetLabel
   ```
  #### Grant View Permissions to the Developer Group

   Allow the developer group to view clusters within the Managed Cluster Set.

   ```yaml
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: developer-crb
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: open-cluster-management:managedclusterset:view:developer
   subjects:
     - kind: Group
       name: developer
       apiGroup: rbac.authorization.k8s.io
   ```
   
  #### Create a Custom ClusterRole for UI Creation

   Define a ClusterRole that grants the necessary permissions for UI-based resource management.

   ```yaml
   ---
   kind: ClusterRole
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: open-cluster-management:subscription-admin-customized
   rules:
  - verbs:
      - create
      - get
      - list
      - watch
      - update
      - delete
      - deletecollection
      - patch
    apiGroups:
      - argoproj.io
    resources:
      - applications
      - applications/status
      - argocds
      - applicationsets 
  - verbs:
      - create
      - get
      - list
      - watch
      - update
      - delete
      - deletecollection
      - patch
    apiGroups:
      - app.k8s.io
    resources:
      - applications
  - verbs:
      - '*'
    apiGroups:
      - apps.open-cluster-management.io
    resources:
      - '*'
  - verbs:
      - '*'
    apiGroups:
      - ''
    resources:
      - configmaps
      - secrets
      - namespaces
  - verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
    apiGroups:
      - cluster.open-cluster-management.io
      - register.open-cluster-management.io
      - clusterview.open-cluster-management.io
    resources:
      - gitopsclusters
      - multiclusterapplicationsetreports
      - managedclustersets/join
      - managedclustersets/bind
      - managedclusters/accept
      - managedclustersets
      - managedclusters
      - managedclustersetbindings
      - placements
      - placementdecisions
```
#### Bind Namespace to the Developer Cluster Set

Associate the developer namespace with the Managed Cluster Set.

```yaml
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSetBinding
metadata:
  name: developer
  namespace: developer
spec:
  clusterSet: developer
```
#### Create a Placement for Developer Deployments

Define a Placement that restricts deployments to the developer clusters.

```yaml
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: developer-gitops-placement
  namespace: developer
spec:
  clusterSets:
    - developer
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: env
              operator: In
              values:
                - developer
```
#### Configure ArgoCD in the Developer Namespace

Set up ArgoCD within the developer namespace to manage applications.

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: developer
  namespace: developer
spec:
  applicationSet:
    resources:
      limits:
        cpu: "2"
        memory: 1Gi
      requests:
        cpu: 250m
        memory: 512Mi
  controller:
    resources:
      limits:
        cpu: "2"
        memory: 2Gi
      requests:
        cpu: 250m
        memory: 1Gi
  dex:
    openShiftOAuth: true
    resources:
      limits:
        cpu: 500m
        memory: 256Mi
      requests:
        cpu: 250m
        memory: 128Mi
  grafana:
    enabled: false
    ingress:
      enabled: false
    resources:
      limits:
        cpu: 500m
        memory: 256Mi
      requests:
        cpu: 250m
        memory: 128Mi
  ha:
    enabled: false
    resources:
      limits:
        cpu: 500m
        memory: 256Mi
      requests:
        cpu: 250m
        memory: 128Mi
  prometheus:
    enabled: false
    ingress:
      enabled: false
  rbac:
    policy: |
      g, system:cluster-admins, role:admin
      g, developer, role:admin
    scopes: '[groups]'
  redis:
    resources:
      limits:
        cpu: 500m
        memory: 256Mi
      requests:
        cpu: 250m
        memory: 128Mi
  repo:
    resources:
      limits:
        cpu: "1"
        memory: 1Gi
      requests:
        cpu: 250m
        memory: 256Mi
  resourceExclusions: |
    - apiGroups:
      - tekton.dev
      - policy.open-cluster-management.io/v1
      clusters:
      - '*'
      kinds:
      - TaskRun
      - PipelineRun
      - PlacementRule
      - PlacementBinding
  server:
    autoscale:
      enabled: false
    ingress:
      enabled: false
    resources:
      limits:
        cpu: 500m
        memory: 256Mi
      requests:
        cpu: 125m
        memory: 128Mi
    route:
      enabled: true
  tls:
    ca: {}
```

This example Policy generated a namespace developer on the Managed-Clusters and ensures the namespace is managed by Gitops-Instance.
Else you would get Permission problems deploying an ApplicationSet with remote-namespace developer


```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: remotetest
  namespace: developer
  annotations:
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
    policy.open-cluster-management.io/standards: NIST SP 800-53
spec:
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-namespace-config
        spec:
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: v1
                kind: Namespace
                metadata:
                  name: developer
                  labels:
                    argocd.argoproj.io/managed-by: openshift-gitops
          remediationAction: inform
          severity: low
  remediationAction: enforce
  ```

This OperatorPolicy installs GitopsOperator, it provides advanced OperatorManagement.  When using RHACM we generally recommend that Operators are installed via Policy.
---
```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: gitopsoperator
  namespace: open-cluster-management-global-set
  annotations:
    policy.open-cluster-management.io/categories: ""
    policy.open-cluster-management.io/standards: ""
    policy.open-cluster-management.io/controls: ""
spec:
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1beta1
        kind: OperatorPolicy
        metadata:
          name: install-operator
        spec:
          remediationAction: enforce
          severity: critical
          complianceType: musthave
          subscription:
            name: openshift-gitops-operator
            namespace: openshift-gitops
            channel: stable
            source: redhat-operators
            sourceNamespace: openshift-marketplace
          upgradeApproval: Automatic
          versions:
          operatorGroup:
            name: default
            targetNamespaces:
              - openshift-gitops
```




