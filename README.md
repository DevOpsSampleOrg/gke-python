# GKE Workload Identity

[![Build](https://github.com/DevSecOpsSamples/gke-workload-identity/actions/workflows/build.yml/badge.svg?branch=master)](https://github.com/DevSecOpsSamples/gke-workload-identity/actions/workflows/build.yml) [![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=DevSecOpsSamples_gke-workload-identity&metric=alert_status)](https://sonarcloud.io/summary/new_code?id=DevSecOpsSamples_gke-workload-identity) [![Lines of Code](https://sonarcloud.io/api/project_badges/measure?project=DevSecOpsSamples_gke-workload-identity&metric=ncloc)](https://sonarcloud.io/summary/new_code?id=DevSecOpsSamples_gke-workload-identity)

## Overview

The Workload Identity is the recommended way for your workloads running on Google Kubernetes Engine (GKE) to access Google Cloud services in a secure and manageable way. 
In this sample project, we will learn GKE security with the IAM service account and Workload Identity.

> Applications running on GKE might need access to Google Cloud APIs such as Compute Engine API, BigQuery Storage API, or Machine Learning APIs.
> Workload Identity allows a Kubernetes service account in your GKE cluster to act as an IAM service account. Pods that use the configured Kubernetes service account automatically authenticate as the IAM service account when accessing Google Cloud APIs. Using Workload Identity allows you to assign distinct, fine-grained identities and authorization for each application in your cluster.
>
> https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity


## Objectives

Learn the features below:

- IAM service account and role/permission
- Workload Identity
- Pod specification for GKE service account and GCP load balancer

## Table of Contents

- Step1: Create a GKE cluster
- Step2: Create Kubernetes namespace and service account
- Step3: IAM service account for bucket-api
    - 3.1. Creating an IAM Service Account
    - 3.2. IAM policy binding between IAM service account and Kubernetes service account
    - 3.3. Annotate the Kubernetes service account
- Step4: GCS bucket creation and grant a permission
- Step5: Deploy bucket-api
- Step6: IAM service account for pubsub-api
    - 6.1. Creating an IAM Service Account
    - 6.2. IAM policy binding between IAM service account and Kubernetes service account
    - 6.3. Annotate the Kubernetes service account
- Step7: Create a Topic/Subscription and grant a permission
    - 7.1. Create a Topic and Subscription.
    - 7.2. Grant permission to IAM service account to publish to Topic
    - 7.3. Grant permission to IAM service account for subscription
- Step8: Deploy pubsub-api

---

## Prerequisites

### Installation

- [Install the gcloud CLI](https://cloud.google.com/sdk/docs/install)
- [Install gsutil](https://cloud.google.com/storage/docs/gsutil_install#install)
- [Install kubectl and configure cluster access](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl)

### Set environment variables

```bash
PROJECT_ID="sample-project" # replace with your project
COMPUTE_ZONE="us-central1"
SERVICE_ACCOUNT="bucket-api"
PUBSUB_SERVICE_ACCOUNT="pubsub-api"
GCS_BUCKET_NAME="bucket-api"
```

|   | Environment Variable           | Value             | Description                     |
|---|--------------------------------|-------------------|---------------------------------|
| 1 | PROJECT_ID                     | sample-project    | This variable will also be used for pub/sub deployment.          |
| 2 | COMPUTE_ZONE                   | us-central1       | Run `gcloud compute zones list` to get all zones.                |
| 3 | SERVICE_ACCOUNT                | bucket-api        | IAM service account for bucket-api to access to GCS bucket only. |
| 4 | PUBSUB_SERVICE_ACCOUNT         | pubsub-api        | IAM service account for pubsub-api to access to pub/sub only.    |
| 5 | GCS_BUCKET_NAME                | bucket-api        |           |

### Set GCP project

```bash
gcloud config set project ${PROJECT_ID}
gcloud config set compute/zone ${COMPUTE_ZONE}
```

---

## Step1: Create a GKE cluster

Create an Autopilot GKE cluster. It may take around 9 minutes.

```bash
gcloud container clusters create-auto hello-cluster --region=${COMPUTE_ZONE}
gcloud container clusters get-credentials hello-cluster
```

## Step2: Create Kubernetes namespace and service account

| API        | Object            | Name            | Description                 |
|------------|-------------------|-----------------|-----------------------------|
| bucket-api | namespace         | bucket-api      |                             |
| bucket-api | service account   | bucket-api-ksa  | Kubernetes service account  |
| pubsub-api | namespace         | pubsub-api      |                             |
| pubsub-api | service account   | pubsub-api-ksa  | Kubernetes service account  | 

```bash
kubectl create namespace bucket-api
kubectl create namespace pubsub-api

kubectl create serviceaccount --namespace bucket-api bucket-api-ksa
kubectl create serviceaccount --namespace pubsub-api pubsub-api-ksa
```

## Step3: IAM service account for bucket-api

3.1. Creating an IAM Service Account.

```bash
echo "PROJECT_ID: ${PROJECT_ID}, SERVICE_ACCOUNT: ${SERVICE_ACCOUNT}"

gcloud iam service-accounts create ${SERVICE_ACCOUNT} --display-name="bucket-api service account"
gcloud iam service-accounts list | grep bucket-api
```

3.2. Allow the Kubernetes service account to impersonate the IAM service account by adding an IAM policy binding between the two service accounts.

```bash
gcloud iam service-accounts add-iam-policy-binding \
       --role roles/iam.workloadIdentityUser \
       --member "serviceAccount:${PROJECT_ID}.svc.id.goog[bucket-api/bucket-api-ksa]" \
       ${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com
```

```yaml
Updated IAM policy for serviceAccount [bucket-api@sample-project.iam.gserviceaccount.com].
bindings:
- members:
  - serviceAccount:sample-project.svc.id.goog[bucket-api/bucket-api-ksa]
  role: roles/iam.workloadIdentityUser
etag: BwXtbNaPnNg=
version: 1
```

3.3. Annotate the Kubernetes service account with the email address of the IAM service account.

```bash
kubectl annotate serviceaccount --namespace bucket-api bucket-api-ksa \
        iam.gke.io/gcp-service-account=${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com
```

3.4. GCS bucket creation and grant a permission.

Create a GCS bucket

```bash
gcloud storage buckets create gs://${GCS_BUCKET_NAME}
```

Grant objectAdmin role to IAM service account to access a GCS bucket.

```bash
gsutil iam ch serviceAccount:${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com:objectAdmin
       gs://${GCS_BUCKET_NAME}/
```

Refer to the https://cloud.google.com/storage/docs/access-control/iam-roles page for predefined roles.

Use `serviceAccountName` for Pods:

[bucket-api-template.yaml](bucket-api/bucket-api-template.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bucket-api
  namespace: bucket-api
  annotations:
    app: 'bucket-api'
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bucket-api
  template:
    metadata:
      labels:
        app: bucket-api
    spec:
      serviceAccountName: bucket-api-ksa
      containers:
        - name: bucket-api
```

**NOTE:** If you want to test for debugging on your desktop with the IAM key, refer to the [README-test.md](README-test.md).

## Step4: Deploy bucket-api

4.1. Build and push to GCR:

```bash
docker build -t bucket-api . --platform linux/amd64
docker tag bucket-api:latest gcr.io/${PROJECT_ID}/bucket-api:latest

gcloud auth configure-docker
docker push gcr.io/${PROJECT_ID}/bucket-api:latest
```

4.2. Create and deploy K8s Deployment, Service, HorizontalPodAutoscaler, Ingress, and GKE BackendConfig using a template file.

```bash
sed -e "s|<project-id>|${PROJECT_ID}|g" bucket-api-template.yaml > bucket-api.yaml
cat bucket-api.yaml
kubectl apply -f bucket-api.yaml
```

Confirm that pod configuration and logs after deployment:

```bash
kubectl describe pods -n bucket-api
kubectl logs -l app=bucket-api -n bucket-api
```

4.3 Invoke `/bucket` API using a load balancer IP:

```bash
LB_IP_ADDRESS=$(gcloud compute forwarding-rules list | grep bucket-api | awk '{ print $2 }')
echo ${LB_IP_ADDRESS}

curl http://${LB_IP_ADDRESS}/
```

```json
{"host":"34.149.214.247","message":"bucket-api","method":"GET","url":"http://34.149.214.247/"}
```

```bash
curl http://${LB_IP_ADDRESS}/bucket
```

```json
{"blob_name":"put-test.txt","bucket_name":"bucket-api","response":"read/write test, bucket: bucket-api"}
```

## Step5: IAM service account for pubsub-api

5.1. Creating an IAM Service Account.

```bash
echo "PROJECT_ID: ${PROJECT_ID}, PUBSUB_SERVICE_ACCOUNT: ${PUBSUB_SERVICE_ACCOUNT}"

gcloud iam service-accounts create ${PUBSUB_SERVICE_ACCOUNT} --display-name="pubsub-api service account"
gcloud iam service-accounts list | grep pubsub-api
```

5.2. Allow the Kubernetes service account to impersonate the IAM service account by adding an IAM policy binding between the two service accounts.

```bash
gcloud iam service-accounts add-iam-policy-binding \
       --role roles/iam.workloadIdentityUser \
       --member "serviceAccount:${PROJECT_ID}.svc.id.goog[pubsub-api/pubsub-api-ksa]" \
       ${PUBSUB_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com
```

```yaml
Updated IAM policy for serviceAccount [pubsub-api@sample-project.iam.gserviceaccount.com].
bindings:
- members:
  - serviceAccount:sample-project.svc.id.goog[pubsub-api/pubsub-api-ksa]
  role: roles/iam.workloadIdentityUser
etag: BwXtbNaPnNg=
version: 1
```

5.3. Annotate the Kubernetes service account with the email address of the IAM service account.

```bash
kubectl annotate serviceaccount --namespace pubsub-api pubsub-api-ksa \
        iam.gke.io/gcp-service-account=${PUBSUB_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com
```

## Step6: Create a Topic/Subscription and grant a permission

6.1. Create a Topic and Subscription.

```bash
gcloud services enable cloudresourcemanager.googleapis.com pubsub.googleapis.com \
       container.googleapis.com
gcloud pubsub topics create echo
gcloud pubsub subscriptions create echo-read --topic=echo
```

6.2. Grant permission to IAM service account to publish to Topic.

```bash
echo "PUBSUB_SERVICE_ACCOUNT: ${PUBSUB_SERVICE_ACCOUNT}"
gcloud pubsub topics add-iam-policy-binding echo  \
      --member=serviceAccount:${PUBSUB_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com \
      --role=roles/pubsub.publisher
```

```bash
gcloud pubsub topics get-iam-policy echo --format yaml
```

```yaml
bindings:
- members:
  - serviceAccount:pubsub-api@sample-project.iam.gserviceaccount.com
  role: roles/pubsub.publisher
etag: BwXtaNHVrCw=
version: 1
```

6.3. Grant permission to IAM service account for subscription.

```bash
gcloud pubsub subscriptions add-iam-policy-binding echo-read \
       --member=serviceAccount:${PUBSUB_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com  \
       --role=roles/pubsub.subscriber
```

```bash
gcloud pubsub subscriptions get-iam-policy projects/${PROJECT_ID}/subscriptions/echo-read \
       --format yaml
```

```yaml
bindings:
- members:
  - serviceAccount:pubsub-api@sample-project.iam.gserviceaccount.com
  role: roles/pubsub.subscriber
etag: BwXtaNHVrCw=
version: 1
```

## Step7: Deploy pubsub-api

Build and push to GCR:

```bash
cd ../pubsub-api
docker build -t pubsub-api . --platform linux/amd64
docker tag pubsub-api:latest gcr.io/${PROJECT_ID}/pubsub-api:latest

gcloud auth configure-docker
docker push gcr.io/${PROJECT_ID}/pubsub-api:latest
```

Create and deploy K8s Deployment, Service, HorizontalPodAutoscaler, Ingress, and GKE BackendConfig using a template file.

```bash
sed -e "s|<project-id>|${PROJECT_ID}|g" pubsub-api-template.yaml > pubsub-api.yaml
cat pubsub-api.yaml
kubectl apply -f pubsub-api.yaml -n pubsub-api
```

Confirm that pod configuration and logs after deployment:

```bash
kubectl describe pods -n pubsub-api
kubectl logs -l app=pubsub-api -n pubsub-api
```

Confirm that response of `/pub`, `/sub`, and `/bucket` APIs.

```bash
LB_IP_ADDRESS=$(gcloud compute forwarding-rules list | grep pubsub-api | awk '{ print $2 }')
echo ${LB_IP_ADDRESS}
```

```bash
curl http://${LB_IP_ADDRESS}/pub
```

```json
{
  "result": "6237829865389825",
  "topic_name": "projects/sample-project/topics/echo"
}
```

```bash
curl http://${LB_IP_ADDRESS}/sub
```

```json
{
  "acknowledged": 7,
  "subscription_path": "projects/sample-project/subscriptions/echo-read",
  "topic_name": "projects/sample-project/topics/echo"
}
```

```bash
curl http://${LB_IP_ADDRESS}/bucket
```

`/bucket` API does not work because permission granted to pub/pub service only.

## Structure

```bash
├── build.gradle
├── bucket-api
│   ├── Dockerfile
│   ├── app.py
│   ├── bucket-api-template.yaml
│   ├── deploy.sh
│   └── requirements.txt
└── pubsub-api
    ├── Dockerfile
    ├── app.py
    ├── deploy.sh
    ├── pubsub-api-template.yaml
    └── requirements.txt
```
  
- [bucket-api-template.yaml](bucket-api/bucket-api-template.yaml)
- [pubsub-api-template.yaml](pubsub-api/pubsub-api-template.yaml)

## Cleanup

```bash
kubectl delete -f bucket-api/bucket-api.yaml
kubectl delete -f pubsub-api/pubsub-api.yaml

kubectl delete namespace bucket-api
kubectl delete namespace pubsub-api
```

```bash
gcloud storage buckets delete gs://${GCS_BUCKET_NAME}

gcloud pubsub subscriptions remove-iam-policy-binding echo-read --member=serviceAccount:${PUBSUB_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/pubsub.subscriber
gcloud pubsub subscriptions delete echo-read
gcloud pubsub topics delete echo

gcloud iam service-accounts delete --iam-account "${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com" 
gcloud iam service-accounts delete --iam-account "${PUBSUB_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com" 

docker system prune -a
```

## References

- [Use Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)
- [Cloud SDK > Documentation > Reference > gcloud iam service-accounts add-iam-policy-binding](https://cloud.google.com/sdk/gcloud/reference/iam/service-accounts/add-iam-policy-binding)
- [Cloud Pub/Sub > Documentation > Samples > Subscribe with synchronous pull](https://cloud.google.com/pubsub/docs/samples/pubsub-subscriber-sync-pull)
- https://github.com/GoogleCloudPlatform/kubernetes-engine-samples
- [Enabling IAP for GKE](https://cloud.google.com/iap/docs/enabling-kubernetes-howto)
