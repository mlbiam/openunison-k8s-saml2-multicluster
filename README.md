# Orchestra For Kubernetes - SAML2


Orchestra is an automation portal for Kubernetes built on OpenUnison.  Orachestra integrates a user's identity into Kubernetes enabling:

1. SSO between the API server and your LDAP infrastructure
2. SSO with the Kubernetes Dashboard
3. Self service access to existing Namespaces
4. Self service creation of new Namespaces
5. Workflows for automating access approvals without getting system administrators involved
6. Built in self service reporting

![Orchestra Portal Screen Shot](imgs/orchestra-portal-screenshot.png)

When a user accesses Kubernetes using Orchestra, they'll access both the self service portal and the dashboard through OpenUnison's reverse proxy (instead of directly via an ingress).  OpenUnison will inject the user's identity into each request, allowing the dashboard to act on their behalf.

Orchestra stores all Kubernetes access information as a groups inside of a relational database, as opposed to a group in an external directory.  OpenUnison will create the appropriate Roles and RoleBindings to allow for the access.


![Orchestra Architecture](imgs/openunison_qs_kubernetes.png)

When a user accesses Kubernetes using OpenUnison, they'll access both te self service portal and the dashboard through OpenUnison (instead of directly via an ingress).  OpenUnison will inject the user's identity into each request, allowing the dashboard to act on their behalf.

The OpenUnison deployment stores all Kubernetes access information as a groups inside of a relational database, as opposed to a group in an external directory.  OpenUnison will create the appropriate Roles and RoleBindings to allow for the access.

# Roles Supported

## Cluster

1.  Administration - Full cluster management access

## Namespace

1.  Administrators - All operations inside of a namespace
2.  Viewers - Can view contents of a namespace (except `Secret`s), but can not make changes

## Non-Kubernetes

1.  System Approver - Able to approve access to roles specific to OpenUnison
2.  Auditor - Able to view audit reports, but not request projects or approve access

# Deployment

## What You Need To Start

Prior to deploying OpenUnison you will need:

