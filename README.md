# GCP Workload Identity Federation Kubernetes

## Explanation (TODO)

## Demo Requirements

- GCP Project Linked to Billing Account
- GCP User with enough privileges on the GCP project (Owner for testing purposes)
- A local minikube installation to test with k8s as an IdP

## Set Vars

```
export PROJECT_ID=<PROJECT_ID>
export SA_NAME=workload-identity
export ZONE=europe-west1-b
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")

export IDENTITY_POOL_NAME=local-minikube
export IDENTITY_POOL_PROVIDER=local-minikube

gcloud config set project $PROJECT_ID

## Inline substitutions for raw k8s manifests
sed -i "s/PROJECT_NUMBER/$PROJECT_NUMBER/g"  workload-identity-pod.yaml
sed -i "s/IDENTITY_POOL_NAME/$IDENTITY_POOL_NAME/g"  workload-identity-pod.yaml
sed -i "s/IDENTITY_POOL_PROVIDER/$IDENTITY_POOL_PROVIDER/g"  workload-identity-pod.yaml
```

## Local k8s setup

### Start minikube cluster locally

```
minikube start
```

### Create dedicated namespace and kubernetes service account and deployment using it

```
kubectl create namespace workload-identity
kubectl create sa workload-identity -n workload-identity

```

### Manually create a long-lived API token for the Service Account

```
kubectl apply -f long-lived-token-sa.yaml
kubectl get secret workload-identity -n workload-identity -o yaml
```

## GCP Service Account Setup

### Enable Workload Identity APIs

```
gcloud services enable \
    iam.googleapis.com \
    cloudresourcemanager.googleapis.com \
    iamcredentials.googleapis.com \
    sts.googleapis.com
```

### Create sample service account and grant viewer role on project to test API call later

```
gcloud iam service-accounts create $SA_NAME

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/viewer
```

## Workload Identity Pool setup

### Create workload identity pool

```
gcloud iam workload-identity-pools create $IDENTITY_POOL_NAME \
    --location global \
    --description $IDENTITY_POOL_NAME \
    --display-name $IDENTITY_POOL_NAME
```

### Get the issuer URI from k8s to use for WIP

```
export ISSUER_URI=$(kubectl get --raw /.well-known/openid-configuration | jq -r .issuer)
```

### Get the cluster JSON Web Key Set (JWKS)

```
kubectl get --raw /openid/v1/jwks > cluster-jwks.json
```

### Add the kubernetes cluster as Workload Identity pool

```
gcloud iam workload-identity-pools providers create-oidc $IDENTITY_POOL_PROVIDER \
    --location global \
    --workload-identity-pool $IDENTITY_POOL_NAME \
    --issuer-uri $ISSUER_URI \
    --attribute-mapping google.subject=assertion.sub \
    --jwk-json-path cluster-jwks.json
```

### Grant to SA workload identity user role

```
gcloud iam service-accounts add-iam-policy-binding $SA_NAME@$PROJECT_ID.iam.gserviceaccount.com \
    --role=roles/iam.workloadIdentityUser \
    --member="principal://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$IDENTITY_POOL_NAME/subject/system:serviceaccount:workload-identity:workload-identity"
```


### Create a credential configuration file to be used from k8s

```
gcloud iam workload-identity-pools create-cred-config \
    projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$IDENTITY_POOL_NAME/providers/$IDENTITY_POOL_PROVIDER \
    --service-account=$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com \
    --credential-source-file=/var/run/service-account/token \
    --credential-source-type=text \
    --output-file=credential-configuration.json
```

### Create ConfigMap importing the credential configuration

```
kubectl create configmap credential-configuration \
  --from-file credential-configuration.json \
  --namespace workload-identity
```

### Create workload identity pod using k8s projected voluems to inject KSA token audience and obtain auth token for GSA

```
kubectl apply -f workload-identity-pod.yaml
kubectl exec -it workload-identity --namespace workload-identity -- /bin/bash
cat /var/run/service-account/token 
gcloud auth print-access-token
exit
```

### Test manually by calling STS APIs to obtain GCP token from workload federation

```
export OIDC_TOKEN=$(kubectl exec -it workload-identity --namespace workload-identity -- cat /var/run/service-account/token)

export FEDERATED_TOKEN=$(curl -s -X POST \
   -d "grantType=urn:ietf:params:oauth:grant-type:token-exchange" \
   -d "audience=//iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$IDENTITY_POOL_NAME/providers/$IDENTITY_POOL_PROVIDER" \
   -d "subjectTokenType=urn:ietf:params:oauth:token-type:jwt" \
   -d "requestedTokenType=urn:ietf:params:oauth:token-type:access_token" \
   -d "scope=https://www.googleapis.com/auth/cloud-platform" \
   -d "subjectToken=$OIDC_TOKEN" \
   https://sts.googleapis.com/v1/token | jq -r '.access_token')
```

### This raw federated token only works for GCS and IAM APIs, so it could be used to impersonate the service account itself (or others where the Federated Service Account has a Service Account Token Creator Role)

### Create short-lived access token for service account using impersonation

```
export ACCESS_TOKEN=$(curl -X POST \
    -H "Authorization: Bearer $FEDERATED_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"scope": ["https://www.googleapis.com/auth/cloud-platform"]}' \
    "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com:generateAccessToken" \
    | jq -r '.accessToken')
```

### Test with API call

```
curl https://compute.googleapis.com/compute/v1/projects/$PROJECT_ID/zones/$ZONE/instances \
    -H "Authorization: Bearer $ACCESS_TOKEN"
```

## Cleanup

```
sed -i "s/$PROJECT_NUMBER/PROJECT_NUMBER/g"  workload-identity-pod.yaml
sed -i "s/$IDENTITY_POOL_NAME/IDENTITY_POOL_NAME/g"  workload-identity-pod.yaml
sed -i "s/$IDENTITY_POOL_PROVIDER/IDENTITY_POOL_PROVIDER/g"  workload-identity-pod.yaml

gcloud iam workload-identity-pools providers delete $IDENTITY_POOL_PROVIDER \
    --location global \
    --workload-identity-pool $IDENTITY_POOL_NAME \
    --quiet

gcloud iam workload-identity-pools delete $IDENTITY_POOL_NAME \
    --location global \
    --quiet

gcloud projects remove-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/viewer

gcloud iam service-accounts delete $SA_NAME@$PROJECT_ID.iam.gserviceaccount.com --quiet

kubectl delete namespace workload-identity
```