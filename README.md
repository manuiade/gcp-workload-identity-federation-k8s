# GCP Workload Identity Federation Kubernetes

## Explanation (TODO)

## Demo Requirements

- GCP Project Linked to Billing Account
- GCP User with enough privileges on the GCP project (Owner for testing purposes)
- A local minikube installation to test with k8s as an IdP

## Set Vars

```bash
export PROJECT_ID=<PROJECT_ID>
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
export K8S_NAMESPACE=dev-ns
export K8S_SERVICE_ACCOUNT=app123
export IDENTITY_POOL_NAME=local-minikube
export IDENTITY_POOL_PROVIDER=local-minikube
export GCP_SERVICE_ACCOUNT=app123

gcloud config set project $PROJECT_ID

## Inline substitutions for raw k8s manifests
sed -i "s/<PROJECT_ID>/$PROJECT_ID/g" app123-pod-direct.yaml app123-pod-impersonification.yaml
sed -i "s/<PROJECT_NUMBER>/$PROJECT_NUMBER/g" app123-pod-direct.yaml app123-pod-impersonification.yaml
sed -i "s/<IDENTITY_POOL_NAME>/$IDENTITY_POOL_NAME/g" app123-pod-direct.yaml app123-pod-impersonification.yaml
sed -i "s/<IDENTITY_POOL_PROVIDER>/$IDENTITY_POOL_PROVIDER/g" app123-pod-direct.yaml app123-pod-impersonification.yaml
```

## Local k8s setup

Start minikube cluster locally:

```bash
minikube start
```

Create dedicated namespace and kubernetes service account and deployment using it:

```bash
kubectl create namespace $K8S_NAMESPACE
kubectl create sa $K8S_SERVICE_ACCOUNT -n $K8S_NAMESPACE
```

## Workload Identity Setup

Enable Workload Identity APIs:

```bash
gcloud services enable \
    iam.googleapis.com \
    cloudresourcemanager.googleapis.com \
    iamcredentials.googleapis.com \
    sts.googleapis.com
```

Create workload identity pool:

```bash
gcloud iam workload-identity-pools create $IDENTITY_POOL_NAME \
    --location global \
    --description $IDENTITY_POOL_NAME \
    --display-name $IDENTITY_POOL_NAME
```

Get the issuer URI from k8s to use for WIP:

```bash
export ISSUER_URI=$(kubectl get --raw /.well-known/openid-configuration | jq -r .issuer)
```

Get the cluster JSON Web Key Set (JWKS):

```bash
kubectl get --raw /openid/v1/jwks > cluster-jwks.json
```

Add the kubernetes cluster as Workload Identity pool:

```bash
gcloud iam workload-identity-pools providers create-oidc $IDENTITY_POOL_PROVIDER \
    --location global \
    --workload-identity-pool $IDENTITY_POOL_NAME \
    --issuer-uri $ISSUER_URI \
    --attribute-mapping google.subject=assertion.sub \
    --jwk-json-path cluster-jwks.json
```

## Grant Access to Kubernetes Workload: Direct Access

Grant WIF identity access to GCP:

```bash
gcloud projects add-iam-policy-binding projects/$PROJECT_ID \
    --role=roles/compute.viewer \
    --member=principal://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$IDENTITY_POOL_PROVIDER/subject/system:serviceaccount:$K8S_NAMESPACE:$K8S_SERVICE_ACCOUNT \
    --condition=None
```

Create credential configuration files to instruct application environment fetch credentials for GCP following the OAuth and STS flow:

```bash
gcloud iam workload-identity-pools create-cred-config \
    projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$IDENTITY_POOL_NAME/providers/$IDENTITY_POOL_PROVIDER \
    --credential-source-file=/var/run/service-account/token \
    --credential-source-type=text \
    --output-file=credential-configuration.json
```

Create ConfigMap importing the credential configuration

```bash
kubectl create configmap credential-configuration \
  --from-file credential-configuration.json \
  --namespace dev-ns
```

Create workload identity pod using k8s projected voluems to inject KSA token audience and obtain auth token for GSA

```bash
kubectl apply -f app123-pod-direct.yaml --namespace $K8S_NAMESPACE
kubectl exec -it app123-pod-direct --namespace $K8S_NAMESPACE -- bash
```

Obtain access token using gcloud SDK (command inside pod):

```bash
gcloud auth login --cred-file $GOOGLE_APPLICATION_CREDENTIALS
gcloud auth list
export GCLOUD_ACCESS_TOKEN=$(gcloud auth print-access-token)
```

