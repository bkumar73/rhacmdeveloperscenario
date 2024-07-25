### Sometimes we are getting the question:

How can you use RHACM for Developers?   

In the following I would like to explain an approach how it could be implemented.


a developer:

* should work in RHACM-UI, but only in his own `preconfigured-namespace`
* he should deploy to clusters in his `developer-clusterset`
* he should not be able to modify any `bindings, clustersets`, etc
* he should be able to edit `Placements in his clusterSet`

Let us go step by step:

1. create developer namespace on the hub

```
---
apiVersion: v1
kind: Namespace
metadata:
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
  name: developer
---
```

2. create developer group on the hub

```
---
apiVersion: user.openshift.io/v1
kind: Group
metadata:
  name: developer
users:
  - developer
---
```

3. Grant developer admin-role in this namespace on the Hub

```
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
```

4. Create Managed-Cluster-Set for developers

```
---
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSet
metadata:
  annotations:
    cluster.open-cluster-management.io/submariner-broker-ns: default-broker
  name: developer
spec:
  clusterSelector:
    selectorType: ExclusiveClusterSetLabel
---
```


5. Give developer group view permissions to the Clusterset

```
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
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: developer
---
```

Now we create a ClusterRole which enables the UI-create feature.

```
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: 'open-cluster-management:subscription-admin-customized'
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
6. bind namespace to the developer clusterset

```
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSetBinding
metadata:
  name: demo
  namespace:  developer
spec:
  clusterSet:  developer
---
```

7. Create a Placement that developer can deploy to developer clusters

```
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name:  developer-gitops-placement
  namespace:  developer
spec:
  clusterSets:
    - demo
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: env
              operator: In
              values:
                -  developer
---
```
8.  Configure ArgoCD in the developer namespace
```
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
    processors: {}
    resources:
      limits:
        cpu: "2"
        memory: 2Gi
      requests:
        cpu: 250m
        memory: 1Gi
    sharding: {}
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
    route:
      enabled: false
  ha:
    enabled: false
    resources:
      limits:
        cpu: 500m
        memory: 256Mi
      requests:
        cpu: 250m
        memory: 128Mi
  initialSSHKnownHosts: {}
  prometheus:
    enabled: false
    ingress:
      enabled: false
    route:
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
    grpc:
      ingress:
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
    service:
      type: ""
  tls:
    ca: {}
---
```
      
9. On the Hub-Cluster create a Policy to rollout Managed-Service-Account and Cluster Permissions

```
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-gitops
  namespace: openshift-gitops
  annotations:
    policy.open-cluster-management.io/standards: NIST-CSF
    policy.open-cluster-management.io/categories: PR.PT Protective Technology
    policy.open-cluster-management.io/controls: PR.PT-3 Least Functionality
spec:
  remediationAction: enforce
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-gitops-sub
        spec:
          pruneObjectBehavior: None
          remediationAction: enforce
          severity: low
          object-templates-raw: |
            {{ range $placedec := (lookup "cluster.open-cluster-management.io/v1beta1" "PlacementDecision" "default" "" "cluster.open-cluster-management.io/placement=developer-gitops-placement").items }}
            {{ range $clustdec := $placedec.status.decisions }}
            - complianceType: musthave
              objectDefinition:
                apiVersion: authentication.open-cluster-management.io/v1alpha1
                kind: ManagedServiceAccount
                metadata:
                  name: managed-sa-sample
                  namespace: {{ $clustdec.clusterName }}
                spec:
                  rotation: {}
            - complianceType: musthave
              objectDefinition:
                apiVersion: rbac.open-cluster-management.io/v1alpha1
                kind: ClusterPermission
                metadata:
                  name: clusterpermission-msa-subject-sample
                  namespace: {{ $clustdec.clusterName }}
                spec:
                  roles:
                  - namespace: developer
                    rules:
                    - apiGroups: ["apps"]
                      resources: ["deployments"]
                      verbs: ["get", "list", "create", "update", "delete"]
                    - apiGroups: [""]
                      resources: ["configmaps", "secrets", "pods", "podtemplates", "persistentvolumeclaims", "persistentvolumes"]
                      verbs: ["get", "update", "list", "create", "delete"]
                    - apiGroups: ["storage.k8s.io"]
                      resources: ["*"]
                      verbs: ["list"]
                  - namespace: mortgage
                    rules:
                    - apiGroups: ["apps"]
                      resources: ["deployments"]
                      verbs: ["get", "list", "create", "update", "delete"]
                    - apiGroups: [""]
                      resources: ["configmaps", "secrets", "pods", "services", "namespace"]
                      verbs: ["get", "update", "list", "create", "delete"]
                  clusterRole:
                    rules:
                    - apiGroups: ["*"]
                      resources: ["*"]
                      verbs: ["get", "list"]
                  roleBindings:
                  - namespace: default
                    roleRef:
                      kind: Role
                    subject:
                      apiGroup: authentication.open-cluster-management.io
                      kind: ManagedServiceAccount
                      name: managed-sa-sample
                  roleBindings:
                  - namespace: mortgage
                    roleRef:
                      kind: Role
                    subject:
                      apiGroup: authentication.open-cluster-management.io
                      kind: ManagedServiceAccount
                      name: managed-sa-sample
                  clusterRoleBinding:
                    subject:
                      apiGroup: authentication.open-cluster-management.io
                      kind: ManagedServiceAccount
                      name: managed-sa-sample
            {{ end }}
            {{ end }}
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: binding-policy-gitops
  namespace: openshift-gitops
placementRef:
  name: local-cluster-placement
  kind: Placement
  apiGroup: cluster.open-cluster-management.io
subjects:
  - name: policy-gitops
    kind: Policy
    apiGroup: policy.open-cluster-management.io
```    
       

9. Create Gitops-ClusterResource in the namespace

```
---
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata: 
  name: developer-gitops-cluster
  namespace: developer
spec:
  managedServiceAccountRef: "msa"
  argoServer:
    cluster: local-cluster
    argoNamespace: developer
  placementRef:
    kind: Placement
    apiVersion: cluster.open-cluster-management.io/v1beta1
    name: developer-gitops-placement   
```


Now when you login as a Developer you can deploy to any Cluster in your Developer-Clusterset.


```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
 name: developer-app-metrics-role
rules:
 - apiGroups:
     - "cluster.open-cluster-management.io"
   resources:
     - managedclusters # fixed
   Verbs: # represents namespaces of managed clusters
     - metrics/developer
--- 
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: developer-app-metrics-role-binding
subjects:
 - kind: Group
   apiGroup: rbac.authorization.k8s.io
   name: developer
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: ClusterRole
 name: developer-app-metrics-role
```
  



















