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
  --role roles/owner --name google-config-connector

gcloud iam service-accounts keys create --iam-account \
  cnrm-system@$PROJECT_ID.iam.gserviceaccount.com key.json

```

## Configure GCP service account for PKS

> working on not needing to do this, but for now :shrug:

```bash
gcloud iam service-accounts create pks-master

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member serviceAccount:opsman-pks-master@$PROJECT_ID.iam.gserviceaccount.com \
  --name opsman-pks-master \
  --role roles/compute.networkAdmin \
  --role roles/compute.instanceAdmin \
  --role roles/compute.storageAdmin \
  --role roles/compute.securityAdmin \
  --role roles/iam.serviceAccountActor

gcloud iam service-accounts create pks-worker

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member serviceAccount:opsman-pks-worker@$PROJECT_ID.iam.gserviceaccount.com \
  --name opsman-pks-master \
  --role roles/compute.viewer \

```

Install GCC to GKE Cluster:

```bash
kubectl create namespace cnrm-system
kubectl create namespace tekton-install-pks
kubectl create namespace pgtm-pczarkowski

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

```
kubectl apply -f charts/cert-manager/manifests/crds.yaml
helm install cert-manager stable/cert-manager \
  --values charts/cert-manager/values.yaml \
  --namespace cnrm-system
```

## Deploy Ops Manager

1. edit `./opsman/values.yaml` to change any defaults

2. Create manifests from helm and apply them:

```bash
helm install opsman ./charts/opsman --namespace $PROJECT_ID \
  --set "dns.zone=$FQDN" --set "projectID=$PROJECT_ID"

```

## Set up DNS for opsman

Get opsman ip address from gcloud and pass it to a helm upgrade (you should persist it in a values file)

```bash
PKS_IP=$(gcloud compute addresses describe opsman-pks-api --region us-central1 --format "value(address)")

OPSMAN_IP=$()

helm upgrade opsman ./charts/opsman --namespace $PROJECT_ID \
  --set "opsmanAddress=$OPSMAN_IP" --set "dns.zone=$FQDN" \
  --set "projectID=$PROJECT_ID" \
  --set "pksAddress=$PKS_IP"
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
    --from-literal="version=1.6.0" \
    --from-literal="username=pksadmin"

kubectl -n tekton-install-pks create secret \
    generic pks-config \
    --from-literal="password=XXXXX"

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
kubectl -n tekton-install-pks apply -f tekton-pipeline/pipeline-run/install-pks.yaml
```

Check on status of pipeline:

```bash
kubectl -n tekton-install-pks get pods
kubectl -n tekton-install-pks logs ...

```

## Misc Operations

### Update Opsman / PKS certs

Unfortunately the Opsman cert is not fully managed, cert-manager in kubernetes will update the cert itself, but it doesn't auto apply to cert manager. Here's how to upgrade it:

```bash
kubectl -n tekton-install-pks apply -f tekton-pipeline/task-run/update-opsman-cert.yaml
```

The same can be said for the PKS certificate, Doing a re-configure of PKS will have it pick up the new certificate. This can take some time if it decides to run the "upgrade all clusters" errand, however it shouldn't cause downtime:

```bash
kubectl -n tekton-install-pks apply -f tekton-pipeline/task-run/configure-pks.yaml
```

### Upgrade PKS

This should be relatively straight forward between versions that don't change too much. Since we created a `configmap` that contains our version, we can edit that configmap, change the version and run an update:

> Note: A release with breaking changes to the pks properties structure may require updating the helm chart first.

```bash
kubectl -n tekton-install-pks edit configmap pks-config
```

```bash
kubectl -n tekton-install-pks apply -f tekton-pipeline/task-run/configure-pks.yaml
```