Obtain access token by manually calling STS APIs (command inside pod):

```bash
apk add jq
export OIDC_TOKEN=$(cat /var/run/service-account/token)
export STS_ACCESS_TOKEN=$(curl -s -X POST \
   -d "grantType=urn:ietf:params:oauth:grant-type:token-exchange" \
   -d "audience=//iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$IDENTITY_POOL_NAME/providers/$IDENTITY_POOL_PROVIDER" \
   -d "subjectTokenType=urn:ietf:params:oauth:token-type:jwt" \
   -d "requestedTokenType=urn:ietf:params:oauth:token-type:access_token" \
   -d "scope=https://www.googleapis.com/auth/cloud-platform" \
   -d "subjectToken=$OIDC_TOKEN" \
   https://sts.googleapis.com/v1/token | jq -r '.access_token')
```

Test GCP API call in several ways (command inside pod):

```bash
gcloud compute instances list --project $PROJECT_ID

curl https://compute.googleapis.com/compute/v1/projects/$PROJECT_ID/zones/europe-west1-b/instances \
    -H "Authorization: Bearer $GCLOUD_ACCESS_TOKEN"

curl https://compute.googleapis.com/compute/v1/projects/$PROJECT_ID/zones/europe-west1-b/instances \
    -H "Authorization: Bearer $STS_ACCESS_TOKEN"

exit
```


## Grant Access to Kubernetes Workload: Impersonating GCP Service Account

Create sample service account and grant viewer role on project to test API call later:

```bash
gcloud iam service-accounts create $GCP_SERVICE_ACCOUNT

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:$GCP_SERVICE_ACCOUNT@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/compute.viewer
```

Grant workload identity subject the Workload Identity User role to impersonate the service account using STS:

```bash
gcloud iam service-accounts add-iam-policy-binding $GCP_SERVICE_ACCOUNT@$PROJECT_ID.iam.gserviceaccount.com \
    --role=roles/iam.workloadIdentityUser \
    --member="principal://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$IDENTITY_POOL_PROVIDER/subject/system:serviceaccount:$K8S_NAMESPACE:$K8S_SERVICE_ACCOUNT"
```

Create a credential configuration file to be used from k8s (notice this time we are passing the service account argument for impersonification):

```bash
gcloud iam workload-identity-pools create-cred-config \
    projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$IDENTITY_POOL_NAME/providers/$IDENTITY_POOL_PROVIDER \
    --service-account=$GCP_SERVICE_ACCOUNT@$PROJECT_ID.iam.gserviceaccount.com \
    --credential-source-file=/var/run/service-account/token \
    --credential-source-type=text \
    --output-file=credential-configuration-impersonification.json
```

Create ConfigMap importing the credential configuration:

```bash
kubectl create configmap credential-configuration-impersonification \
  --from-file credential-configuration-impersonification.json \
  --namespace $K8S_NAMESPACE
```

Create workload identity pod using k8s projected voluems to inject KSA token audience and obtain auth token for GSA

```bash
kubectl apply -f app123-pod-impersonification.yaml --namespace $K8S_NAMESPACE
kubectl exec -it app123-pod-impersonification --namespace $K8S_NAMESPACE -- bash
```

Test manually by calling STS APIs to obtain GCP token from workload federation:

```bash
apk add jq
export OIDC_TOKEN=$(cat /var/run/service-account/token)
export FEDERATED_TOKEN=$(curl -s -X POST \
   -d "grantType=urn:ietf:params:oauth:grant-type:token-exchange" \
   -d "audience=//iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$IDENTITY_POOL_NAME/providers/$IDENTITY_POOL_PROVIDER" \
   -d "subjectTokenType=urn:ietf:params:oauth:token-type:jwt" \
   -d "requestedTokenType=urn:ietf:params:oauth:token-type:access_token" \
   -d "scope=https://www.googleapis.com/auth/cloud-platform" \
   -d "subjectToken=$OIDC_TOKEN" \
   https://sts.googleapis.com/v1/token | jq -r '.access_token')
```

### This raw federated token only works a limited set of APIs, so it could be used to impersonate the service account itself (or others where the Federated Service Account has a Service Account Token Creator Role)

Create short-lived access token for service account using impersonation:

