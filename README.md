# New and fancy GCP lab for PKS

> This repo is a working proof of concept. There may be some hard coded things in here that I haven't found (or am too lazy to fix) ... so don't just follow the instructions blindly.  You can always `grep -r czar` and that would probably find most hardcoded bits.

This describes using a base GKE cluster (yeah weird right) to spin up the infrastructure required to install PKS.  It uses a combination of [Google Config Connector](https://cloud.google.com/config-connector/docs/reference/resources) and my own deprecated [GCP Operator](https://github.com/paulczar/gcp-cloud-compute-operator).

> Note: The use of GCP Operator will disappear over time as GCC becomes more fully featured.

## Preperation

You need a GKE cluster... you can use the following to get a default cluster:

```
gcloud container clusters create platform-automation \
  --zone us-central1-c
```

## Helm etc

You should have helm installed, We're using helm 3 to avoid tiller.

## Deploy Google Config Connector (GCC)

Export some env vars:

```bash
export PROJECT_ID=pgtm-pczarkowski
export FQDN=example.com
```

## Configure GCP service account for Google Config Connector (GCC)

```bash
gcloud iam service-accounts create cnrm-system

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member serviceAccount:cnrm-system@$PROJECT_ID.iam.gserviceaccount.com \
  --role roles/owner

gcloud iam service-accounts keys create --iam-account \
  cnrm-system@$PROJECT_ID.iam.gserviceaccount.com key.json

```

Install GCC to GKE Cluster:

```bash
kubectl create namespace cnrm-system tekton-install-pks
kubectl -n cnrm-system create secret generic gcp-key \
    --from-file key.json
```

install GCC:

```bash
k apply -f charts/gcc/crd
helm install gcc ./charts/gcc --namespace cnrm-system
```


## Deploy GCP Operator

> Note: The original GCP Operator still has some resource support that is missing in GCC.  Soon we hope to remote the need for this.

> Note: before running this modify the PROJECT ID in `gcp-operator/operator.yaml`

```bash
kubectl apply -f charts/gcp-operator/crd

helm install gcp-operator ./charts/gcp-operator --namespace cnrm-system \
  --set watchNamespace=$PROJECT_ID

```

## Cert Manager

This will allow for the use of a real cert for opsman

> Note you'll need to edit `cluster-issuer.yaml` to suit your environment, I should add it into the helm chart.

```
kubectl apply -f charts/cert-manager/manifests/crds.yaml
helm install cert-manager stable/cert-manager \
  --values charts/cert-manager/values.yaml \
  --namespace cnrm-system

kubectl apply -f charts/cert-manager/manifests/cluster-issuer.yaml

```

```

## Deploy Ops Manager

1. edit `./opsman/values.yaml` to change any defaults

2. Create manifests from helm and apply them:

```bash
helm install opsman ./charts/opsman --namespace $PROJECT_ID \
  --set "dns.zone=$FQDN" --set "opsman.config.projectID=$PROJECT_ID"

```

## Set up DNS for opsman

Get opsman ip address from gcloud and pass it to a helm upgrade (you should persist it in a values file)

```bash
IP=$(gcloud compute addresses describe opsman-pks-api --region us-central1 --format "value(address)")

helm upgrade opsman ./charts/opsman --namespace $PROJECT_ID \
  --set "opsmanIP=$IP" --set "dns.zone=$FQDN" \
  --set "opsman.config.projectID=$PROJECT_ID"
```


## Deploy tekton

```bash
kubectl apply -f manifests/tekton
```

create configmap / secret for opsman creds:

```
kubectl create ns tekton-install-pks

kubectl -n tekton-install-pks create secret \
    generic google-credentials \
    --from-file="key.json"

kubectl -n tekton-install-pks create configmap \
    opsman-auth \
    --from-literal="host=opsman.$FQDN" \
    --from-literal="user=admin"

kubectl -n tekton-install-pks create secret \
    generic opsman-auth \
    --from-literal="password=this-is-a-bad-password" \
    --from-literal="decryptionPassword=lets-encrypt-this-thing"

kubectl -n tekton-install-pks create configmap \
    pks-config \
    --from-literal="dns=pks.$FQDN" \
    --from-literal="version=1.5.0"

kubectl -n tekton-install-pks create secret \
    generic pivnet-auth \
    --from-literal="token=XXXXXX"
```


Create the tasks and pipeline:

```bash
kubectl -n tekton-install-pks apply -f tekton-pipeline/task
kubectl -n tekton-install-pks apply -f tekton-pipeline/pipeline/install-pks.yaml
```

Run the pipeline:

```bash
kubectl -n tekton-install-pks apply -f tekton-pipeline/pipeline/pipeline-run.yaml
```

Check on status of pipeline:

```bash
kubectl -n tekton-install-pks get pods
kubectl -n tekton-install-pks logs ...

```
