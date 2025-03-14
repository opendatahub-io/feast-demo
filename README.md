# Feast Customer Demo

## Demo High Level Overview

In this demo, we'll walk through the 
[feast credit score local tutorial][tutorial-repo] to highlight use
of Feast for training a model and querying Feast at model
inference time.

We'll first walk through this demo in a workbench environment, showcasing
the data scientist user persona's experience when developing model
training code against a Feature Store.

In the second part of the demo, we'll deploy a Feature Store on OpenShift
using the Feast operator. This showcases the MLOps persona's user experience,
particularly the ability to provision a feature store from a predefined
feature repository in Git.

## Prerequisites

### OpenShift Client
You'll first need to [install the OpenShift oc client](https://docs.openshift.com/container-platform/4.18/cli_reference/openshift_cli/getting-started-cli.html#installing-openshift-cli) to be
able to run the command contained in this document.

### OpenShift AI Configuration
We'll run this demo from an OpenShift AI Workbench (Jupyter notebook). As such, we need to
[install OpenShift AI](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2-latest/html/installing_and_uninstalling_openshift_ai_self-managed/configuring-custom-namespaces#installing-the-openshift-data-science-operator_operator-install)
on the cluster and enable the dashboard and workbenches components.

You can apply the following DataScienceCluster custom resource to
enable just the required components of OpenShift AI:

```
cat <<EOF | oc apply -f -
apiVersion: datasciencecluster.opendatahub.io/v1
kind: DataScienceCluster
metadata:
  name: default-dsc
  namespace: redhat-ods-operator
  labels:
    app.kubernetes.io/name: datasciencecluster
    app.kubernetes.io/instance: default-dsc
    app.kubernetes.io/part-of: rhods-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: rhods-operator
spec:
  components:
    codeflare:
      managementState: Removed
    kserve:
      nim:
        managementState: Removed
      serving:
        ingressGateway:
          certificate:
            type: OpenshiftDefaultIngress
        managementState: Removed
        name: knative-serving
      managementState: Removed
    modelregistry:
      registriesNamespace: rhoai-model-registries
      managementState: Removed
    trustyai:
      managementState: Removed
    ray:
      managementState: Removed
    kueue:
      managementState: Removed
    workbenches:
      managementState: Managed
    dashboard:
      managementState: Managed
    modelmeshserving:
      managementState: Removed
    datasciencepipelines:
      managementState: Removed
    trainingoperator:
      managementState: Removed
EOF
```

For the purposes of the demo, we will also update the RHOAI dashboard
deployment to deploy only a single pod (the default is 5), therefore reducing the compute resource requirements:

```
oc patch deployment/rhods-dashboard -n redhat-ods-applications --patch='{"spec": {"replicas": 1}}'
```

### Project Creation

Next, we'll programmatically create an OpenShift AI project to house the demo workbench and feature store instance:

```
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    openshift.io/description: "Feast"
    openshift.io/display-name: "Feast"
  labels:
    kubernetes.io/metadata.name: feast
    opendatahub.io/dashboard: "true"
  name: feast
EOF
```

### Data Store Setup

For this demo, we'll use a Postgresql instance for Feast's registry server and Redis for
Feast's offline server. Run the following to create the Postgresql and Redis instances and
associated services and secrets:

```
oc apply -f https://raw.githubusercontent.com/feast-dev/feast/refs/heads/0.47-branch/examples/operator-quickstart/postgres.yaml
oc apply -f https://raw.githubusercontent.com/feast-dev/feast/refs/heads/0.47-branch/examples/operator-quickstart/redis.yaml
```

## Demo Phase 1 - Run the example in a workbench

First, within the `Feast` project that was created earlier,
[create a new workbench](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.17/html/working_on_data_science_projects/using-project-workbenches_projects#creating-a-project-workbench_projects) 
from the OpenShift AI Dashboard. When creating the workbench, select 
the `Standard Data Science` notebook image. The image version and container size can 
be left at their default values. No data connection needs to
be created, and you can use the default option of creating new cluster storage
for the workbench.

**Important** Be sure to add the following environment variables when creating the 
workbench, as they will be used to initialize the Feast SDK connection to the feature store endpoints. For this demo we'll add the environment variables as key/value pairs 
in a ConfigMap. In a production setup, at a minimum thepassword values should be 
pulled from a secret:

  * Key: `POSTGRES_HOST`, Value: `postgres.feast.svc.cluster.local`
  * Key: `POSTGRES_USER`, Value: `feast`
  * Key: `POSTGRES_PASSWORD`, Value `feast`
  * Key: `POSTGRES_DB`, Value: `feast`
  * Key: `REDIS_HOST`, Value: `redis.feast.svc.cluster.local`

Once the workbench has been started, open it in your brower by clicking on the link
from the OpenShift AI dashboard.

![open workbench screen](images/open-workbench.png)

From within the running workbench, [clone the example repository](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2-latest/html/working_in_your_data_science_ide/working_in_jupyterlab#collaborating-on-notebooks-by-using-git_ide)
by cloning the URL `https://github.com/accorvin/feast-credit-score-local-tutorial.git` then switching to the `demo`
branch (**Note** You may find it easier to change branches via the terminal in your notebook environment)

You should then see the contents of the Git repository in the file tree of your notebook environment:

![workbench after clone](images/workbench-after-clone.png)

Finally, perform the following to step through the demo:
  1. Install the python requirements
  2. Initialize the feature store by applying the feature definitions by running
     a `feast apply` command
  3. Populate the Feast online store by running a `feast materialize` command
  4. Demonstrate the functionality by running the `run.py` script

The easiest way to do all of this this is by stepping through the
 `demo_walkthrough.ipynb` notebook that is included in the repository. Alternatively,
 the necessary commands are provided below:

```
pip install --upgrade pip && pip install -r requirements.txt
feast -c $HOME/feast-credit-score-local-tutorial/feature_repo apply
feast -c $HOME/feast-credit-score-local-tutorial/feature_repo materialize 1990-01-01T00:00:00 $(date -u +"%Y-%m-%dT%H:%M:%S")
python $HOME/feast-credit-score-local-tutorial/run.py
```

## Demo Phase 2 - Deploy a Feature Store via the Feast Operator

In this second phase of the demo, we will use the Feast Operator to deploy
a feature store on OpenShift and initialize the feature store with the same
feature repository used in the previous example.

### Feast Operator Installation

Normally we will install the Feast operator via the ODH/RHOAI operator. For the
purposes of this demo, however, we want to use an unreleased version of the 
operator (to be able to use the new feast Git init capabnility). We therefore
need to run this command to install a specific version of the Feast operator:

```
oc apply -f https://raw.githubusercontent.com/feast-dev/feast/refs/heads/0.47-branch/infra/feast-operator/dist/install.yaml
```

### Feature Store Instance Creation

Next, we'll run the following command to create a Feature Store instance in the `feast` namespace.

```
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: feast-data-stores
  namespace: feast
stringData:
  redis: |
    connection_string: redis.feast.svc.cluster.local:6379
  sql: |
    path: postgresql+psycopg://\${POSTGRES_USER}:\${POSTGRES_PASSWORD}@postgres.feast.svc.cluster.local:5432/\${POSTGRES_DB}
    cache_ttl_seconds: 60
    sqlalchemy_config_kwargs:
        echo: false
        pool_pre_ping: true
---
apiVersion: feast.dev/v1alpha1
kind: FeatureStore
metadata:
  name: example
  namespace: feast
spec:
  feastProject: credit_scoring_local
  feastProjectDir:
    git:
      url: https://github.com/accorvin/feast-credit-score-local-tutorial.git
      ref: demo
  services:
    ui:
      envFrom:
      - secretRef:
          name: postgres-secret
    offlineStore:
      persistence:
        file:
          type: duckdb
    onlineStore:
      persistence:
        store:
          type: redis
          secretRef:
            name: feast-data-stores
      server:
        envFrom:
        - secretRef:
            name: postgres-secret
        env:
        - name: MPLCONFIGDIR
          value: /tmp
        resources:
          requests:
            cpu: 150m
            memory: 128Mi
    registry:
      local:
        persistence:
          store:
            type: sql
            secretRef:
              name: feast-data-stores
EOF
```

This will initialize a feature store instance backed by ephemeral storage
and also clone the [Feast credit score local tutorial repository][tutorial-repo]
into the Feature Store to facilitate easily applying the Feature Store object
definitionsrequired for the demo.

### (Optional) Apply the Feature Store object definitions

If you've been following along and already performed phase 1 of the demo,
then the feature store should already be initialized. If you want
to apply the feature store objects manually and materialize the data,
perform the following:

```
oc exec -n feast deploy/feast-example -itc online -- feast apply
```

You should see output similar to the following:

```{: .optional-language-as-class .no-copy}
 oc exec -n feast deploy/feast-example -itc online -- feast apply
No project found in the repository. Using project name credit_scoring_local defined in feature_store.yaml
Applying changes for project credit_scoring_local
Deploying infrastructure for zipcode_features
Deploying infrastructure for credit_history
```

You can view the definitions for the objects that were created
[here](https://github.com/accorvin/feast-credit-score-local-tutorial/blob/demo/feature_repo/features.py).

Finally, we can initialize the Feature Store's online store to ensure that it
has the latest values for all features availble for on-demand querying:

```
oc exec -n feast deploy/feast-example -itc online -- bash -c 'feast materialize 1990-01-01T00:00:00 $(date -u +"%Y-%m-%dT%H:%M:%S")'
```

You should see output similar to the following:

```{: .optional-language-as-class .no-copy}
% oc exec -n feast deploy/feast-example -itc online -- bash -c 'feast materialize 1990-01-01T00:00:00 $(date -u +"%Y-%m-%dT%H:%M:%S")'
Materializing 2 feature views from 1990-01-01 00:00:00+00:00 to 2025-03-14 16:23:32+00:00 into the redis online store.

zipcode_features:
100%|██████████████████████████████████████████████████████| 28844/28844 [00:01<00:00, 16583.22it/s]
credit_history:
100%|██████████████████████████████████████████████████████| 28633/28633 [00:02<00:00, 13212.07it/s]
```

### Accessing the Feast UI

Finally, you can access the Feature Store UI to explore the data sources,
entities, and feature views that were created as part of the demo.

Get the URL for the UI by running the following command:

```
oc -n feast get route feast-example-ui --template={{.spec.host}}
```

Navigate in a browser to the resultant URL. You should see a page similar
to the following:

![feast UI](images/feast-ui.png)

[tutorial-repo]: https://github.com/feast-dev/feast-credit-score-local-tutorial