1. Kubernetes 1.10 or higher
2. The Nginx Ingress Controler deployed (https://kubernetes.github.io/ingress-nginx/deploy/)
3. A MySQL or MariaDB Database
4. The SAML2 Metadata for your identity provider
5. An SMTP server for sending notifications
6. Deploy the dashboard to your cluster

## Create Environments File

Orchestra stores environment specific information, such as host names, passwords, etc, in a properties file that will then be loaded by OpenUnison and merged with its configuration.  This file will be stored in Kubernetes as a secret then accessed by OpenUnison on startup to fill in the `#[]` parameters in `unison.xml` and `myvd.conf`.  For instance the parameter `#[OU_HOST]` in `unison.xml` would have an entry in this file.  Below is an example `input.props` file:

```properties
OU_HOST=k8sou.tremolo.lan
K8S_DASHBOARD_HOST=k8sdb.tremolo.lan
K8S_URL=https://k8s-installer-master.tremolo.lan:6443
OU_HIBERNATE_DIALECT=org.hibernate.dialect.MySQL5InnoDBDialect
OU_QUARTZ_DIALECT=org.quartz.impl.jdbcjobstore.StdJDBCDelegate
OU_JDBC_DRIVER=com.mysql.jdbc.Driver
OU_JDBC_URL=jdbc:mysql://dbs.tremolo.lan:3308/unison
OU_JDBC_USER=root
OU_JDBC_PASSWORD=start123
OU_JDBC_VALIDATION=SELECT 1
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=donotreply@domain.com
SMTP_PASSWORD=xxxx
SMTP_FROM=donotreply@domain.com
SMTP_TLS=true
OU_CERT_OU=k8s
OU_CERT_O=Tremolo Security
OU_CERT_L=Alexandria
OU_CERT_ST=Virginia
OU_CERT_C=US
unisonKeystorePassword=start123
USE_K8S_CM=true
SESSION_INACTIVITY_TIMEOUT_SECONDS=900
```

*Detailed Description or Properties*

| Property | Description |
| -------- | ----------- |
| OU_HOST  | The host name for OpenUnison.  This is what user's will put into their browser to login to Kubernetes |
| K8S_DASHBOARD_HOST | The host name for the dashboard.  This is what users will put into the browser to access to the dashboard. **NOTE:** `OU_HOST` and `K8S_DASHBOARD_HOST` **MUST** share the same DNS suffix. Both `OU_HOST` and `K8S_DASHBOARD_HOST` **MUST** point to OpenUnison |
| K8S_URL | The URL for the Kubernetes API server |
| OU_HIBERNATE_DIALECT | Hibernate dialect for accessing the database.  Unless customizing for a different database do not change |
| OU_QUARTZ_DIALECT | Dialect used by the Quartz Scheduler.  Unless customizing for a different database do not change  |
| OU_JDBC_DRIVER | JDBC driver for accessing the database.  Unless customizing for a different database do not change |
| OU_JDBC_URL | The URL for accessing the database |
| OU_JDBC_USER | The user for accessing the database |
| OU_JDBC_PASSWORD | The password for accessing the database |
| OU_JDBC_VALIDATION | A query for validating database connections/ Unless customizing for a different database do not change |
| SMTP_HOST | Host for an email server to send notifications |
| SMTP_PORT | Port for an email server to send notifications |
| SMTP_USER | Username for accessing the SMTP server (may be blank) |
| SMTP_PASSWORD | Password for accessing the SMTP server (may be blank) |
| SMTP_FROM | The email address that messages from OpenUnison are addressed from |
| SMTP_TLS | true or false, depending if SMTP should use start tls |
| OU_CERT_OU | The `OU` attribute for the forward facing certificate |
| OU_CERT_O | The `O` attribute for the forward facing certificate |
| OU_CERT_L | The `L` attribute for the forward facing certificate |
| OU_CERT_ST | The `ST` attribute for the forward facing certificate |
| OU_CERT_C | The `C` attribute for the forward facing certificate |
| unisonKeystorePassword | The password for OpenUnison's keystore |
| USE_K8S_CM | Tells the deployment system if you should use k8s' built in certificate manager.  If your distribution doesn't support this (such as Canonical and Rancher), set this to false |
| SESSION_INACTIVITY_TIMEOUT_SECONDS | The number of seconds of inactivity before the session is terminated, also the length of the refresh token's session |
| MYVD_CONFIG_PATH | The path to the MyVD configuration file, unless being customized, use `WEB-INF/myvd.conf` |

## Prepare Deployment

Perform these steps from a location with a working `kubectl` configuration:

1. Create a directory to store secrets, ie `/path/to/secrets` and put `input.props` (the properties file defined above) in that directory
2. Create a directory for config maps, ie `/path/to/configmaps`, copy the metadata from your SAML2 identity provider to `/path/to/configmaps/saml2-metadata.xml`


## Deployment

Based on where you put the files from `Prepare Deployment`, run the following:

```
curl https://raw.githubusercontent.com/TremoloSecurity/kubernetes-artifact-deployment/master/src/main/bash/deploy_openunison.sh | bash -s /path/to/configmaps /path/to/secrets https://raw.githubusercontent.com/OpenUnison/openunison-k8s-saml2/master/src/main/yaml/artifact-deployment.yaml
```

The output will look like:

```
namespace/openunison-deploy created
configmap/extracerts created
secret/input created
clusterrolebinding.rbac.authorization.k8s.io/artifact-deployment created
job.batch/artifact-deployment created
NAME                        READY     STATUS    RESTARTS   AGE
artifact-deployment-jzmnr   0/1       Pending   0          0s
artifact-deployment-jzmnr   0/1       Pending   0         0s
artifact-deployment-jzmnr   0/1       ContainerCreating   0         0s
artifact-deployment-jzmnr   1/1       Running   0         4s
artifact-deployment-jzmnr   0/1       Completed   0         15s
```

Once you see `Completed`, you can exit the script (`Ctl+C`).  This script creates all of the appropriate objects in Kubernetes, signs certificates and deploys both OpenUnison and the Dashboard.  

## Complete Integrate with your Identity Provider

Run `kubectl describe configmap api-server-config -n openunison` to get the metadata for your identity provider.  Import it into your identity provider and add the following attributes to the assertion so OpenUnison knows how the logged in uers is:

| Attribute Name | Active Directory Attribute | Description |
| -------------- | -------------------------- | ----------- |
| uid            | samAccountName             | User's login id |
| givenName      | givenName                  | User's first name |
| sn             | sn                         | User's last name |
| mail           | mail                       | User's email address |

If using Active Directory Federation Services, you can use the following claims transformation rule:
```
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
 => issue(store = "Active Directory", types = ("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier", "uid", "givenName", "sn", "mail"), query = ";sAMAccountName,sAMAccountName,givenName,sn,mail;{0}", param = c.Value);
```

Once the metadata is imported and the attributes are added, you are ready to login to Orchestra.

## Complete SSO Integration with Kubernetes

Run `kubectl describe configmap api-server-config -n openunison` to get the SSO integration artifacts.  The output will give you both the certificate that needs to be trusted and the API server flags that need to be configured on your API servers.

## First Login to Orchestra

At this point you should be able to login to OpenUnison using the host specified in  the `OU_HOST` of your properties.  Once you are logged in, logout.  Users are created in the database "just-in-time", meaning that once you login the data representing your user is created inside of the database deployed for Orchestra.

## Create First Administrator

The user you logged in as is currently unprivileged.  In order for other users to login and begin requesting access to projects this first user must be enabled as an approver.  Login to the MySQL database deployed for Orchestra and execute the following SQL:

```sql
insert into userGroups (userId,groupId) values (2,1);
```

This will add the administrator group to your user.  Logout of OpenUnison and log back in.

## Self Request & Approve Cluster Administrator

Once SSO is enabled in the next step, you'll need a cluster administrator to be able to perform cluster level operations:

1.  Login to Orchestra
2.  Click on "Request Access" in the title bar
3.  Click on "Kubernetes Administration"
4.  Click "Add To Cart" next to "Cluster Administrator"
5.  Next to "Check Out" in the title bar you'll see a red `1`, click on "Check Out"
6.  For "Supply Reason", give a reason like "Initial user" and click "Submit Request"
7.  Since you are the only approver refresh OpenUnison, you will see a red `1` next to "Open Approvals".  Click on "Open Approvals"
8. Click "Review" next to your email address
9. Specify "Initial user" for the "Justification" and click "Approve"
10. Click on "Confirm Approval"

At this point you will be provisioned to the `k8s-cluster-administrators` group in the database that has a RoleBinding to the `cluster-admin` Role.  Logout of Orchestra and log back in.  If you click on your email address in the upper left, you'll see that you have the Role `k8s-cluster-administrators`.  

# Updating Secrets and Certificates

In order to change the secrets or update certificate store:

Download the contents of `openunison-secrets` in the `openunison` namespace into an empty directory

```
kubectl get  secret openunison-secrets -o json  -n openunison | python /path/to/openunison-k8s-saml2/src/main/python/download_secrets.py
```

`download_secrets.py` is a utility script for pulling the files out of secrets and config maps.  Next, make your changes.  You can't apply over an existing secret, so next delete the current secret:

```
kubectl delete secret openunison-secrets -n openunison
```

Finally, create the secret from the directory where you downloaded the secrets:

```
kubectl create secret generic openunison-secrets --from-file=. -n openunison
```

Redeploy Orchestra to pick up the changes.  The easiest way is to update an environment variable in the `openunison` deployment

# Whats next?
Users can now login to create namespaces, request access to cluster admin or request access to other clusters.

Now you can begin mapping OpenUnison's capabilities to your business and compliance needs.  For instance you can add multi-factor authentication with TOTP or U2F, Create privileged workflows for onboarding, scheduled workflows that will deprovision users, etc.