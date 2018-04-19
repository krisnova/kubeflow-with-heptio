# kubeflow-with-heptio
Working repository for our presentation on running Kubeflow with Heptio tools on the backend

# Setup gcloud
```
export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)"
echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" |\
sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg |\
sudo apt-key add -
sudo apt-get update && sudo apt-get install google-cloud-sdk kubectl
```

# Check to make sure it’s installed and up to date.
```
gcloud --version
```

# A bunch of GCP Setup Variables
```
export PROJECT_ID=ce-tpu-training
export SUBNETWORK_NAME=tpu-subnetwork
export CLUSTER_NAME=kubeflow-demo
export NAMESPACE=kubeflow
export EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:cloud-endpoints-controller" \
    --format='value(email)')
export BUCKET_NAME=kubeflow-data-bucket
gcloud config set project $PROJECT_ID
gcloud config set compute/zone us-central1-c
gcloud config set container/use_v1_api_client false
```
# Create service accounts
# IAM
```
gcloud iam service-accounts create ${SVC_ACCT} --display-name=${SVC_ACCT}
```

# Grant cluster admin role
```
gcloud projects add-iam-policy-binding ${PROJECT_ID} --member=serviceAccount:${SVC_ACCT}@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/container.clusterAdmin
```
# Grant service account actor role
```
gcloud projects add-iam-policy-binding ${PROJECT_ID} --member=serviceAccount:${SVC_ACCT}@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/iam.serviceAccountActor

gcloud iam service-accounts keys create /tmp/${SVC_ACCT}_key.json --iam-account=${SVC_ACCT}@${PROJECT_ID}.iam.gserviceaccount.com
gcloud auth activate-service-account ${SVC_ACCT}@${PROJECT_ID}.iam.gserviceaccount.com --key-file=/tmp/${SVC_ACCT}_key.json
```

# Create the cluster
# For TPUs
```
gcloud alpha container clusters create keynote_cluster \
--enable-kubernetes-alpha \
--zone=us-central1-c \
--num-nodes=3 \
--machine-type=n1-standard-4 \
--enable-ip-alias \
--create-subnetwork name=$SUBNET_NAME \
--scopes=cloud-platform \ 
--enable-tpu
```

# For GPUs
```
gcloud beta container clusters create ${CLUSTER_NAME}-gpu \
     --accelerator type=nvidia-tesla-p100,count=4 \
     --zone us-central1-c \
     --cluster-version 1.9.3 \
	--num-nodes=3 \
	--machine-type=n1-standard-4
```

# ***************** Below for GPUs only **************
# Install device drivers for GPU on the cluster nodes
```
kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/k8s-1.9/nvidia-driver-installer/cos/daemonset-preloaded.yaml
```

# Setup RBAC
```
kubectl create clusterrolebinding cluster-admin-binding \
--clusterrole cluster-admin --user $(gcloud config get-value account)
```