```bash
export ACCESS_TOKEN=$(curl -X POST --http1.1 \
    -H "Authorization: Bearer $FEDERATED_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"scope": ["https://www.googleapis.com/auth/cloud-platform"]}' \
    "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/$GCP_SERVICE_ACCOUNT@$PROJECT_ID.iam.gserviceaccount.com:generateAccessToken" \
    | jq -r '.accessToken')
```
Test with API call

```bash
curl https://compute.googleapis.com/compute/v1/projects/$PROJECT_ID/zones/europe-west1-b/instances \
    -H "Authorization: Bearer $ACCESS_TOKEN"
```

## Test Attribute Conditions

Create production namespace in k8s:

```bash
kubectl create namespace prod-ns
kubectl create sa $K8S_SERVICE_ACCOUNT -n prod-ns
```

Add attribute mapping and condition to allow viewer role only for *prod-ns* namespace

```bash
gcloud iam workload-identity-pools providers update-oidc $IDENTITY_POOL_PROVIDER \
    --location global \
    --workload-identity-pool $IDENTITY_POOL_NAME \
    --attribute-mapping="google.subject=assertion.sub,\
attribute.namespace=assertion['kubernetes.io']['namespace'],\
attribute.service_account_name=assertion['kubernetes.io']['serviceaccount']['name'],\
attribute.pod=assertion['kubernetes.io']['pod']['name']" \
    --attribute-condition "assertion['kubernetes.io']['namespace'] in ['prod-ns']"
```

Retry to access GCP resource with *app123-pod-direct* and verify error:

```bash
kubectl exec -it app123-pod-direct --namespace $K8S_NAMESPACE -- bash
gcloud auth list
gcloud compute instances list
```

Create another pod in the *prod-ns* namespace and verify access:

```bash
kubectl create configmap credential-configuration \
  --from-file credential-configuration.json \
  --namespace prod-ns

kubectl apply -f app123-pod-direct.yaml --namespace prod-ns
kubectl exec -it app123-pod-direct --namespace prod-ns -- bash

gcloud auth login --cred-file $GOOGLE_APPLICATION_CREDENTIALS
gcloud auth list
# Credentials are good but know we get error on unsufficient privileges
gcloud compute instances list --project $PROJECT_ID
exit
```

Grant Viewer role on principalSet based on namespace membership:

```bash
gcloud projects add-iam-policy-binding projects/$PROJECT_ID \
    --role=roles/compute.viewer \
    --member=principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$IDENTITY_POOL_PROVIDER/attribute.namespace/prod-ns \
    --condition=None
```

Test again access to GCP (now it works):

```bash
kubectl exec -it app123-pod-direct --namespace prod-ns -- bash
gcloud compute instances list --project $PROJECT_ID
exit
```

## Cleanup

```bash
sed -i "s/$PROJECT_ID/<PROJECT_ID>/g" app123-pod-direct.yaml app123-pod-impersonification.yaml
sed -i "s/$PROJECT_NUMBER/<PROJECT_NUMBER>/g" app123-pod-direct.yaml app123-pod-impersonification.yaml
sed -i "s/$IDENTITY_POOL_NAME/<IDENTITY_POOL_NAME>/g" app123-pod-direct.yaml app123-pod-impersonification.yaml
sed -i "s/$IDENTITY_POOL_PROVIDER/<IDENTITY_POOL_PROVIDER>/g" app123-pod-direct.yaml app123-pod-impersonification.yaml

gcloud projects remove-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:$GCP_SERVICE_ACCOUNT@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/compute.viewer

gcloud projects remove-iam-policy-binding projects/$PROJECT_ID \
    --role=roles/compute.viewer \
    --member=principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$IDENTITY_POOL_PROVIDER/attribute.namespace/prod-ns

gcloud projects remove-iam-policy-binding projects/$PROJECT_ID \
    --role=roles/compute.viewer \
    --member=principal://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$IDENTITY_POOL_PROVIDER/subject/system:serviceaccount:$K8S_NAMESPACE:$K8S_SERVICE_ACCOUNT

gcloud iam workload-identity-pools providers delete $IDENTITY_POOL_PROVIDER \
    --location global \
    --workload-identity-pool $IDENTITY_POOL_NAME \
    --quiet

gcloud iam workload-identity-pools delete $IDENTITY_POOL_NAME \
    --location global \
    --quiet

gcloud iam service-accounts delete $GCP_SERVICE_ACCOUNT@$PROJECT_ID.iam.gserviceaccount.com --quiet

kubectl delete namespace $K8S_NAMESPACE
kubectl delete namespace prod-ns
```