#Multi Cluster

First, deploy Orchestra

## Create Organization for the First Cluster

1.  In `src/main/webapp/WEB-INF/unison.xml` find the `<org>` XML element.  Add once for the main cluster called "Cluster 1".  The `uuid` is arbitrary but must be unique amongst all organizations.
2.  Move `Kubernetes Namespace Administrators`, `Kubernetes Administration`, `Kubernetes Namespace Viewers` into `Cluster 1`
3.  Look for `<portal>` in  unison.xml, for    `Kubernetes Dashboard`, `New Kubernetes Namespace` and `Kubernetes Tokens` change the `org` attribute to the `uuid` created in #1
4.  In `src/main/webapp/WEB-INF/applications/10-scale.xml` look for `showPortalOrgs`, change it to `true`

## Create SSO with Second Cluster

1.  Create an OpenID Connect Identity Provider -  `src/main/webapp/WEB-INF/applications/41-OpenUnisonIdp.xml` create a `trust` (duplicating the first one is a good idea) replacing `K8S_URL` with `CX_K8S_OU_URL`, `X` is the cluster number)
2.  Change `/redirect_uri` in the new trust to `/auth/oidc`
3.  Update the `openunison` secret with the `CX_K8S_OU_URL` parameter
4.  Deploy the Orchestra OpenID Connect login portal on the new cluster: https://github.com/OpenUnison/openunison-k8s-login-oidc

When configuring the login portal for your new cluster, use the below guide to setup your trust with the primary cluster:

```
OIDC_CLIENT_ID=kubernetes_clusterX
OIDC_CLIENT_SECRET=
OIDC_IDP_AUTH_URL=https://OU_HOST/auth/idp/OpenUnisonIdp/auth
OIDC_IDP_TOKEN_URL=https://OU_HOST/auth/idp/OpenUnisonIdp/token
OIDC_IDP_LIMIT_DOMAIN=
```

Where `X` is the number of your cluster and `OU_HOST` is the same as the value for the `OU_HOST` in your primary cluster.

**NOTE**: If the TLS certificate for your first Orchestra is not commercially signed, you'll need to import the certificate into the `openunison-secrets` in the new cluster.

## Add Portal Links

1.  In `src/main/webapp/WEB-INF/unison.xml` go to `<org>` section, duplicate the `Cluster 1` `orgs` tag, replacing all the `uuid`s with new unique values.
2.  Look for `<portal>` in  unison.xml and duplicate `Kubernetes Dashboard`, `New Kubernetes Namespace` and `Kubernetes Tokens` changing the `org` attribute to the `uuid` created in #1
3.  For each `<urls>` copied, For `Kubernetes Tokens` and `Kubernetes Dashboard` change the authorization rules to have the `constraint` be `(groups=k8s-X-*)` where `X` is the cluster number and change the `url` to point to the values in the second cluster (add environment variables to keep things generic).
4.  for `New Kubernetes Namespace` change `newproject` to `newproject-2` in the `url`

## Add RBAC and ServiceAccount To The Second Cluster

On the second cluster create a service account in the openunison namespace:

```
kubectl create sa openunison-provisioning -n openunison
```

Create an RBAC policy and binding for remote management:

```
---
kind: ClusterRole
apiVersion: "rbac.authorization.k8s.io/v1"
metadata: 
 name: "list-namespaces"
rules: 
 - apiGroups: 
   - ""
   resources: 
   - namespaces
   verbs: 
   - list
---
kind: ClusterRoleBinding
apiVersion: "rbac.authorization.k8s.io/v1"
metadata: 
 name: "openunison-cluster-list-namespaces"
subjects: 
 - kind: Group
   name: users
   apiGroup: "rbac.authorization.k8s.io"
roleRef: 
 kind: ClusterRole
 name: "list-namespaces"
 apiGroup: "rbac.authorization.k8s.io"
---
kind: ClusterRoleBinding
apiVersion: "rbac.authorization.k8s.io/v1"
metadata: 
 name: "openunison-cluster-administrators"
subjects: 
 - kind: Group
   name: "k8s-2-cluster-administrators"
   apiGroup: "rbac.authorization.k8s.io"
 - kind: ServiceAccount
   name: openunison-provisioning
   namespace: openunison
roleRef: 
 kind: ClusterRole
 name: "cluster-admin"
 apiGroup: "rbac.authorization.k8s.io"
 ```

 ## Create administrators group in the database

 `insert into localGroups (name,description) values ('k8s-2-cluster-administrators','Kubernetes cluster 2 administrators');`

 ## Create the cluster provisioning target

 1. Copy `src/main/webapps/WEB-INF/targets/20-k8s.xml` to `src/main/webapps/WEB-INF/targets/30-k8s2.xml`
 2. In `k8s2.xml` rename the target to `k8s2`
 3. Add in `#[CLUSTER2_API_SERVER]` and `#[CLUSTER2_API_TOKEN]` for `url` and `token`, also add these to your ou.env secrets
 4. Import the second cluster's api server's certificate into the primary's keystore

