# external secrets operator example for gke

## create project

```sh
PROJECT_ID=eso-$RANDOM # must be unique
gcloud projects create $PROJECT_ID
gcloud projects describe $PROJECT_ID
gcloud config set project $PROJECT_ID
```

## enable billing on account

```sh
# billing account id
ACCOUNT_ID=REDACTED
gcloud beta billing projects describe $PROJECT_ID
gcloud beta billing accounts describe $ACCOUNT_ID

gcloud beta billing projects link $PROJECT_ID \
  --billing-account=$ACCOUNT_ID
```

## enable apis

```sh
gcloud services enable \
container.googleapis.com # for gke \
secretmanager.googleapis.com # for gsm \
iamcredentials.googleapis.com # for workload identity
```

## Create cluster

```sh
CLUSTER_NAME=eso
LOCATION=us-east4-c
# Enable workload identity
gcloud container clusters create $CLUSTER_NAME \
  --location=$LOCATION \
  --workload-pool=$PROJECT_ID.svc.id.goog \
  --num-nodes=3
```

## get credentials

```sh
gcloud container clusters get-credentials $CLUSTER_NAME \
  --location=$LOCATION
```

## authz and principles

```sh
NAMESPACE=example
KSA_NAME=example-ksa
# use yq or just look at describe output
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID | yq .projectNumber)

kubectl create namespace $NAMESPACE

kubectl create serviceaccount $KSA_NAME \
  --namespace $NAMESPACE

gcloud projects add-iam-policy-binding projects/$PROJECT_ID \
    --role=roles/container.clusterViewer \
    --role=roles/secretmanager.secretAccessor \
    --member=principal://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$PROJECT_ID.svc.id.goog/subject/ns/$NAMESPACE/sa/$KSA_NAME \
    --condition=None
```

## Test workload identity

```sh
BUCKET=eso-$RANDOM # must be unique

gcloud storage buckets create gs://$BUCKET

gcloud storage buckets add-iam-policy-binding gs://$BUCKET \
    --role=roles/storage.objectViewer \
    --member=principal://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$PROJECT_ID.svc.id.goog/subject/ns/$NAMESPACE/sa/$KSA_NAME \
    --condition=None

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: $NAMESPACE
spec:
  # nodeSelector:
  #   iam.gke.io/gke-metadata-server-enabled: "true"
  serviceAccountName: $KSA_NAME
  containers:
  - name: test-pod
    image: google/cloud-sdk:slim
    command: ["sleep","infinity"]
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
        ephemeral-storage: 10Mi
EOF

kubectl get pods --namespace=$NAMESPACE

# note we pass BUCKET env to kubectl exec for test below
kubectl exec -it pods/test-pod --namespace=$NAMESPACE -- \
  env BUCKET=$BUCKET /bin/bash

root@test-pod:/# curl -X GET -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    "https://storage.googleapis.com/storage/v1/b/$BUCKET/o"

# you should see
{
  "kind": "storage#objects"
}
# which shows that your Pod can access objects in the bucket
```

## now try gsm secrets

using External Secrets Operator (eso)

```sh
GITHUB_USER=YOUR_USER # your actual gh user lol
# if you don't alredy use GitHub's gh command, you're welcome
gh auth token | docker login ghcr.io \
  --username $GITHUB_USER \
  --password-stdin

# install the operator the OCI way
# to-do: write a PR to update their Helm install docs
CHART=oci://ghcr.io/external-secrets/charts/external-secrets
helm install external-secrets $CHART \
  -n external-secrets \
  --create-namespace

# because we installed with Helm, the SA is "external-secrets"
# Creating Workload Identity Service Accounts option
# ESO links to above guide. I am using a service principle above
# so let's use it here too
ESO_NAMESPACE=external-secrets
ESO_KSA_NAME=external-secrets
# PROJECT_NUMBER already set above
# PROJECT_ID already set above

gcloud projects add-iam-policy-binding projects/$PROJECT_ID \
    --role=roles/secretmanager.secretAccessor \
    --member=principal://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$PROJECT_ID.svc.id.goog/subject/ns/$ESO_NAMESPACE/sa/$ESO_KSA_NAME \
    --condition=None

# add these resources to the namespace you want the gsm secret synced to
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-store
  namespace: example
spec:
  provider:
    gcpsm:
      projectID: $PROJECT_ID
EOF

# create google secret manager secret
echo 'my-api-key' > readonly-key.txt
gcloud secrets create readonly-key \
  --data-file readonly-key.txt \
  --ttl=3600s

cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-api-key
  namespace: example
spec:
  # super fast refresh for demo only
  # you can re-deploy this CRD with any new value (label etc),
  # or delete and recreate, to trigger a re-sync before the next interval
  # refreshInterval: 1h
  refreshInterval: 30s
  secretStoreRef:
    kind: SecretStore
    name: gcp-store
  target:
    name: readonly-key
    creationPolicy: Owner
  data:
  - secretKey: key
    remoteRef:
      key: readonly-key
EOF

# you should now have a new k8s secret created by the operator
kubectl get secret
NAME                                     TYPE                 DATA   AGE
readonly-key                             Opaque               1      15s

SECRETNAME=readonly-key
kubectl get secret $SECRETNAME -o json | jq '.data | map_values(@base64d)'

# you should see
{
  "key": "my-api-key"
}
```

## cleanup

```sh
# easiest way is to delete the entire project
gcloud projects delete $PROJECT_ID
```

## Resources

- <https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity>
- <https://cloud.google.com/kubernetes-engine/docs/tutorials/workload-identity-secrets>
- <https://cloud.google.com/billing/docs/how-to/verify-billing-enabled#console>
