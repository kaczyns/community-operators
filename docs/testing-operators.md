# Testing your Operator with Operator Framework

These instructions walk you through how to test if your Operator deploys correctly with Operator Framework.

The process below assume that you have an Kubernetes Operator in the Operator Framework *bundle* format, for example:

```
$ ls my-operator/
my-operator.v1.0.0.clusterserviceversion.yaml
my-operator-crd1.crd.yaml
my-operator-crd2.crd.yaml
my-operator.package.yaml
```

where *my-operator* is the name of your Operator. If you don't have this format yet, refer to our [README](https://github.com/operator-framework/community-operators/blob/master/README.md). We will refer to this example of `my-operator` in the following instructions.

# Table of Contents

[Pre-Requisites](#pre-requisites)
* [Kubernetes Cluster](#kubernetes-cluster)
* [Repositories](#repositories)
* [Tools](#tools)
    * [operator-courier](#operator-courier)
    * [Quay Login](#quay-login)
* [Linting](#linting)
* [Push to Quay.io](#push-to-quayio)

[Testing on Kubernetes](#testing-operator-deployment-on-kubernetes)

[Testing on OpenShift](#testing-operator-deployment-on-openshift)

[Testing with `scorecard`](#testing-with-scorecard)

[Additional Ressources](#additional-resources)

## Pre-Requisites

### Kubernetes cluster

For "upstream-community" operators targeting Kubernetes and [OperatorHub.io](https://operatorhub.io):
* A running Kubernetes cluster; [minikube](https://kubernetes.io/docs/setup/minikube/) is the simplest approach

For "community" operators targeting OCP/OKD and OperatorHub on OpenShift:
* either a running Kubernetes cluster; [minikube](https://kubernetes.io/docs/setup/minikube/) is the simplest approach
* or access to a running OpenShift 4 cluster, use [try.openshift.com](https://try.openshift.com/) to get a cluster on an AWS environment within ~30 mins

### Repositories

The following repositories are used throughout the process and should be cloned locally:

* [operator-marketplace](https://github.com/operator-framework/operator-marketplace)
* [operator-courier](https://github.com/operator-framework/operator-courier)
* [operator-lifecycle-manager](https://github.com/operator-framework/operator-lifecycle-manager)

For simplicity, the following commands will clone all of the repositories above:

```
git clone https://github.com/operator-framework/operator-marketplace.git
git clone https://github.com/operator-framework/operator-courier.git
git clone https://github.com/operator-framework/operator-lifecycle-manager.git
```

Before you begin your current working dir should look like the following, with `my-operator` as an example for the name of your bundle:

```
my-operator
operator-marketplace
operator-courier
operator-lifecycle-manager
```

### Tools

#### operator-courier

`operator-courier` is used for metadata syntax checking and validation. This can be installed directly from `pip`:

```
pip3 install operator-courier
```

#### Quay Login

In order to test the Operator installation flow, store your Operator bundle on [quay.io](https://quay.io). You can easily create an account and use the free tier (public repositories only). To upload your Operator to quay.io a token is needed. This only needs to be done once and can be saved locally. The `operator-courier` repository has a script to retrieve the token:

```
./operator-courier/scripts/get-quay-token

Username: johndoe
Password: 
{"token": "basic abcdefghijkl=="}
```

A token takes the following form and should be saved in an environment variable:

```
export QUAY_TOKEN="basic abcdefghijkl=="
```

### Linting

`operator-courier` will verify the fields included in the Operator metadata (CSV). The fields can also be manually reviewed according to [the operator CSV documentation](https://github.com/operator-framework/community-operators/blob/master/docs/required-fields.md).

The following command will run `operator-courier` against the bundle directory `my-operator/` from the example above.

```
operator-courier verify --ui_validate_io my-operator/
```

If there is no output, the bundle passed `operator-courier` validation. If there are errors, your bundle will not work. If there are warnings we still encourage you to fix them before proceeding to the next step.

### Push to quay.io

The Operator metadata in its bundle format will be uploaded into your namespace in [quay.io](http://quay.io).

The value for `PACKAGE_NAME` **must** be the same as in the operator's `*package.yaml` file and the operator bundle directory name. Assuming it is `my-operator`, this can be found by running `cat my-operator/*.package.yaml`.

The `PACKAGE_VERSION` is entirely up for you to decide. The version is independent of the Operator version since your bundle will contain all versions of your Operator metadata files. If you already uploaded your bundle to Quay.io at an earlier point, make sure to increment the version.

```
OPERATOR_DIR=my-operator/
QUAY_NAMESPACE=johndoe
PACKAGE_NAME=my-operator
PACKAGE_VERSION=1.0.0
TOKEN=$QUAY_TOKEN

operator-courier push "$OPERATOR_DIR" "$QUAY_NAMESPACE" "$PACKAGE_NAME" "$PACKAGE_VERSION" "$TOKEN"
```

Once that has completed, you should see it listed in your account's [Applications](https://quay.io/application/) tab.

> If the application has a lock icon, click through to the application and its Settings tab and select to make the application public.

Your Operator bundle is now ready for testing. To upload subsequent versions, bump semver string in the `PACKAGE_VERSION` variable, as `operator-marketplace` always downloads the newest bundle according to [semantic versioning](https://semver.org/).

## Testing Operator Deployment on Kubernetes

Please ensure you have fulfilled the [pre-requisites](#pre-requisites) before continuing with the instructions below.

### 1. Get a Kubernetes cluster

Start a Kubernetes `minikube` cluster:

```
minikube start
```

### 2. Install OLM

Install OLM into the cluster in the `olm` namespace:

```
kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.10.0/crds.yaml
kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.10.0/olm.yaml
```

### 3. Install the Operator Marketplace

Install Operator Marketplace into the cluster in the `marketplace` namespace:

```
kubectl apply -f operator-marketplace/deploy/upstream/
```

### 4. Create the OperatorSource

An `OperatorSource` object is used to define the external datastore we are using to store operator bundles. More information including example can be found in the documentation included in the `operator-marketplace` [repository](https://github.com/operator-framework/operator-marketplace#operatorsource).

**Replace** `johndoe` in `metadata.name` and `spec.registryNamespace` with your quay.io username in the example below and save it to a file called `operator-source.yaml`.

```
apiVersion: operators.coreos.com/v1
kind: OperatorSource
metadata:
  name: johndoe-operators
  namespace: marketplace
spec:
  type: appregistry
  endpoint: https://quay.io/cnr
  registryNamespace: johndoe
```

Now add the source to the cluster:

```
kubectl apply -f operator-source.yaml
```

The `operator-marketplace` controller should successfully process this object:

```
kubectl get operatorsource johndoe-operators -n marketplace

NAME                TYPE          ENDPOINT              REGISTRY   DISPLAYNAME  PUBLISHER   STATUS      MESSAGE                                       AGE
johndoe-operators   appregistry   https://quay.io/cnr   johndoe                 Succeeded   The object has been successfully reconciled   30s
```

### 5. View Available Operators

Once the `OperatorSource` is deployed, the following command can be used to list the available operators (until an operator is pushed into quay, this list will be empty):

> The command below assumes `johndoe-operators` as the name of the `OperatorSource` object. Adjust accordingly.

```
kubectl get opsrc johndoe-operators -o=custom-columns=NAME:.metadata.name,PACKAGES:.status.packages -n marketplace

NAME                PACKAGES
johndoe-operators   my-operator
```

### 6. Create CatalogSourceConfig

Once the OperatorSource has been added, a `CatalogSourceConfig` needs to be created in the `marketplace` namespace to make those Operators available on cluster.

Create the following file as `catalog-source-config.yaml`:

```
apiVersion: operators.coreos.com/v1
kind: CatalogSourceConfig
metadata:
  name: johndoe-operators
  namespace: marketplace
spec:
  targetNamespace: olm
  packages: my-operator
```

In the above example:

* `olm` is a namespace that OLM is watching for `CatalogSource` objects
* `packages` is a comma-separated list of operators that have been pushed to quay.io and should be deployable by this source.

> The file above assumes `my-operator` as the name of the operator bundle. Adjust accordingly.

Deploy the `CatalogSourceConfig` resource:

```
kubectl apply -f catalog-source-config.yaml
```

When this file is deployed, a `CatalogSourceConfig` resource is created in the `marketplace` namespace.

```
kubectl get catalogsourceconfig -n marketplace

NAME                      STATUS      MESSAGE                                       AGE
johndoe-operators         Succeeded   The object has been successfully reconciled   93s
```

Additionally, a `CatalogSource` is created in the namespace indicated in `spec.targetNamespace` (in the above example, `olm`):

```
kubectl get catalogsource -n olm

NAME                           NAME        TYPE   PUBLISHER   AGE
johndoe-operators              Custom      grpc   Custom      3m32s
[...]
```
### 7. Create an OperatorGroup

An `OperatorGroup` is used to denote which namespaces your Operator should be watching. It must in the namespace where your operator should be deployed, we'll use `default` in this example.

Its configuration depends on your Operator supporting watching its own namespace, a single namespace or all namespaces (as indicated by `spec.installModes` in the CSV).

Create the following file as  `operator-group.yaml` if your Operator supports watching its own or a single namespace.

If your Operator supports watching all namespaces you can omit the following step and place your `Subscription` (see next step) in the `operators` namespace instead.

```
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  name: my-operatorgroup
  namespace: default
spec:
  targetNamespaces:
  - default
```

Deploy the `OperatorGroup` resource:

```
kubectl apply -f operator-group.yaml
```

### 8. Create a Subscription

The last piece ties together all of the previous steps. A `Subscription` is created to the operator. Save the following to a file named: `operator-subscription.yaml`:

```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-operator-subsription
  namespace: default
spec:
  channel: <channel-name>
  name: my-operator
  source: johndoe-operators
  sourceNamespace: olm
```

If your Operator supports watching all namespaces, change the namespace of the Subscription from `default` to `operators`.

### 9. Verify Operator health

Watch your Operator being deployed by OLM from the catalog source created by Operator Marketplace with the following command:

```
kubectl get clusterserviceversion -n default

NAME                 DISPLAY       VERSION   REPLACES   PHASE
my-operator.v1.0.0   My Operator   1.0.0                Succeeded
```

> The above command assumes you have created the `Subscription` in the `default` namespace. Adjust accordingly if you have selected a different namespace.


If your Operator deployment (CSV) shows a `Succeeded` in the `InstallPhase` status, your Operator is deployed successfully. If that's not the case check the `ClusterServiceVersion` objects status for details.

Optional also check your Operator's deployment:

```
kubectl get deployment -n default
```

## Testing Operator Deployment on OpenShift

On OpenShift Container Platform and OKD 4.1 or newer `operator-marketplace` and `operator-lifeycle-manager` are already installed. You can start right away by creating an `OperatorSource` in the `openshift-marketplace` namespace as a user with the `cluster-admin` role. You will then use the UI to install your Operator. If you are interested what happens in the background, go through the [Testing on Kubernetes](#testing-operator-deployment-on-kubernetes) section above.

### 1. Create the OperatorSource

An `OperatorSource` object is used to define the external datastore we are using to store operator bundles. More information including example can be found in the documentation included in the `operator-marketplace` [repository](https://github.com/operator-framework/operator-marketplace#operatorsource).

**Replace** `johndoe` in `metadata.name` and `spec.registryNamespace` with your quay.io username in the example below and save it to a file called `operator-source.yaml`.

```
apiVersion: operators.coreos.com/v1
kind: OperatorSource
metadata:
  name: johndoe-operators
  namespace: openshift-marketplace
spec:
  type: appregistry
  endpoint: https://quay.io/cnr
  registryNamespace: johndoe
  displayName: "John Doe's Operators"
  publisher: "John Doe"
```

Create the object:

```
oc apply -f operator-source.yaml
```

Check if the `OperatorSource` was processed correctly:

```
oc get operatorsource johndoe-operators -n openshift-marketplace

NAME                TYPE          ENDPOINT              REGISTRY   DISPLAYNAME            PUBLISHER   STATUS      MESSAGE                                       AGE
johndoe-operators   appregistry   https://quay.io/cnr   johndoe    John Doe's Operators   John Doe    Succeeded   The object has been successfully reconciled   30s
```

### 2. Find your Operator in the OperatorHub UI

Go to your OpenShift UI and find your Operator by filtering for the *Custom* category:

![Find your Operator in OperatorHub](images/my-operator-in-hub.png)

### 3. Install your Operator from OperatorHub

To install your Operator simply click its icon and in the proceeding dialog click *Install*.

![Install your Operator from OperatorHub](images/my-operator-install.png)

 You will be asked where to install your Operator. Select either of the desired installation modes, if your Operator supports it and then click *Subscribe*

![Install your Operator from OperatorHub](images/my-operator-subscription.png)

You will be forwarded to the *Subscription Management* section of the OLM UI and after a couple of moments your Operator will be transitioning to *Installed*.

![Subscribe to your Operator from OperatorHub](images/my-operator-subscribed.png)

### 4. Verify Operator health

Change to the *Installed Operators* section in the left-hand navigation menu to verify your Operator's installation status:

![See your installed Operator](images/my-operator-installed.png)

It should have transitioned into the state *InstallationSucceeded*. You can now test it by starting to use its APIs.

## Testing with scorecard

If your Operator is up and running you can verify it is working as intended using its APIs. Additionally you can run [operator-sdk](https://github.com/operator-framework/operator-sdk/blob/master/doc/test-framework/scorecard.md)'s `scorecard` utility for validating against good practice and correctness of your Operator.

Assuming you are still in your top-level directory where `my-operator/` is your bundle location and an environment variable called `KUBECONFIG` points to a running `minikube` or OpenShift cluster with OLM present:

```
operator-sdk scorecard --olm-deployed --crds-dir my-operator/ --csv-path my-operator/my-operator.v1.0.0.clusterserviceversion.yaml
```

## Additional Resources

* [Cluster Service Version Spec](https://github.com/operator-framework/operator-lifecycle-manager/blob/master/Documentation/design/building-your-csv.md)
* [Example Bundle](https://github.com/operator-framework/community-operators/tree/master/upstream-community-operators/etcd)