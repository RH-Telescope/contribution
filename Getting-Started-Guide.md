# Getting Started with Project Telescope

## Pre-requisites:

- Openshift 4.10 Cluster with the following operators (take defaults during operator install):
 - Red Hat OpenShift GitOps, 1.5.8 provided by Red Hat Inc.
 - Advanced Cluster Security for Kubernetes 3.73.1 provided by Red Hat

##  CLI Tools installed:
### oc
```
# subscription-manager repos --enable="rhocp-4.11-for-rhel-8-x86_64-rpms"
# sudo yum install openshift-clients
```

### argocd
```
# curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
# chmod +x /usr/local/bin/argocd
```
If you need to create a cluster, easily create a Single Node Openshift (SNO) with the assisted installer at cloud.redhat.com.

In the future a hybrid cloud pattern will be available to install operators, plugins and applications on a pre-existing cluster.

## Initial Deployment Use Case
This first use case is for an individual user tenant, ie telescope-username, deployment with Advanced Cluster Security (ACS) integration.
## Step by Step Procedure
### Step 1: Login via the CLI to the Openshift Cluster with oc login
Example:
```
$ oc login --token=token --server=https://api.cluster….com:6443
(update with token or user/pass and server)
```
Output:
```
Logged into "https://api.cluster……..com:6443" as "user" using the token provided.
```
### Step 2: Edit Service Account Permissions

Add cluster-admin rights to the openshift-gitops-argocd-application-controller service account in the openshift-gitops namespace

Example:
```
$ oc adm policy add-cluster-role-to-user cluster-admin -z openshift-gitops-argocd-application-controller -n openshift-gitops
```

Output:
```
clusterrole.rbac.authorization.k8s.io/cluster-admin added: “openshift-gitops-argocd-application-controller"
Step 3: Retrieve Gitops url
```
### Step 3: Retrieve Gitops url

Example:
```
$ argoURL=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}')
$ echo $argoURL
openshift-gitops-server-openshift-gitops.apps.cluster……com
```

### Step 4: Retrieve Gitops admin password

Example:
```
$ argoPass=$(oc get secret/openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)
$ echo $argoPass
0ZqHQWQWQWQWQWQWQWQWQWQWLy
```
### Step 5: Login to Gitops

Example:
```
$ argocd login --insecure --grpc-web $argoURL  --username admin --password $argoPass
```

Output:
```
'admin:login' logged in successfully
Context 'openshift-gitops-server-openshift-gitops.apps.cluster…….com' updated
```
### Step 5: Create the deployment file.  
Copy the below into a file and modify username and server. (will be moved to getting started repo)

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  name: telescope-username
  namespace: openshift-gitops
spec:
  destination:
    namespace: telescope-username
    server: 'https://kubernetes.default.svc'
  project: default
  source:
    chart: telescope-argocd
    helm:
      values: |
        applicationPrefix: username
        charts:
          telescope-frontend:
            values:
              backendUrl: https://telescope-backend-telescope-username.apps.ztcluster.zerotrust.foundation
    repoURL: 'https://rh-telescope.github.io/helm-charts'
    targetRevision: x
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```
### Step 6: Deploy the application
Example:
```
$ oc apply -f ./deployment.yaml
application.argoproj.io/telescope-username created
```

### Step 7: Check the status of the deployment
Example:
```
$ argocd app list
```
### Step 8: Get the frontend url
Example:
```
$ oc get route telescope-frontend -n telescope-kpeeples -o jsonpath='{.spec.host}{"\n"}'
telescope-frontend-telescope-username.apps.cluster………..com
```
### Step 9: SUCCESS, BROWSE TO TELESCOPE!

This is a living document so please check back for any updates.
