# New and fancy GCP lab for PKS

This describes using a base GKE cluster (yeah weird right) to spin up the infrastructure required to install PKS.  It uses a combination of [Google Config Connector](https://cloud.google.com/config-connector/docs/reference/resources) and my own deprecated [GCP Operator](https://github.com/paulczar/gcp-cloud-compute-operator).

> Note: The use of GCP Operator will disappear over time as GCC becomes more fully featured.

## Preperation

First you need a GKE cluster (the smallest one possible is fine) and a CloudSQL postgres cluster.

Use the GCP gui to get those started.

## Helm etc

You should have helm installed, We use `helm template` to create our manifests.

## Deploy Google Config Connector (GCC)

Export some env vars:

```bash
export PROJECT_ID=pgtm-pczarkowski
```

## Configure GCP service account for GCC

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
kubectl create namespace cnrm-system
kubectl create secret generic gcp-key --from-file key.json --namespace cnrm-system
```

Download and install GCC:

```bash
curl -X GET -sLS \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  --location-trusted \
  https://us-central1-cnrm-eap.cloudfunctions.net/download/latest/infra/install-bundle.tar.gz | tar xzvf -

kubectl apply -f install-bundle/
```


## Deploy GCP Operator

> Note: The original GCP Operator still has some resource support that is missing in GCC.  Soon we hope to remote the need for this.

> Note: before running this modify the PROJECT ID in `gcp-operator/operator.yaml`

```bash
kubectl -n cnrm-system apply -f gcp-operator
```

Create a namespace that matches your PROJECT ID:

```bash
kubectl create namespace $PROJECT_ID
```

## Deploy Ops Manager

1. edit `./opsman/values.yaml` to change any defaults

2. Create manifests from helm and apply them:

```bash
helm template opsman --name gcc --namespace pgtm-pczarkowski > deploy/opsman.yaml
kubectl -n $PROJECT_ID apply -f opsman.yaml
```

## Set up DNS for opsman

manually right now

## Deploy Concourse

1. edit `./concourse/values.yaml`

2. Deploy

```bash
$ helm install stable/concourse --name concourse --version 8.2.5 \
   --namespace concourse --values ./concourse/values.yaml
```

## Configure Opsman

```bash
fly -t control-plane login -c http://127.0.0.1:8080

 fly -t control-plane set-pipeline \
     -p install-pks \
     -c pipeline.yml \
     -l env.yml

```