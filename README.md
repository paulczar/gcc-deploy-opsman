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

Get the credentials:

```
gcloud container clusters get-credentials platform-automation \
  --zone us-central1-c
```

## Helm etc

You should have helm installed, We're using helm 3 to avoid tiller.

## Deploy Google Config Connector (GCC)

Export some env vars:

```bash
export PROJECT_ID=example-project-id
export FQDN=example.com
```

## Configure GCP service account for Google Config Connector (GCC)

Make sure the IAM api is enabled - https://console.cloud.google.com/apis/library/iam.googleapis.com

```bash
gcloud iam service-accounts create cnrm-system

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member serviceAccount:cnrm-system@$PROJECT_ID.iam.gserviceaccount.com \
  --role roles/owner

gcloud iam service-accounts keys create --iam-account \
  cnrm-system@$PROJECT_ID.iam.gserviceaccount.com scratch/key.json

```

## Install GCC and legacy Operator

Set up cluster namespaces:

```bash
kubectl create namespace cnrm-system
kubectl create namespace tekton-install-pks
kubectl create namespace pgtm-pczarkowski
```

Create secret for GCC:

```bash
kubectl -n cnrm-system create secret generic gcp-key \
    --from-file scratch/key.json
```

download GCC:

```bash
cd scratch
curl -X GET -sLO \
-H "Authorization: Bearer $(gcloud auth print-access-token)" \
--location-trusted \
https://us-central1-cnrm-eap.cloudfunctions.net/download/latest/infra/install-bundle.tar.gz
```

Untar and apply:

```bash
tar xzvf install-bundle.tar.gz
k apply -n cnrm-system -f scratch/install-bundle/
cd ..
```


## Deploy GCP Operator

> Note: The original GCP Operator still has some resource support that is missing in GCC.  Soon we hope to remote the need for this.

```bash
kubectl apply -f charts/gcp-operator/crd

helm install gcp-operator ./charts/gcp-operator --namespace cnrm-system \
  --set watchNamespace=$PROJECT_ID

```

## Cert Manager

This will allow for the use of a real cert for opsman

> Note: if you don't have the stable repo in helm you may need to run `helm repo add stable https://kubernetes-charts.storage.googleapis.com`.

```
kubectl apply -f charts/cert-manager/manifests/crds.yaml
helm install cert-manager stable/cert-manager \
  --values charts/cert-manager/values.yaml \
  --namespace cnrm-system
```

## Deploy Ops Manager

1. edit `./charts/opsman/values.yaml` to change any defaults

2. Create manifests from helm and apply them:

```bash
helm install opsman ./charts/opsman --namespace $PROJECT_ID \
  --set "dns.zone=$FQDN" --set "projectID=$PROJECT_ID"

```

## Set up DNS for opsman

Get opsman ip address from gcloud and pass it to a helm upgrade (you should persist it in a values file)

```bash
PKS_IP=$(gcloud compute addresses describe opsman-pks-api --region us-central1 --format "value(address)")

OPSMAN_IP=$(kubectl -n $PROJECT_ID get instances -o custom-columns=":status.primaryExternalIP" --no-headers)

helm upgrade opsman ./charts/opsman --namespace $PROJECT_ID \
  --set "opsmanAddress=$OPSMAN_IP" --set "dns.zone=$FQDN" \
  --set "projectID=$PROJECT_ID" \
  --set "pksAddress=$PKS_IP"
```


## Deploy tekton and Run automation

```bash
kubectl apply -f manifests/tekton
```

create configmap / secret for opsman creds:

```
kubectl create ns tekton-install-pks

kubectl -n tekton-install-pks create secret \
    generic google-credentials \
    --from-file="scratch/key.json"

kubectl -n tekton-install-pks create configmap \
    opsman-auth \
    --from-literal="host=opsman.$FQDN" \
    --from-literal="user=admin"

kubectl -n tekton-install-pks create secret \
    generic opsman-auth \
    --from-literal="password=opsman-password" \
    --from-literal="decryptionPassword=lets-encrypt-this-thing"

kubectl -n tekton-install-pks create configmap \
    pks-config \
    --from-literal="dns=pks.$FQDN" \
    --from-literal="version=1.6.0" \
    --from-literal="username=pksadmin"

kubectl -n tekton-install-pks create secret \
    generic pks-config \
    --from-literal="password=pks-password"

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
$ kubectl -n tekton-install-pks get pipelineruns
NAME          SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
install-pks   Unknown     Running   30s
```

```bash
kubectl -n tekton-install-pks logs -l "tekton.dev/pipeline=install-pks" -f
```

## Connect to PKS and create a cluster

Make sure you have the PKS CLI installed and then run:

```bash
kubectl -n tekton-install-pks get secrets pks-config -o json | jq -r .data.password | base64 --decode
pks login -k -a pks.$FQDN -u "pksadmin"
```

Create a cluster:

```bash
pks create-cluster cluster1 --plan basic --external-hostname k8s.cluster1.$FQDN
```

Set DNS for that cluster:

> Note: you'll need to find the name of your zone via `gcloud dns managed-zones list`.

```bash
ZONE=opsman
CLUSTER_NAME=workshop
CLUSTER_DNS=$(pks cluster $CLUSTER_NAME | grep "Master Host" | awk '{print $4}')
UUID=$(pks cluster $CLUSTER_NAME | grep "UUID" | awk '{print $2}')
EXTERNAL_IP=$(gcloud compute addresses list | grep pks-$UUID | awk '{print $3}')

gcloud dns record-sets transaction start --zone=$ZONE \
  --transaction-file=scratch/dns.yaml
gcloud dns record-sets transaction add "$EXTERNAL_IP" \
  --name=$CLUSTER_DNS. --type=A --zone=$ZONE --ttl=14400 \
  --transaction-file=scratch/dns.yaml
gcloud dns record-sets transaction execute --zone=$ZONE \
  --transaction-file=scratch/dns.yaml
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