## Workflow - Cluster2 Administrator

1. Copy `src/main/webapps/WEB-INF/workflows/40-ClusterAdmin.xml` to `src/main/webapps/WEB-INF/workflows/41-ClusterAdminCluster2.xml`
2. in `41-ClusterAdminCluster2.xml` change the name to `ClusterAdminCluster2`
3. Change the `orgid` to the same `orgid` you created for `Kubernetes Administration` for cluster 2
4. Search for `k8s-cluster-administrators` and replace with `k8s-2-cluster-administrators`
5. Update the `label`, `description` accordingly

## Workflow - Cluster2 Namespace Admins

1. Copy `src/main/webapps/WEB-INF/workflows/20-ProjectAdministrators.xml` to `src/main/webapps/WEB-INF/workflows/21-ProjectAdministrators2.xml`
2. in `21-ProjectAdministrators2.xml` change the name to `ProjectAdministrators2`
3. Change the `orgid` to the same `orgid` you created for `Kubernetes Namespace Viewers` for cluster 2
4. Search for `approvers-k8s-` and replace with `approvers-k8s-2-`
5. Search for `cn=k8s-cluster-administrators` and replace with `cn=k8s-2-cluster-administrators`
6. Search for `k8s-namespace-administrators-` and replace with `k8s-2-namespace-administrators-`
7. Search for `<param name="targetName" value="k8s"/>` and replace with `<param name="targetName" value="k8s2"/>`
8. Update the `label`, `description` accordingly
9. Change the `target` in the `dynamicConfiguration` to `k8s2`


## Workflow - Cluster2 Namespace Viewer

1. Copy `src/main/webapps/WEB-INF/workflows/50-ProjectViewers.xml` to `src/main/webapps/WEB-INF/workflows/51-ProjectViewers2.xml`
2. in `51-ProjectViewers2.xml` change the name to `ProjectViewers2`
3. Change the `orgid` to the same `orgid` you created for `Kubernetes Namespace Administrators` for cluster 2
4. Search for `approvers-k8s-` and replace with `approvers-k8s-2-`
5. Search for `cn=k8s-cluster-administrators` and replace with `cn=k8s-2-cluster-administrators`
6. Search for `k8s-namespace-viewer-` and replace with `k8s-2-namespace-viewer-`
7. Search for `<param name="targetName" value="k8s"/>` and replace with `<param name="targetName" value="k8s2"/>`
8. Update the `label`, `description` accordingly
9. Change the `target` in the `dynamicConfiguration` to `k8s2`

## Workflow - Cluster 2 New Namespace

1. Copy `src/main/webapps/WEB-INF/workflows/30-NewK8SNamespace.xml` to `src/main/webapps/WEB-INF/workflows/31-NewK8SNamespace2.xml`
2. in `31-NewK8SNamespace2.xml` change the name to `NewK8SNamespace2`
3. Search for `approvers-k8s-` and replace with `approvers-k8s-2-`
4. Search for `cn=k8s-cluster-administrators` and replace with `cn=k8s-2-cluster-administrators`
5. Search for `k8s-namespace-viewer-` and replace with `k8s-2-namespace-viewer-`
6. Search for `k8s-namespace-administrators-` and replace with `k8s-2-namespace-administrators-`
7. Search for `<param name="targetName" value="k8s"/>` and replace with `<param name="targetName" value="k8s2"/>`
8. Update the `label`, `description` accordingly
9. Change the `target` in the `dynamicConfiguration` to `k8s2`

## Application ScaleJS Register (New Namespace)

1. Duplicate `src/main/webapps/WEB-INF/applications/10-scale.xml` to `src/main/webapps/WEB-INF/applications/11-cluster2-new-namespace.xml` 
2. Change the name to `cluster2-new-namespace`
3. Replace `/newproject` with `/newproject2`
4. Change `<param name="callWorkflowInit" value="targetName=k8s" />` to `<param name="callWorkflowInit" value="targetName=k8s2" />`
5. Change `<param name="callWorkflowInit" value="workflowName=NewK8SNamespace" />` to `<param name="callWorkflowInit" value="workflowName=NewK8SNamespace2" />`
6. Copy `src/main/webapp/newproject` to `src/main/webapps/newproject2`