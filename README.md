# estafette-gke-preemptible-killer

This small Kubernetes application loop through a given preemptibles node pool and kill a node before the regular [24h
life time of a preemptible VM](https://cloud.google.com/compute/docs/instances/preemptible#limitations).

[![License](https://img.shields.io/github/license/estafette/estafette-gke-preemptible-killer.svg)](https://github.com/estafette/estafette-gke-preemptible-killer/blob/master/LICENSE)


## Why?

When creating a cluster, all the node are created at the same time and should be deleted after 24h of activity. To
prevent large disruption, the estafette-gke-preemptible-killer can be used to kill instances during a random period
of time between 12 and 24h. It makes use of the node annotation to store the time to kill value.


## How does that work

At a given interval, the application get the list of preemptible nodes and check weither the node should be
deleted or not. If the annotation doesn't exist, a time to kill value is added to the node annotation with a
random range between 12h and 24h based on the node creation time stamp.
When the time to kill time is passed, the Kubernetes node is marked as unschedulable, drained and the instance
deleted on GCloud.


## Usage

You can either use environment variables or flags to configure the following settings:

| Environment variable   | Flag                     | Default  | Description
| ---------------------- | ------------------------ | -------- | -----------------------------------------------------------------
| DRAIN_TIMEOUT          | --drain-timeout          | 300      | Max time in second to wait before deleting a node
| INTERVAL               | --interval (-i)          | 600      | Time in second to wait between each node check
| KUBECONFIG             | --kubeconfig             |          | Provide the path to the kube config path, usually located in ~/.kube/config. This argument is only needed if you're running the killer outside of your k8s cluster
| METRICS_LISTEN_ADDRESS | --metrics-listen-address | :9001    | The address to listen on for Prometheus metrics requests
| METRICS_PATH           | --metrics-path           | /metrics | The path to listen for Prometheus metrics requests

### Create a Google Service Account

In order to have the estafette-gke-preemptible-killer instance delete nodes,
create a service account and give the _compute.instances.delete_ permissions.

You can either create the service account and associate the role using the
GCloud web console or the cli:

```bash
$ export project_id=<PROJECT>
$ gcloud iam --project=$project_id service-accounts create preemptible-killer \
    --display-name preemptible-killer
$ gcloud iam --project=$project_id roles create preemptible_killer \
    --project $project_id \
    --title preemptible-killer \
    --description "Delete compute instances" \
    --permissions compute.instances.delete
$ export service_account_email=$(gcloud iam --project=$project_id service-accounts list --filter preemptible-killer --format 'value([email])')
$ gcloud projects add-iam-policy-binding $project_id \
    --member=serviceAccount:${service_account_email} \
    --role=projects/${project_id}/roles/preemptible_killer
$ gcloud iam --project=$project_id service-accounts keys create \
    --iam-account $service_account_email \
    google_service_account.json
```

### Deploy with Helm

```bash
# Prepare Helm/Tiller
$ kubectl create sa tiller -n kube-system
$ helm init --service-account tiller
$ kubectl create clusterrolebinding tiller \
    --clusterrole=cluster-admin \
    --serviceaccount=kube-system:tiller

# Install
$ helm upgrade estafette-gke-preemptible-killer \
    --namespace estafette \
    --install \
    --set rbac.create=true \
    --set-file googleServiceAccount=./google_service_account.json \
    ./chart/estafette-gke-preemptible-killer
```

### Deploy without Helm

```bash
export NAMESPACE=estafette
export APP_NAME=estafette-gke-preemptible-killer
export TEAM_NAME=tooling
export VERSION=1.0.35
export GO_PIPELINE_LABEL=1.0.35
export GOOGLE_SERVICE_ACCOUNT=$(cat google_service_account.json | base64)
export DRAIN_TIMEOUT=300
export INTERVAL=600
export CPU_REQUEST=10m
export MEMORY_REQUEST=16Mi
export CPU_LIMIT=50m
export MEMORY_LIMIT=128Mi

# Setup RBAC
curl https://raw.githubusercontent.com/estafette/estafette-gke-preemptible-killer/master/rbac.yaml | envsubst | kubectl apply -n ${NAMESPACE} -f -

# Run application
curl https://raw.githubusercontent.com/estafette/estafette-gke-preemptible-killer/master/kubernetes.yaml | envsubst | kubectl apply -n ${NAMESPACE} -f -
```

### Local development

```bash
# proxy master
kubectl proxy

# in another shell
go build && ./estafette-gke-preemptible-killer -i 10
```

Note: `KUBECONFIG=~/.kube/config` as environment variable can also be used if you don't want to use the `kubectl proxy`
command.
