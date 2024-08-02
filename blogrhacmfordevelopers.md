### Leveraging RHACM for Developers: A Comprehensive Guide

One common question we receive is: *How can developers effectively use Red Hat Advanced Cluster Management (RHACM)?* In this post, we'll guide you through setting up RHACM to empower developers while maintaining necessary constraints.

#### Overview

Our goal is to set up a controlled environment where developers can:

- Work within their designated `preconfigured-namespace` on the hub.
- Deploy applications to clusters within their `developer-clusterset`.
- Edit `Placements` within their cluster set.
- Avoid modifying critical resources like `bindings` and `clustersets`.
- Adhere to restrictions on `ManagedCluster` resources.

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
       openshift.io/sa.scc.mcs: s0:c39,c29
       openshift.io/sa.scc.supplemental-groups: 1001540000/10000
       openshift.io/sa.scc.uid-range: 1001540000/10000
     labels:
       kubernetes.io/metadata.name: developer
       openshift-pipelines.tekton.dev/namespace-reconcile-version: 1.14.3
       pod-security.kubernetes.io/audit: baseline
       pod-security.kubernetes.io/audit-version: v1.24
       pod-security.kubernetes.io/warn: baseline
       pod-security.kubernetes.io/warn-version: v1.24
   ---
Create a Developer Group on the Hub

Set up a group for developers to manage permissions.

yaml
Copy code
---
apiVersion: user.openshift.io/v1
kind: Group
metadata:
  name: developer
users:
  - developer
---
Grant Admin Role to the Developer Group

Assign the admin role to the developer group within their namespace.

yaml
Copy code
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
---  
Create a Managed Cluster Set for Developers

Define a Managed Cluster Set for developers to access.

yaml
Copy code
---
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSet
metadata:
  name: developer
spec:
  clusterSelector:
    selectorType: ExclusiveClusterSetLabel
---
Grant View Permissions to the Developer Group

Allow the developer group to view clusters within the Managed Cluster Set.

yaml
Copy code
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
---
Create a Custom ClusterRole for UI Creation

Define a ClusterRole that grants the necessary permissions for UI-based resource management.

yaml
Copy code
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
---
Bind Namespace to the Developer Cluster Set

Associate the developer namespace with the Managed Cluster Set.

yaml
Copy code
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSetBinding
metadata:
  name: developer
  namespace: developer
spec:
  clusterSet: developer
---
Create a Placement for Developer Deployments

Define a Placement that restricts deployments to the developer clusters.

yaml
Copy code
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
---
Configure ArgoCD in the Developer Namespace

Set up ArgoCD within the developer namespace to manage applications.

yaml
Copy code
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
---
Create a Policy for Managed Service Account and Cluster Permissions

Define a policy to manage service accounts and permissions across clusters.

yaml
Copy code
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-gitops
  namespace: openshift-gitops
  annotations:
    policy.open-cluster-management.io/standards: NIST-CSF
    policy.open-cluster-management.io/categories: PR.PT Protective Technology
    policy.open-cluster-management.io/controls: PR.PT-3 Least
