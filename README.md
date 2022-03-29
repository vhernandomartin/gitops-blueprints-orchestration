# gitops-blueprints-orchestration
- [gitops-blueprints-orchestration](#gitops-blueprints-orchestration)
- [Disclaimer](#disclaimer)
- [Introduction](#introduction)
- [Pre-requisites](#pre-requisites)
  - [ArgoCD](#argocd)
  - [Helm](#helm)
    - [Chart.yaml file](#chartyaml-file)
    - [templates folder](#templates-folder)
    - [values.yaml file](#valuesyaml-file)
    - [Reference helm in ArgoCD app](#reference-helm-in-argocd-app)
- [Working with ArgoCD](#working-with-argocd)
  - [Deploy Gitops Operator - ArgoCD](#deploy-gitops-operator---argocd)
    - [Deploying Gitops ArgoCD in a automated way](#deploying-gitops-argocd-in-a-automated-way)
    - [Deploying Gitops ArgoCD manually](#deploying-gitops-argocd-manually)
  - [Download the argocd CLI](#download-the-argocd-cli)
  - [Configure kubeconfig](#configure-kubeconfig)
  - [Add a remote k8s/OpenShift cluster to ArgoCD](#add-a-remote-k8sopenshift-cluster-to-argocd)
  - [Deploy sample apps with ArgoCD](#deploy-sample-apps-with-argocd)
  - [Deploy Operators with ArgoCD](#deploy-operators-with-argocd)
    - [Deploying Local Storage Operator with ArgoCD](#deploying-local-storage-operator-with-argocd)
      - [CR definitions](#cr-definitions)
      - [Values file](#values-file)
      - [Deploy LSO app](#deploy-lso-app)
    - [Deploying ODF Operator with ArgoCD](#deploying-odf-operator-with-argocd)
      - [CR definitions](#cr-definitions-1)
      - [Values file](#values-file-1)
      - [Deploy ODF app](#deploy-odf-app)
  - [Deploy Operators in multiple clusters](#deploy-operators-in-multiple-clusters)
- [RHACM](#rhacm)
  - [Deploying RHACM](#deploying-rhacm)

# Disclaimer

> :warning:
> This tool is not supported by Red Hat, this has been prepared for testing, standarization and automation purposes within our internal labs.
> Use it at your own risk.

# Introduction

This document has been prepared to demonstrate and document how to deploy and configure the required artifacts (Operators, CRs, etc) on top of the OpenShift Clusters in a infrastructure.

OpenShift provides a tool called Red Hat Advanced Cluster Management (RHACM) that may complement and support this approach adding other really cool functionalities like deploying new OpenShift clusters using a Gitops approach.

# Pre-requisites
Whether we want use gitops to deploy Operators and applications or we want to deploy a RHACM environment to enable the ZTP pipeline, we'll need an OpenShift cluster that will serve as the HubCluster. There is no an official way to provide automation to deploy such HubCluster, in order to fill this gap, it has been prepared a wrapper script that makes use of the standalone Assisted Installer, this tool can be used either to deploy a HubCluster or other multi-node or SNO clusters.
To deploy cluster easily using this method please check this [repository](https://github.com/vhernandomartin/assisted-installer-howto#how-to-run-the-ocp-installer-wrapper-script).

## ArgoCD
In this document we will use ArgoCD to deploy and mantain operators and applications, for further references on what is ArgoCD and how to work with it please check the [official documentation](https://argo-cd.readthedocs.io/en/stable/).

## Helm
We make use of Helm Charts in a very easy way to help the developer with the customization application task, this way we can deploy the same application with the same CR definitions and apply different values depending the environment where we want to deploy the application/operator.
For further information on how to work with Helm please check the [official documentation](https://helm.sh/docs/topics/charts/).

The way we use Helm to deploy Operators in this environment is quite easy, just start replicating the following tree by yourself (templates is a folder).
```bash
.
|-- Chart.yaml
|-- templates
`-- values.yaml
```

### Chart.yaml file
This is the file that contains the definition of the Chart application, take the following template as an example, customize it with your own values.

```bash
$ cat Chart.yaml
apiVersion: v2
name: open-cluster-management
description: OpenShift RHACM operator Helm Chart
type: application
version: 1.0.0
appVersion: "1.0.0"
```

### templates folder
This path contains all the CR definitions that will be created in the OpenShift Cluster. Take into consideration that some of the values may reference to a variable that should be specified in the values.yaml file that will be explained later.
Since we are using ArgoCD to deploy the applications/operators it is important to set some annotations in each object definition, mainly "sync-options: SkipDryRunOnMissingResource" and "sync-wave annotations".
The first is very useful to avoid the synchronization task failing if it detects some CRD missing and not created yet, the second, the sync-wave annotation must be set in each CRD to indicate the order that ArgoCD must follow when creating the objects in the cluster.

This is an example on how the annotations should set in each CR.
```bash
metadata:
  annotations:
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
    argocd.argoproj.io/sync-wave: "3"
```

About the variables that you can set in your deployment, you can customize CRs with as many variables as you need, these vars must be defined in the values.yaml file.

This is an example, where the spec.channel value is taken from spec.channel in the values.yaml file.
```bash
$ cat subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  annotations:
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
    argocd.argoproj.io/sync-wave: "3"
  name: rhacm-subscription
  namespace: open-cluster-management
spec:
  channel: {{ .Values.spec.channel }}
  name: advanced-cluster-management
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

### values.yaml file
When using variables to customize your Helm applications you must define and fill this file with your custom values.
This is an example of how a values.yaml should look like:

```bash
$ cat values.yaml
spec:
  channel: release-2.4
  logLevel: debug
  clusterImageSet:
    name: openshift-v4.9.12
    releaseImage: registry.domain.example.com:5000/ocp4:4.9-x86_64
```

### Reference helm in ArgoCD app
Once the Helm application is well structured, defined and with all the CRs in place, we can create an ArgoCD application to control and maintain the full lifecycle of the application. In order to let the ArgoCD application which Helm values file needs to use, it is necessary to set the helm.valueFiles parameter accordingly.
This is just an example of how to include this parameter in the ArgoCD application definition:
```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rhacm-deploy
  namespace: openshift-gitops
spec:
  # The argocd project the application belongs to
  project: default
  destination:
    name: in-cluster
  # Source of the application manifests
  source:
    repoURL: https://github.com/vhernandomartin/gitops-blueprints-orchestration.git
    targetRevision: HEAD
    path: resources/operators/open-cluster-management
    helm:
      valueFiles:
        - values.yaml

... (output truncated)
```

# Working with ArgoCD

This procedure describes how to prepare the environment from the very beginning, from deploying ArgoCD with the openshift-gitops-operator bundle, to creating the ArgoCD applications to deploy these operators and the required configurations.

Before creating the applications is necessary to add any cluster deployed with Assisted Installer as an external remote cluster in ArgoCD, this way ArgoCD will gain control over these clusters and will be able to deploy, monitor and control applications.

## Deploy Gitops Operator - ArgoCD

### Deploying Gitops ArgoCD in a automated way

```bash
$ cd resources/sites/specific-sites
$ oc apply -k hubcluster-gitops/
subscription.operators.coreos.com/openshift-gitops-operator configured
```

```bash
$ oc get csv -n openshift-gitops
NAME                                DISPLAY                      VERSION   REPLACES                           PHASE
openshift-gitops-operator.v1.4.3    Red Hat OpenShift GitOps     1.4.3     openshift-gitops-operator.v1.4.2   Succeeded

$ oc get pods -n openshift-gitops
NAME                                                          READY   STATUS    RESTARTS   AGE
cluster-5784b87db7-7kmlw                                      1/1     Running   0          69s
kam-79b7d44d95-4tn54                                          1/1     Running   0          69s
openshift-gitops-application-controller-0                     1/1     Running   0          68s
openshift-gitops-applicationset-controller-5b684bf665-9ls26   1/1     Running   0          68s
openshift-gitops-dex-server-858f6b7944-lbg7v                  1/1     Running   0          68s
openshift-gitops-redis-7867d74fb4-vz6dq                       1/1     Running   0          68s
openshift-gitops-repo-server-788cf5b6b8-xd8pd                 1/1     Running   0          68s
openshift-gitops-server-84845dc8-fw8rs                        1/1     Running   0          68s
```

### Deploying Gitops ArgoCD manually

```bash
$ cat << EOF >
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/openshift-gitops-operator.openshift-operators: ""
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: openshift-gitops-operator
  source: redhat-operator-index
  startingCSV: openshift-gitops-operator.v1.3.2
  sourceNamespace: openshift-marketplace
  installPlanApproval: Manual
EOF

$ oc create -f 01_GitOps_subscription.yaml

$ oc get installplan
NAME            CSV                                APPROVAL   APPROVED
install-fx8wz   openshift-gitops-operator.v1.3.2   Manual     false

$ oc patch installplan install-fx8wz -n openshift-operators --type merge --patch '{"spec":{"approved":true}}'

$ oc get pods -o wide -n openshift-gitops
NAME                                                          READY   STATUS    RESTARTS   AGE   IP             NODE                            NOMINATED NODE   READINESS GATES
cluster-54b7b77995-kj5pc                                      1/1     Running   0          7s    10.128.0.206   cu-master1.domain.example.com   <none>           <none>
kam-76f5ff8585-87hrb                                          1/1     Running   0          7s    10.128.0.207   cu-master1.domain.example.com   <none>           <none>
openshift-gitops-application-controller-0                     0/1     Running   0          6s    10.128.0.212   cu-master1.domain.example.com   <none>           <none>
openshift-gitops-applicationset-controller-6948bcf87c-sm222   1/1     Running   0          6s    10.128.0.213   cu-master1.domain.example.com   <none>           <none>
openshift-gitops-dex-server-7987699f9b-94pjr                  1/1     Running   0          6s    10.128.0.214   cu-master1.domain.example.com   <none>           <none>
openshift-gitops-redis-7867d74fb4-ghscd                       1/1     Running   0          6s    10.128.0.209   cu-master1.domain.example.com   <none>           <none>
openshift-gitops-repo-server-6dc777c845-426br                 0/1     Running   0          6s    10.128.0.210   cu-master1.domain.example.com   <none>           <none>
openshift-gitops-server-7957cc47d9-xxt2j                      0/1     Running   0          6s    10.128.0.211   cu-master1.domain.example.com   <none>           <none>
```

## Download the argocd CLI

You can download the argocd CLI from the [ArgoCD official website](https://argo-cd.readthedocs.io/en/stable/cli_installation/#download-latest-version).

## Configure kubeconfig
In order to let ArgoCD connect and add other k8s/OpenShift clusters it is necessary to add contexts that identifies any k8s/OpenShift cluster, in a single kubeconfig file.


## Add a remote k8s/OpenShift cluster to ArgoCD

First of all get the ArgoCD admin password.
```bash
$ ARGOCD_PASSWD=$(oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)
```

Log in the ArgoCD instance.
```bash
$ argocd login openshift-gitops-server-openshift-gitops.apps.cu-compact-1.domain.example.com
```

Add the new remote cluster in ArgoCD.
```bash
$ argocd cluster add
ERRO[0000] Choose a context name from:
CURRENT  NAME                                                                            CLUSTER                                   SERVER
         /api-cu-compact-1-domain-example-com:6443/syseng                                api-cu-compact-1-domain-example-com:6443  https://api.cu-compact-1.domain.example.com:6443
         admin                                                                           cu-compact-1                              https://api.cu-compact-1.domain.example.com:6443
         api-du-sno-3-domain-example-com:6443/syseng                                     api-du-sno-3-domain-example-com:6443      https://api.du-sno-3.domain.example.com:6443
         argocd/api-cu-compact-1-domain-example-com:6443/system:admin                    api-cu-compact-1-domain-example-com:6443  https://api.cu-compact-1.domain.example.com:6443
         default/api-cu-compact-1-domain-example-com:6443/syseng                         api-cu-compact-1-domain-example-com:6443  https://api.cu-compact-1.domain.example.com:6443
         default/api-cu-compact-1-domain-example-com:6443/system:admin                   api-cu-compact-1-domain-example-com:6443  https://api.cu-compact-1.domain.example.com:6443
         kube-system/api-cu-compact-1-domain-example-com:6443/system:admin               api-cu-compact-1-domain-example-com:6443  https://api.cu-compact-1.domain.example.com:6443
         openshift-authentication/api-cu-compact-1-domain-example-com:6443/system:admin  api-cu-compact-1-domain-example-com:6443  https://api.cu-compact-1.domain.example.com:6443
         openshift-config/api-cu-compact-1-domain-example-com:6443/system:admin          api-cu-compact-1-domain-example-com:6443  https://api.cu-compact-1.domain.example.com:6443
         openshift-gitops/api-cu-compact-1-domain-example-com:6443/syseng                api-cu-compact-1-domain-example-com:6443  https://api.cu-compact-1.domain.example.com:6443
         openshift-gitops/api-cu-compact-1-domain-example-com:6443/system:admin          api-cu-compact-1-domain-example-com:6443  https://api.cu-compact-1.domain.example.com:6443
         openshift-marketplace/api-cu-compact-1-domain-example-com:6443/system:admin     api-cu-compact-1-domain-example-com:6443  https://api.cu-compact-1.domain.example.com:6443
*        openshift-operators/api-cu-compact-1-domain-example-com:6443/syseng             api-cu-compact-1-domain-example-com:6443  https://api.cu-compact-1.domain.example.com:6443
         openshift-operators/api-cu-compact-1-domain-example-com:6443/system:admin       api-cu-compact-1-domain-example-com:6443  https://api.cu-compact-1.domain.example.com:6443
```

```bash
$ argocd cluster add api-du-sno-3-domain-example-com:6443/syseng
WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `api-du-sno-3-domain-example-com:6443/syseng` with full cluster level admin privileges. Do you want to continue [y/N]? y
INFO[0001] ServiceAccount "argocd-manager" already exists in namespace "kube-system"
INFO[0001] ClusterRole "argocd-manager-role" updated
INFO[0001] ClusterRoleBinding "argocd-manager-role-binding" updated
Cluster 'https://api.du-sno-3.domain.example.com:6443' added
```

```bash
$ argocd cluster list
SERVER                                        NAME                                         VERSION  STATUS   MESSAGE                                              PROJECT
https://api.du-sno-3.domain.example.com:6443  api-du-sno-3-domain-example-com:6443/syseng           Unknown  Cluster has no application and not being monitored.
https://kubernetes.default.svc                in-cluster                                            Unknown  Cluster has no application and not being monitored.
```

## Deploy sample apps with ArgoCD

```bash
$ argocd app create sample-app --repo https://github.com/vhernandomartin/gitops-blueprints-orchestration.git --path resources/apps/sample-app --dest-server https://api.du-sno-3.domain.example.com:6443 --dest-namespace test-argocd
```

```bash
$ argocd app list
NAME        CLUSTER                                       NAMESPACE    PROJECT  STATUS     HEALTH   SYNCPOLICY  CONDITIONS  REPO                                                                                    PATH                       TARGET
sample-app  https://api.du-sno-3.domain.example.com:6443  test-argocd  default  OutOfSync  Missing  <none>      <none>      https://github.com/vhernandomartin/gitops-blueprints-orchestration.git  resources/apps/sample-app
```

```bash
$ argocd app sync sample-app
TIMESTAMP                  GROUP        KIND   NAMESPACE                   NAME    STATUS    HEALTH        HOOK  MESSAGE
2022-03-11T08:37:01-05:00            Service  test-argocd            sample-app  OutOfSync  Missing
2022-03-11T08:37:01-05:00   apps  Deployment  test-argocd            sample-app  OutOfSync  Missing
2022-03-11T08:37:01-05:00            Service  test-argocd            sample-app  OutOfSync  Missing              service/sample-app created
2022-03-11T08:37:01-05:00   apps  Deployment  test-argocd            sample-app  OutOfSync  Missing              deployment.apps/sample-app created
2022-03-11T08:37:01-05:00            Service  test-argocd            sample-app    Synced  Healthy                  service/sample-app created
2022-03-11T08:37:01-05:00   apps  Deployment  test-argocd            sample-app    Synced  Progressing              deployment.apps/sample-app created

Name:               sample-app
Project:            default
Server:             https://api.du-sno-3.domain.example.com:6443
Namespace:          test-argocd
URL:                https://openshift-gitops-server-openshift-gitops.apps.cu-compact-1.domain.example.com/applications/sample-app
Repo:               https://github.com/vhernandomartin/gitops-blueprints-orchestration.git
Target:
Path:               resources/apps/sample-app
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        Synced to  (47c1020)
Health Status:      Progressing

Operation:          Sync
Sync Revision:      47c1020dd5b1c5acfc91a6578329101751101db9
Phase:              Succeeded
Start:              2022-03-11 08:37:00 -0500 EST
Finished:           2022-03-11 08:37:00 -0500 EST
Duration:           0s
Message:            successfully synced (all tasks run)

GROUP  KIND        NAMESPACE    NAME        STATUS  HEALTH       HOOK  MESSAGE
       Service     test-argocd  sample-app  Synced  Healthy            service/sample-app created
apps   Deployment  test-argocd  sample-app  Synced  Progressing        deployment.apps/sample-app created
```

```bash
$ argocd app list
NAME        CLUSTER                                       NAMESPACE    PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                                                                    PATH                       TARGET
sample-app  https://api.du-sno-3.domain.example.com:6443  test-argocd  default  Synced  Healthy  <none>      <none>      https://github.com/vhernandomartin/gitops-blueprints-orchestration.git  resources/apps/sample-app
```

## Deploy Operators with ArgoCD

In this section we are going to describe how to deploy two operators with ArgoCD, here you will find a few examples, you can create your own deployments using ArgoCD and deploy them with based on your own preferences/requirements.

### Deploying Local Storage Operator with ArgoCD

You need to label the nodes first, adding the following tag:

```bash
for master_node in $(oc get nodes -l node-role.kubernetes.io/master -o name)
do
  oc label ${master_node} cluster.ocs.openshift.io/openshift-storage=""
done
```

#### CR definitions
Put the CRs definitions in your git repo, these are the required CRs, you will find these CRs in this repo in the path: resources/operators/openshift-local-storage

```bash
.
|-- Chart.yaml
|-- templates
|   |-- Localvolume.yaml
|   |-- clusterrolebinding.yaml
|   |-- namespace.yaml
|   |-- operatorgroup.yaml
|   `-- subscription.yaml
|-- values-cu.yaml
`-- values-du.yaml
```

#### Values file
With Helm Chart we can use variables to be replaced by real values in the yaml files placed in the templates folder. In this example there are two different values files (values-cu.yaml and values-du.yaml file), depending on the use case (CU or DU) it will be required to load values from one file or from the other. The helm vars file to be used is set in the ArgoCD Application or ApplicationSet file:
```bash
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: du-lso
  namespace: openshift-gitops
spec:
  generators:
  - list:
      # Parameters are generated based on this cluster list, to be substituted
      # into the template below.
      elements:
      - cluster: du-sno-3
        url: https://api.du-sno-3.domain.example.com:6443
      - cluster: du-sno-2
        url: https://api.du-sno-2.domain.example.com:6443
  template:
    # An Argo CD Application template, with support for parameter substitution
    # with values from parameters generated above.
    metadata:
      name: '{{cluster}}-du-lso'
    spec:
      project: default
      source:
        repoURL: https://github.com/vhernandomartin/gitops-blueprints-orchestration.git
        targetRevision: HEAD
        path: resources/operators/openshift-local-storage
        helm:
          valueFiles:
            - values-du.yaml
      destination:
        server: '{{url}}'
        namespace: openshift-local-storage
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

#### Deploy LSO app

In order to deploy the Local Storage Operator just run the following command to create the corresponding ArgoCD application. This application will be in charge of the whole application lifecycle, if any change is applied in the future ArgoCD will sync those changes automatically.

```bash
$ pwd
/home/kubeadmin/git/gitops-blueprints-orchestration/resources/sites/groups

$ oc apply -f du-lso.yaml
applicationset.argoproj.io/du-lso created
```

### Deploying ODF Operator with ArgoCD


#### CR definitions
Put the CRs definitions in your git repo, these are the required CRs, you will find these CRs in this repo in the path: resources/operators/openshift-data-foundation

```bash
.
|-- Chart.yaml
|-- templates
|   |-- clusterrolebinding.yaml
|   |-- namespace.yaml
|   |-- operatorgroup.yaml
|   |-- rook-ceph-operator-config.yaml
|   |-- storagecluster.yaml
|   |-- storagesystem.yaml
|   `-- subscription.yaml
`-- values-cu.yaml
```

#### Values file
With Helm Chart we can use variables to be replaced by real values in the yaml files placed in the templates folder. In this example there are two different values files (values-cu.yaml and values-du.yaml file), depending on the use case (CU or DU) it will be required to load values from one file or from the other. The helm vars file to be used is set in the ArgoCD Application or ApplicationSet file:
```bash
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cu-odf
  namespace: openshift-gitops
spec:
  generators:
  - list:
      # Parameters are generated based on this cluster list, to be substituted
      # into the template below.
      elements:
      - cluster: cu-compact-1
        url: https://kubernetes.default.svc
  template:
    # An Argo CD Application template, with support for parameter substitution
    # with values from parameters generated above.
    metadata:
      name: '{{cluster}}-cu-odf'
    spec:
      project: default
      source:
        repoURL: https://github.com/vhernandomartin/gitops-blueprints-orchestration.git
        targetRevision: HEAD
        path: resources/operators/openshift-data-foundation
        helm:
          valueFiles:
            - values-cu.yaml
      destination:
        server: '{{url}}'
        namespace: openshift-storage
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

#### Deploy ODF app

In order to deploy the OpenShift Data Foundation Operator and all the dependencies just run the following command to create the corresponding ArgoCD application. This application will be in charge of the whole application lifecycle, if any change is applied in the future ArgoCD will sync those changes automatically.

```bash
$ pwd
/home/kubeadmin/git/gitops-blueprints-orchestration/resources/sites/groups

$ oc apply -f cu-odf.yaml
applicationset.argoproj.io/cu-odf created
```

## Deploy Operators in multiple clusters

It is possible to deploy the same application / operator in multiple OpenShift clusters at the same time with the ArgoCD ApplicationSet CR.
The way to achieve that is creating an ApplicationSet object, in this object we indicate a list of servers, these servers are inserted in a variable that is automatically replaced in the template section, this way the ArgoCD Applications are created automatically pointing to the desired servers.

```bash
$ pwd
/home/kubeadmin/git/gitops-blueprints-orchestration/resources/sites/groups

$ cat du-lso.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: du-lso
  namespace: openshift-gitops
spec:
  generators:
  - list:
      # Parameters are generated based on this cluster list, to be substituted
      # into the template below.
      elements:
      - cluster: du-sno-3
        url: https://api.du-sno-3.domain.example.com:6443
      - cluster: du-sno-2
        url: https://api.du-sno-2.domain.example.com:6443
  template:
    # An Argo CD Application template, with support for parameter substitution
    # with values from parameters generated above.
    metadata:
      name: '{{cluster}}-du-lso'
    spec:
      project: default
      source:
        repoURL: https://github.com/vhernandomartin/gitops-blueprints-orchestration.git
        targetRevision: HEAD
        path: resources/operators/openshift-local-storage
        helm:
          valueFiles:
            - values-du.yaml
      destination:
        server: '{{url}}'
        namespace: openshift-local-storage
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

```bash
$ oc apply -f du-lso.yaml
applicationset.argoproj.io/du-lso created
```

```bash
$ argocd app list
NAME             CLUSTER                                       NAMESPACE                PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                                                                    PATH                                         TARGET
du-sno-2-du-lso  https://api.du-sno-2.domain.example.com:6443  openshift-local-storage  default  Synced  Healthy  Auto-Prune  <none>      https://github.com/vhernandomartin/gitops-blueprints-orchestration.git  resources/operators/openshift-local-storage  HEAD
du-sno-3-du-lso  https://api.du-sno-3.domain.example.com:6443  openshift-local-storage  default  Synced  Healthy  Auto-Prune  <none>      https://github.com/vhernandomartin/gitops-blueprints-orchestration.git  resources/operators/openshift-local-storage  HEAD
sample-app       https://api.du-sno-3.domain.example.com:6443  test-argocd              default  Synced  Healthy  Auto-Prune  <none>      https://github.com/vhernandomartin/gitops-blueprints-orchestration.git  resources/apps/sample-app

$ oc get applicationset
NAME     AGE
du-lso   15m

$ oc get applications
NAME              SYNC STATUS   HEALTH STATUS
du-sno-2-du-lso   Synced        Healthy
du-sno-3-du-lso   Synced        Healthy
sample-app        Synced        Healthy
```

# RHACM

Red Hat Advanced Cluster Management (RHACM) comes along with other features/services that helps the administrator to deploy, manage and monitor other OpenShift clusters in the infrastructure.
Once RHACM is deployed, and the multiclusterhub configured along with other components like Hive, AgentServiceConfig and the Assisted Service configuration (among others), the GitOps ZTP pipeline can be deployed, this will enable the administrator to deploy easily OpenShift clusters along the infrastructure, to know more about how to configure, enable and deploy clusters check this [repo](https://github.com/vhernandomartin/ztp-ran#ztp-ran).

## Deploying RHACM

RHACM and the base configuration required to be ready to deploy OpenShift clusters with ZTP can be deployed automatically using GitOps ArgoCD tool. Just run the following to create an ArgoCD application, ArgoCD will complete all the required steps for you.

```bash
$ cd resources/sites/specific-sites/hubcluster-rhacm
$ oc apply -f rhacm.yaml
application.argoproj.io/rhacm-deploy created
```