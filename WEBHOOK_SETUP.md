# Setting up the mutating webhook for K8S/Spinnaker

## Preparations

git clone REPO
export PROJECT_ID=rbmh-deployment-platform-prod
export WEBHOOK_NAME=rbmh-berglas-secrets-webhook
export SA_EMAIL=spinnaker@rbmh-deployment-platform-prod.iam.gserviceaccount.com

## Deployment

```
gcloud services enable --project ${PROJECT_ID} cloudfunctions.googleapis.com

cd examples/kubernetes

gcloud functions deploy ${WEBHOOK_NAME} \
  --project ${PROJECT_ID} \
  --runtime go111 \
  --entry-point F \
  --trigger-http \
  --region europe-west1
```

If you are asked for "Allow unauthorized invocation of new function" say no (N).

```
ENDPOINT=$(gcloud functions describe ${WEBHOOK_NAME} --project ${PROJECT_ID} --format 'value(httpsTrigger.url)' --region europe-west1)
```

Create a "webhook.yaml" file

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  name: berglas-webhook
  labels:
    app: berglas-webhook
    kind: mutator

webhooks:
- name: berglas-webhook.cloud.google.com
  clientConfig:
    url: REPLACE_WITH_YOUR_URL
    caBundle: ""
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["jobs", "pods"]
```

Apply the webhook.yaml file:

```
sed "s|REPLACE_WITH_YOUR_URL|$ENDPOINT|" webhook.yaml | kubectl apply -f -
```

## Permissions

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member "serviceAccount:${SA_EMAIL}" \
  --role roles/logging.logWriter

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member "serviceAccount:${SA_EMAIL}" \
  --role roles/monitoring.metricWriter

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member "serviceAccount:${SA_EMAIL}" \
  --role roles/monitoring.viewer