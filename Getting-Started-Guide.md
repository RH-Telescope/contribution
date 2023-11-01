# Getting Started with Project Telescope

This guide introduces you to the components that are included as part of Project Telescope along with a deployment to an OpenShift environment.

## Architecture

Project Telescope includes two primary components:

* Backend - Interfaces with various security tools and exposes the current compliance posture via a RESTful API.
* Frontend - User interface enabling the visualization of the current level of compliance by exposed by the backend service.

Supporting these applications are a set of [Helm](https://helm.sh) charts which simplifies the deployment to an OpenShift environment.

## Pre-requisites:

The following must be satisfied prior to deploying Project Telescope:

* An OpenShift 4.10+ Cluster with elevated permissions to install Operators and resources at a cluster level.

Additional integrations are also needed to support this walkthrough which will be described in the sections below

##  CLI Tools

In order to integrate with the associated resources in use within this guide, several command line tools must be installed and configured.

### OpenShift

`oc` is the comand line tool for interacting with OpenShift environments and can be installed several different methods depending on the target operating system.

#### Binary Installation

The OpenShift CLI binary can be installed on Linux, OSX or Windows based machines and can be obtained from two primary sources:

1. [Red Hat Console](https://console.redhat.com)
2. The OpenShift Web Console.

To use the OpenShift Web Console as a method for obtaining the CLI, launch the web console and click the question mark dropdown on the toolbar. 

![Selecting the Help Dropdown from the OpenShift Web Console](images/getting-started/oc-image-1.png)

Select _Command Line Tools_.

![Command Line Tools Page of the OpenShift Web Console](images/getting-started/oc-image-2.png)

Select the appropriate resource depending on your Operating System.

Once the download has completed, unpack the archive. On a Linux or OSX machine, the archive can be extracted using the following command:

```shell
tar xvzf <file>
```

Move the `oc` binary to a location that is on your `PATH`

#### RPM (RHEL 8)

To download the OpenShift CLI on a Red Hat Enterprise Linux 8 machine, ensure the appropriate repository is enabled and install the CLI from the `openshift-clients` package.

```shell
sudo subscription-manager repos --enable="rhocp-4.12-for-rhel-8-x86_64-rpms"
sudo yum install openshift-clients
```

NOTE: Ensure that you enable the appropriate repository corresponding to the OpenShift version for the target cluster being used.

### Login to the OpenShift CLI

The simpliest method for authenticating the OpenShift CLI is to obtain the login command from the Web Console.

Navigate to the OpenShift Web Console, select the name of the currently authenticated user at the top righthand corner and select **Copy Login Command**. You may be requested to auhenticate once again.

Select the **Display Token** link and copy the provided command underneath the _Log in with this token_ section. Paste the command into the command line to authenticate the CLI. 

### Argo CD

Project Telescope can be deployed using OpenShift GitOps (Argo CD). To interact with the Argo CD server, the `argocd` CLI is used.

Several methods are available for installing Argo CD as listed within the [project documentation](https://argo-cd.readthedocs.io/en/stable/cli_installation/).

Follow the steps as described within the documenttation to install the Argo CD CLI to your local machine.

## OpenShift GitOps

With the installation of the Argo CD CLI to a local machine as described in the prior section, Argo CD itself should be deployed to the OpenShift cluster. The installation is facilitated using the OperatorHub and can be completed with only a few clicks.

Navigate to OpenShift Web Console and within the _Administrator Perspective_, expand **Operators** and then select **OperatorHub**.

Search for **OpenShift GitOps** and select the tile when it appears.

![OpenShift GitOps in OperatorHub](images/getting-started/operatorhub-argocd-1.png).

Click **Install**.

Select the desired Update Channel, approval strategy and then click **Install** to install the OpenShift GitOps Operator.

![Install the OpenShift GitOps Operator](images/getting-started/operatorhub-argocd-2.png).

Once the operator has been deployed, all of the related resources will be configued within the `openshift-gitops` namespace.

While OpenShift GitOps can operate either at a cluster scoped level or in a namespaced scoped (tenant) level, for the purpose of this guide, it will be configured to operate across any OpenShift namespace within the cluster.

Grant the Argo CD Service Account's for the Application and ApplicationSet controller `cluster-admin` privileged

```shell
oc apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openshift-gitops-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: openshift-gitops-argocd-application-controller
  namespace: openshift-gitops
- kind: ServiceAccount
  name: openshift-gitops-applicationset-controller
  namespace: openshift-gitops
EOF
```

Note: Elevated permissions, such as `cluster-admin` simplifies how OpenShift can manage resources across the entire cluster. However, it is recommended that policies be defined that restrict the permissions that are granted to align with the _Privilege of Least Principle_, which are outside the scope of this guide.

Finally, grant any authenticated user to the Argo CD web console _admin_ access by patching the `ArgoCD` resource using the following command:

```shell
oc apply --server-side=true --force-conflicts -f - <<EOF
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
spec:
  rbac:
    defaultPolicy: 'role:admin'
EOF
```

## Deploying Project Telescope

Project Telescope supports multiple deployment paradigms and can either be installed to support a single cluster based deployment or in an tenant scope mode where multiple instances of Project Telescope can be deployed to a single OpenShift cluster.

Given that OpenShift emphasizes multitenant paradigms, this guide will describe the process of leveraging Project Telescope in a multitenant fashion.

### Set Required Variables

Before deploying Project Telescope, several environment variables will need to be defined as they play a role in the configuration.

First, set an environment variable called `APPS_SUBDOMAIN` by executing the following command:

```shell
export APPS_SUBDOMAIN=apps.$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
```

Next, to support the multitenant model, specify a unique instance name (such as your username). In this guide, `myuser` will be the name specfied. Set the environment variable `INSTANCE_NAME` to the desired value:

```shell
export INSTANCE_NAME=myuser
```

### Deploying Project Telescope Using Argo CD

As previously indicated, Project Telescope makes use of a series of Helm charts to streamline how Project Telescope is deployed and configured. These charts are located in the [helm-charts](https://github.com/RH-Telescope/helm-charts) repository within the [RH-Telescope](https://github.com/RH-Telescope) GitHub organization.

Charts are available for each of the primary Project Telescope components (frontend and backand). An additional chart is also available to support a deployment of the entire stack using Argo CD.

The `telescope-argocd` chart leverages an Argo CD [App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping) which in the process deploys the following charts:

* Frontend
* Backend
* PostgreSQL database supporting the backend

Bootstrapping Project Telescope can be completed with one simple command to define an Argo CD `Application` to deploy the Project Telescope Argo CD Helm Chart to a new namespace called `telescope-${INSTANCE_NAME}` 

Ensure the previously defined variables are still present and execute the following command to create the Argo CD _Application_ to deploy Project Telescope:

```shell
cat <<EOF | envsubst | oc apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  name: telescope-$INSTANCE_NAME
  namespace: openshift-gitops
spec:
  destination:
    namespace: telescope-$INSTANCE_NAME
    server: 'https://kubernetes.default.svc'
  project: default
  source:
    chart: telescope-argocd
    helm:
      values: |
        applicationPrefix: $INSTANCE_NAME
        charts:
          telescope-frontend:
            values:
              backendUrl: https://telescope-backend-telescope-$INSTANCE_NAME.$APPS_SUBDOMAIN
    repoURL: 'https://rh-telescope.github.io/helm-charts'
    targetRevision: x
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

### Verifying the Deployment

There are multiple methods for which the deployment of Project Telescope can be verified.

#### Using the OpenShift CLI

First, use the OpenShift Command Line to verify each of the `Applications` have been created and are healthy.

```shell
oc get applications -n openshift-gitops
```

The result should display four (4) total applications, each with a _Health Status_ of _Healthy_ as shown below.

```shell
NAME                        SYNC STATUS   HEALTH STATUS
myuser-postgresql           Synced        Healthy
myuser-telescope-backend    Synced        Healthy
myuser-telescope-frontend   Synced        Healthy
telescope-myuser            Synced        Healthy
```

#### Using the Argo CD Web Console

THe Argo CD _Applications_ that were deployed can be visualized using the Argo CD Web Console.

Obtain the address of the console by executing the following command:

```shell
echo https://$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}')
```

Navigate to the URL displayed.

Authenticate to Argo CD by selecting the **Log In Via OpenShift** button. 

Once authenticated, you will be presented with a dashboard of the deployed Argo CD applications which will appear similar to the following:

![Project Telescope Argo CD Applications](images/getting-started/argocd-applications.png)

Take note that all _Applications_ are Synchronized and Healthy.

#### Using the Argo CD CLI

Alternately, instead of using the Argo CD Web Console to visualize the state of the Project Telescope _Applications_, the Argo CD CLI installed previously can be used.

Login to the Argo CD CLI using the following command:

```shell
argocd login --insecure --grpc-web $(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}') --sso
```

A web browser will be launched so that the authentication process can be compelted. Enter your OpenShift credentials to authenticate the Argo CD CLI. Close the browser upon completion of the authentication process.

Use the Argo CD CLI to view the list of list of applications which will present a similar output as the OpenShift CLI, but be enriched with additional properties as shown below:

```shell
argocd app list
```

```shell
myuser-postgresql          https://kubernetes.default.svc  telescope-myuser  default  Synced  Healthy  Auto        <none>      https://rh-telescope.github.io/helm-charts        x
myuser-telescope-backend   https://kubernetes.default.svc  telescope-myuser  default  Synced  Healthy  Auto        <none>      https://rh-telescope.github.io/helm-charts        x
myuser-telescope-frontend  https://kubernetes.default.svc  telescope-myuser  default  Synced  Healthy  Auto        <none>      https://rh-telescope.github.io/helm-charts        x
telescope-myuser           https://kubernetes.default.svc  telescope-myuser  default  Synced  Healthy  Auto        <none>      https://rh-telescope.github.io/helm-charts        x
```

## Accessing the Project Telescope Frontend

With the verification process complete, access the Project Telescope frontend in a web browser. The location can be found by executing the following command:

```shell
echo https://$(oc get route telescope-frontend -n telescope-$INSTANCE_NAME -o jsonpath='{.spec.host}')
```

Explore the current status of the security posture of your environment:

![Project Telescope Frontend](images/getting-started/project-telescope-frontend.png)

Congratulations! You have completed deploying Project Telescope to your OpenShift environment using tool including Helm and OpenShift GitOps!

Additional guides coming soon!
