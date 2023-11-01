# Maximus POV

## Build AWS Jumpbox

* Launch Instance and choose Ubuntu 20.04
* Create or Select Keypair
* Create Instance
  * Minimum CPU: 2
  * Minimum RAM: 16GB RAM
  * Minimum Disk: 30GB
* SSH to the new host

### Install DKP
```
wget https://downloads.d2iq.com/dkp/v2.6.0/dkp_v2.6.0_linux_amd64.tar.gz
```
```
tar xvf dkp_v2.6.0_linux_amd64.tar.gz
sudo mv dkp /usr/local/bin
```
### Install Konvoy Image Builder (KIB)
```
wget https://github.com/mesosphere/konvoy-image-builder/releases/download/v2.5.1/konvoy-image-bundle-v2.5.1_linux_amd64.tar.gz
```
```
mkdir kib
mv konvoy-image-bundle-v2.5.1_linux_amd64.tar.gz kib
cd kib
tar xvf konvoy-image-bundle-v2.5.1_linux_amd64.tar.gz
```

### Purge snapd
```
sudo apt -y purge snapd
```

### Install kubectl
```
cd ~
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version
```

### Install Helm on jumphost
```
wget https://get.helm.sh/helm-v3.12.0-linux-amd64.tar.gz
tar -zxvf helm-v3.12.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
rm -r linux-amd64 helm-v3.12.0-linux-amd64.tar.gz
```

### Install Docker
```
cd ~
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
```
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
```
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```
sudo apt-get update
```
```
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
```
sudo usermod -aG docker ${USER}
```
* Log out, log back in
```
docker run hello-world
```

### Install AWS CLI
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```
```
sudo apt -y install unzip
```
```
unzip awscliv2.zip
```
```
sudo ./aws/install
```
```
aws --version
```

### Install Google CLI
```
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-447.0.0-linux-x86_64.tar.gz

tar -xf google-cloud-cli-447.0.0-linux-x86_64.tar.gz

./google-cloud-sdk/install.sh

./google-cloud-sdk/bin/gcloud init
```
<br>
<br>

## Create DKP Management Cluster in AWS
* Note: The management cluster must be installed as EC2 instances because we need to communicate with the control plane for the management cluster and in EKS, AWS manages the control planes so we are unable to do what we need.

### Login to AWS CLI
```
aws configure
```

### Set Environment Variables
```
export AWS_DEFAULT_REGION=us-east-1
export AWS_REGION=us-east-1
export AWS_PROFILE=110465657741_Mesosphere-PowerUser
```
```
export REGISTRY_MIRROR_URL="https://registry-1.docker.io"
export REGISTRY_MIRROR_USERNAME="dkptestdrive"
export REGISTRY_MIRROR_PASSWORD="43bfb7f5-de67-4fb0-a9a7-32dd3bdcc2a6"
```
```
export AWS_VPC_ID=""
export AWS_SUBNET_ID=""
export AWS_SECURITY_GROUP_ID=""
```

### Start the Konvoy Image Build (KIB) process (defaults in region us-west-2)
```
cd kib
```

#### Create overrides file for Docker registry
```
cat <<EOF > docker_overrides.yaml
image_registries_with_auth:
- host: "registry-1.docker.io"
  username: "dkptestdrive"
  password: "43bfb7f5-de67-4fb0-a9a7-32dd3bdcc2a6"
  auth: ""
  identityToken: ""
EOF
```

#### Create the following overrides file for AWS
```
cat <<EOF > aws_overrides.yaml
packer:
  vpc_id: "${AWS_VPC_ID}"
  subnet_id: "${AWS_SUBNET_ID}"
  security_group_id: "${AWS_SECURITY_GROUP_ID}"
  aws_region: "${AWS_REGION}"
EOF
```

Set AWS Default Region
```
aws configure set default.region us-east-1 <profile>
aws configure set region us-east-1 --profile <profile>
```

```
./konvoy-image build aws --overrides docker_overrides.yaml,aws_overrides.yaml --region us-east-1 images/ami/ubuntu-2004.yaml
```

Grab the AWS AMI ID
```
export AWS_AMI_ID=ami-0407cd8c5d40b2d55
```

### Build Cluster and Install Kommander UI
```
export CLUSTER_NAME=
```

#### Delete Bootstrap
```
dkp delete bootstrap
```

#### Create cluster config for AWS, Create cluster in AWS
```
dkp create bootstrap
```

#### Create and build management cluster

Create `CLUSTER_NAME.yaml` file (dry-run)
* Must have the following tags on the subnets
  * Tag : Value
  * kubernetes.io/cluster/`<cluster-name>` : owned
  * Public Subnet:  kubernetes.io/role/internal-elb : 1
  * Private Subnet:  kubernetes.io/role/elb : 1
* Must have both private and public subnets available to use
```
dkp create cluster aws \
--cluster-name=${CLUSTER_NAME} \
--additional-tags=owner=mshayda,expiration=3d \
--with-aws-bootstrap-credentials=true \
--registry-mirror-url ${REGISTRY_MIRROR_URL} \
--registry-mirror-username ${REGISTRY_MIRROR_USERNAME} \
--registry-mirror-password ${REGISTRY_MIRROR_PASSWORD} \
--region us-east-1 \
--ami ${AWS_AMI_ID} \
--subnet-ids=<subnet_id1,subnetid_2> \
--vpc-id <vpc ID> \
--dry-run -o yaml > ${CLUSTER_NAME}.yaml
```

* Edit the ${CLUSTER_NAME}.yaml file to edit the Public subnets to be True
  * Set isPublic: true for all public subnets

```
spec:
  network:
    subnets:
    - id: <subnet id>
      isPublic: false|true
```

Build the cluster in AWS from the yaml file
```
kubectl create -f ${CLUSTER_NAME}.yaml
```
```
watch -n .5 dkp describe cluster -c ${CLUSTER_NAME}
```
If aws credential errors
Update bootstrap credentials
```
dkp update bootstrap credentials aws
```

```
dkp get kubeconfig -c ${CLUSTER_NAME} > ${CLUSTER_NAME}.conf
```
```
dkp create capi-components --kubeconfig ${CLUSTER_NAME}.conf
```
```
dkp move capi-resources --to-kubeconfig ${CLUSTER_NAME}.conf
```

#################################
Ensure resources have moved
```
kubectl get nodes --kubeconfig ${CLUSTER_NAME}.conf
```
```
kubectl get nodes
```
```
dkp delete bootstrap
```

Clean up stale cluster
```
kubectl delete -f ${CLUSTER_NAME}.yaml 
```
#################################

#### Install Kommander UI
```
dkp install kommander --kubeconfig=${CLUSTER_NAME}.conf
```
```
watch -n .5 kubectl get hr -A --kubeconfig ${CLUSTER_NAME}.conf
```

Get Dashboard URL and default username/password
```
dkp open dashboard --kubeconfig=${CLUSTER_NAME}.conf
```

### Licenses

#### DKP
```
eyJhbGciOiJSUzI1NiIsImtpZCI6IiIsInR5cCI6IkpXVCJ9.eyJ2ZXJzaW9uIjoiMS4wIiwibGljZW5zZV9pZCI6IjY0YjkyMTE4LTMwNGYtYTliNC04YzY1LWJmNTJkZTNmZDRmZSIsImRrcF9sZXZlbCI6ImVudGVycHJpc2UiLCJjb3JlX2NhcGFjaXR5IjoyMDAsImV4cCI6MTY5ODczNTYwMCwiaWF0IjoxNjk1ODQyNDE1LCJpc3MiOiJEMmlRLCBJbmMuIiwibmJmIjoxNjk1Nzk4MDAwLCJzdWIiOiIwMDEzWjAwMDAyUlVyT3NRQUwifQ.NVH5AGNuHCdIn0C_Gl0oJMDWMpYS-SUiXJtybFtF2_wuN8pgY-5__aGyzQJboFlmSCH4D4IddXaSxJgvM48174HsIrEz67iwMuWKn-sCOi1OOXDdadm28I3_RlGi761-9OsvWHeMDBWBhSYQkauDp7Qu_5oWP4bTq157cOYNicY
```

#### Insights
```
eyJhbGciOiJSUzI1NiIsImtpZCI6IiIsInR5cCI6IkpXVCJ9.eyJ2ZXJzaW9uIjoiMS4wIiwibGljZW5zZV9pZCI6IjZhZWI1M2E2LTUwZTctMTFlZS1hYzZhLTIzNWI3ODI4NmJjOCIsImRrcF9sZXZlbCI6Imluc2lnaHRzIiwiY29yZV9jYXBhY2l0eSI6MSwiZXhwIjoxNzEwMTM2ODAwLCJpYXQiOjE2OTQ0NjY5NTMsImlzcyI6IkQyaVEsIEluYy4iLCJuYmYiOjE2OTQ0MTIwMDAsInN1YiI6IjAwMWowMDAwMDBYQmlHS0FBMSJ9.gZMWsL3pj6vcv79G8_b_Opo1sAtB2xUZsOHagvdFwASjREWzAWYKHZTdp1dRDOC3ygHDc2gGUT0ERcfz3r9YwJ9fWGn9xpHdABylNKewYFtJI-C7fH26CD6GfyhrdI5h6Xhmj0WlBNrugaZ4XK7JPqqDqazgp97yp31cxw9h-tE
```

## Patch the Management Cluster
```
kubectl set image -n kommander deployment/kommander-appmanagement manager=mesosphere/kommander2-appmanagement:v2.6.1-dev
```

## Create AWS Cluster and EKS Cluster from the DKP UI
* Create Workspace
* Continue to that new Workspace
* Setup AWS Infrastructure Provider
<br>
Role ARN for Infrastructure Provider:
arn:aws:iam::110465657741:role/control-plane.cluster-api-provider-aws.sigs.k8s.io

Change Trust Relationships to
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::110465657741:root"
            },
            "Action": "sts:AssumeRole"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "eks.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```
* Create AWS Cluster:
  * Select Clusters -> Add Cluster -> Create Cluster -> Select AWS
<br>
* Create EKS Clsuter:
  * Select Clusters -> Add Cluster -> Create Cluster -> Select EKS
<br>
Note: For the POV, we were unable to create clusters through the UI due to the way Maximus sets up their Roles/Policies.  This will be worked out with a Professional Services engagement.

<br>
<br>

## Create AWS Cluster from the DKP CLI and Attach through the UI
```
dkp create bootstrap
```
Set CLUSTER_NAME variable to your new worker cluster to create
```
export CLUSTER_NAME=
```
Create `CLUSTER_NAME.yaml` file (dry-run)
```
dkp create cluster aws \
--cluster-name=${CLUSTER_NAME} \
--additional-tags=owner=mshayda,expiration=3d \
--with-aws-bootstrap-credentials=true \
--registry-mirror-url ${REGISTRY_MIRROR_URL} \
--registry-mirror-username ${REGISTRY_MIRROR_USERNAME} \
--registry-mirror-password ${REGISTRY_MIRROR_PASSWORD} \
--region us-east-1 \
--ami ${AWS_AMI_ID} \
--subnet-ids=<subnet_id1,subnetid_2> \
--vpc-id <vpc ID> \
--dry-run -o yaml > ${CLUSTER_NAME}.yaml
```
* Edit the ${CLUSTER_NAME}.yaml file to edit the Public subnets to be True
  * Set isPublic: true for all public subnets

```
spec:
  network:
    subnets:
    - id: <subnet id>
      isPublic: false|true
```

Build the cluster in AWS from the yaml file
```
kubectl create -f ${CLUSTER_NAME}.yaml
```
```
watch -n .5 dkp describe cluster -c ${CLUSTER_NAME}
```
If there are AWS credential errors in the logs you'll need to update bootstrap credentials
```
dkp update bootstrap credentials aws
```
Create the cluster config file
```
dkp get kubeconfig -c ${CLUSTER_NAME} > ${CLUSTER_NAME}.conf
```
```
dkp create capi-components --kubeconfig ${CLUSTER_NAME}.conf
```
```
dkp move capi-resources --to-kubeconfig ${CLUSTER_NAME}.conf
```
<br>

Attach the cluster to DKP for management through the UI

* Download the ${CLUSTER_NAME}.conf file
* In the UI, go to the Workspace you want to deploy the cluster
* Select Clusters -> Add Cluster -> Attach Cluster -> No Additional Networking Restrictions
* Upload the kubeconfig file of the newly created AWS worker cluster
* Click Create
<br>


## Create an EKS Cluster from the CLI and Attach to DKP through the CLI

* NOTE: To Create EKS cluster from CLI must have EKS Roles in place
  * Use Cloud formation stack from: https://docs.d2iq.com/dkp/2.6/eks-cluster-iam-policies-and-roles

* Re-log into AWS (if needed)

### Create the EKS Cluster and attach to DKP
Set EKS_CLUSTER_NAME
```
export EKS_CLUSTER_NAME=
```
```
dkp create cluster eks \
--cluster-name=${EKS_CLUSTER_NAME} \
--additional-tags=owner=mshayda,expiration=3d \
--with-aws-bootstrap-credentials=true \
--registry-mirror-url ${REGISTRY_MIRROR_URL} \
--registry-mirror-username ${REGISTRY_MIRROR_USERNAME} \
--registry-mirror-password ${REGISTRY_MIRROR_PASSWORD} \
--region us-east-1 \
--subnet-ids=<subnet_id1,subnetid_2> \
--vpc-id <vpc ID> \
--dry-run -o yaml > ${EKS_CLUSTER_NAME}.yaml
```
Build the cluster in AWS from the yaml file
```
kubectl create -f ${EKS_CLUSTER_NAME}.yaml
```
```
watch -n .5 dkp describe cluster -c ${EKS_CLUSTER_NAME}
```
If there are AWS credential errors in the logs you'll need to update bootstrap credentials
```
dkp update bootstrap credentials aws
```
```
dkp get kubeconfig -c ${EKS_CLUSTER_NAME} > ${CLUSTER_NAME}.conf
```
```
dkp create capi-components --kubeconfig ${CLUSTER_NAME}.conf
```
```
dkp move capi-resources --to-kubeconfig ${CLUSTER_NAME}.conf
```
```
dkp get kubeconfig -c ${CLUSTER_NAME} > ${CLUSTER_NAME}.conf
```


### When complete, check you can see the cluster via CLI
```
export REGION=us-east-1
```
```
aws eks --region ${REGION} list-clusters
```
```
export EKS_CLUSTER_NAME=
```

### Download EKS cluster kubeconfig.conf files
```
aws eks --region ${REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME} --kubeconfig ${EKS_CLUSTER_NAME}.conf
```

Check you can see the services of the cluster
```
kubectl get svc --kubeconfig ${EKS_CLUSTER_NAME}.conf
```

### Configure EKS kubeconfig to be DKP ready and Attach

#### Create necessary service account (do for each EKS cluster)
```
kubectl -n kube-system create serviceaccount kommander-cluster-admin --kubeconfig ${EKS_CLUSTER_NAME}.conf
```

Create token secret for the serviceaccount (do for each EKS cluster)
```
kubectl -n kube-system create --kubeconfig ${EKS_CLUSTER_NAME}.conf -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: kommander-cluster-admin-sa-token
  annotations:
    kubernetes.io/service-account.name: kommander-cluster-admin
type: kubernetes.io/service-account-token
EOF
```

Verify service token is running
```
kubectl -n kube-system get secret kommander-cluster-admin-sa-token --kubeconfig ${EKS_CLUSTER_NAME}.conf -o yaml
```

Configure new service account for cluster-admin permissions (do for each EKS cluster)
```
cat << EOF | kubectl apply --kubeconfig ${EKS_CLUSTER_NAME}.conf -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kommander-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kommander-cluster-admin
  namespace: kube-system
EOF
```

#### Setup the following variables AND generate a new kubeconfig (do for each EKS cluster)
```
export USER_TOKEN_VALUE=$(kubectl --kubeconfig ${EKS_CLUSTER_NAME}.conf -n kube-system get secret/kommander-cluster-admin-sa-token -o=go-template='{{.data.token}}' | base64 --decode)
export CURRENT_CONTEXT=$(kubectl --kubeconfig ${EKS_CLUSTER_NAME}.conf config current-context)
export CURRENT_CLUSTER=$(kubectl --kubeconfig ${EKS_CLUSTER_NAME}.conf config view --raw -o=go-template='{{range .contexts}}{{if eq .name "'''${CURRENT_CONTEXT}'''"}}{{ index .context "cluster" }}{{end}}{{end}}')
export CLUSTER_CA=$(kubectl --kubeconfig ${EKS_CLUSTER_NAME}.conf config view --raw -o=go-template='{{range .clusters}}{{if eq .name "'''${CURRENT_CLUSTER}'''"}}"{{with index .cluster "certificate-authority-data" }}{{.}}{{end}}"{{ end }}{{ end }}')
export CLUSTER_SERVER=$(kubectl --kubeconfig ${EKS_CLUSTER_NAME}.conf config view --raw -o=go-template='{{range .clusters}}{{if eq .name "'''${CURRENT_CLUSTER}'''"}}{{ .cluster.server }}{{end}}{{ end }}')
```
Verify variables were set correctly
```
export -p | grep -E 'USER_TOKEN_VALUE|CURRENT_CONTEXT|CURRENT_CLUSTER|CLUSTER_CA|CLUSTER_SERVER'
```

Generate a new kubeconfig file with the environment variables just set
```
cat << EOF > kommander-cluster-admin-config-${EKS_CLUSTER_NAME}.conf
apiVersion: v1
kind: Config
current-context: ${CURRENT_CONTEXT}
contexts:
- name: ${CURRENT_CONTEXT}
  context:
    cluster: ${CURRENT_CONTEXT}
    user: kommander-cluster-admin
    namespace: kube-system
clusters:
- name: ${CURRENT_CONTEXT}
  cluster:
    certificate-authority-data: ${CLUSTER_CA}
    server: ${CLUSTER_SERVER}
users:
- name: kommander-cluster-admin
  user:
    token: ${USER_TOKEN_VALUE}
EOF
```

Verify the new config file can access the cluster (do this for each EKS cluster)
```
kubectl --kubeconfig $(pwd)/kommander-cluster-admin-config-${EKS_CLUSTER_NAME}.conf get all --all-namespaces
```

### Use `kommander-cluster-admin-config-${EKS_CLUSTER_NAME}.conf` Config File to Upload to DKP to Attach Cluster

#### Attach EKS Cluster via CLI
```
dkp attach cluster -n ${EKS_CLUSTER_NAME} --attached-kubeconfig kommander-cluster-admin-config-${EKS_CLUSTER_NAME}.conf
```

#### Attach EKS Cluster via UI
* Download the `kommander-cluster-admin-config-${EKS_CLUSTER_NAME}.conf` to your local machine
* From the DKP UI
  * Go to the new Workspace
  * Create AWS Cluster:
    * Select Clusters -> Add Cluster -> Attach Cluster -> No additional networking restrictions
    * Upload kubeconfig Files (select `kommander-cluster-admin-config-${EKS_CLUSTER_NAME}.conf`)
    * Change name of cluster to ${EKS_CLUSTER_NAME}
<br>
<br>
<br>

## Setup Insights and other Applications for AWS Clusters

* Under the new Workspace
* Select Applications in the left pane
* Select the 3 dots for Rook Ceph
  * Select Enable for all AWS clusters
* Select Rook Ceph Cluster
  * Select Enable for all AWS clusters
* Select Insights
  * Select Enable for all AWS clusters

<br>
<br>
<br>


## GCP and GKE

### Create/Retrieve GCP Roles/Credentials

Setup the environment variables
```
export GCP_PROJECT=<your GCP project ID>
export GCP_SERVICE_ACCOUNT_USER=<some new or existing service account user>
export GOOGLE_APPLICATION_CREDENTIALS="$HOME/.gcloud/credentials.json"
export GCP_REGION=
export GCP_REGION=
export NETWORK_NAME=
```
Example environment variables
```
export GCP_PROJECT=konvoy-gcp-se
export GCP_SERVICE_ACCOUNT_USER=mshayda-service-account
export GOOGLE_APPLICATION_CREDENTIALS="$HOME/.gcloud/credentials.json"
export GCP_REGION=us-west1
export NETWORK_NAME=shayda-network
```

Login to GCP and set the Project ID
```
gcloud auth login
```
or
```
gcloud auth login --no-launch-browser
```
```
gcloud config set project ${GCP_PROJECT}
```

Create a NEW service account
```
gcloud iam service-accounts create "${GCP_SERVICE_ACCOUNT_USER}" --project=${GCP_PROJECT}
gcloud projects add-iam-policy-binding ${GCP_PROJECT} --member="serviceAccount:${GCP_SERVICE_ACCOUNT_USER}@${GCP_PROJECT}.iam.gserviceaccount.com" --role=roles/compute.instanceAdmin.v1
gcloud projects add-iam-policy-binding ${GCP_PROJECT} --member="serviceAccount:${GCP_SERVICE_ACCOUNT_USER}@${GCP_PROJECT}.iam.gserviceaccount.com" --role=roles/iam.serviceAccountUser
gcloud projects add-iam-policy-binding ${GCP_PROJECT} --member="serviceAccount:${GCP_SERVICE_ACCOUNT_USER}@${GCP_PROJECT}.iam.gserviceaccount.com" --role=roles/compute.securityAdmin
gcloud iam service-accounts keys create ${GOOGLE_APPLICATION_CREDENTIALS} --iam-account="${GCP_SERVICE_ACCOUNT_USER}@${GCP_PROJECT}.iam.gserviceaccount.com"
```


If you have already created a service account, RETRIEVE the credentials for an EXISTING service account using the following gcloud commands:
```
gcloud projects add-iam-policy-binding ${GCP_PROJECT} --member="serviceAccount:${GCP_SERVICE_ACCOUNT_USER}@${GCP_PROJECT}.iam.gserviceaccount.com" --role=roles/compute.instanceAdmin.v1
gcloud projects add-iam-policy-binding ${GCP_PROJECT} --member="serviceAccount:${GCP_SERVICE_ACCOUNT_USER}@${GCP_PROJECT}.iam.gserviceaccount.com" --role=roles/iam.serviceAccountUser
gcloud projects add-iam-policy-binding ${GCP_PROJECT} --member="serviceAccount:${GCP_SERVICE_ACCOUNT_USER}@${GCP_PROJECT}.iam.gserviceaccount.com" --role=roles/compute.securityAdmin
gcloud iam service-accounts keys create ${GOOGLE_APPLICATION_CREDENTIALS} --iam-account="${GCP_SERVICE_ACCOUNT_USER}@${GCP_PROJECT}.iam.gserviceaccount.com"
```

<br>
<br>

#### If you get the error `ERROR: (gcloud.iam.service-accounts.keys.create) FAILED_PRECONDITION: Precondition check failed.`

Then you'll need to do one of the following:
1. Create a new service account, or
2. Delete keys from the existing service account

To delete the keys from the existing service account:
```
gcloud iam service-accounts keys list --iam-account="$GCP_SERVICE_ACCOUNT_USER@$GCP_PROJECT.iam.gserviceaccount.com"
```
```
gcloud iam service-accounts keys delete 8e7508bd36cf5c7df5e144896b81a70003b43b0b --iam-account="$GCP_SERVICE_ACCOUNT_USER@$GCP_PROJECT.iam.gserviceaccount.com"
```
<br>
<br>

Base64 Encode the Credentials
```
export GCP_B64ENCODED_CREDENTIALS=$(base64 < "${GOOGLE_APPLICATION_CREDENTIALS}" | tr -d '\n')
```

<br>
<br>

### KIB an Image

#### GCP Prerequisite for KIB
Verify that your Google Cloud project does not have the Enable OS Login feature enabled.
* The Enable OS Login feature is sometimes enabled by default in GCP projects. If the OS login feature is enabled, KIB will not be able to ssh to the VM instances it creates and will not be able to successfully create an image.
* To check if it is enabled, use the commands on this page https://cloud.google.com/compute/docs/metadata/setting-custom-metadata#console_2 to inspect the metadata configured in in your project.  If you find the the enable-oslogin flag set to TRUE, you must remove (or set it to FALSE) to use KIB.
  -> Login to GCP console -> Settings -> Metadata

OR
```
gcloud compute project-info add-metadata --metadata enable-oslogin=FALSE
```
OR
```
gcloud compute instances remove-metadata INSTANCE_NAME --keys=enable-oslogin
```
<br>
<br>

```
cd ~/kib
./konvoy-image build gcp --project-id ${GCP_PROJECT} --region ${GCP_REGION} --overrides docker_overrides.yaml images/gcp/ubuntu-2004.yaml
```
To specify a network to use:
```
cd ~/kib
./konvoy-image build gcp --project-id ${GCP_PROJECT} --network ${NETWORK_NAME} --region ${GCP_REGION} --overrides docker_overrides.yaml images/gcp/ubuntu-2004.yaml
```

Collect the `googlecompute.kib_image` name for example `konvoy-ubuntu-2004-1-26-6-20230923011215`

To find a list of images in your account run
```
gcloud compute images list --no-standard-images
```



### Build the Cluster in GCP

Set variables
```
export GCP_CLUSTER_NAME=
```
```
export REGISTRY_MIRROR_URL="https://registry-1.docker.io"
export REGISTRY_MIRROR_USERNAME="dkptestdrive"
export REGISTRY_MIRROR_PASSWORD="43bfb7f5-de67-4fb0-a9a7-32dd3bdcc2a6"
```
```
export IMAGE_NAME=projects/${GCP_PROJECT}/global/images/konvoy-ubuntu-2004-1-26-6-20231027131750
```
```
export GOOGLE_APPLICATION_CREDENTIALS=${HOME}/.gcloud/credentials.json
```
Cleanup and create new bootstrap
```
dkp delete bootstrap
unset KUBECONFIG
```
```
dkp create bootstrap --with-gcp-bootstrap-credentials=true
```



Create a GCP Cluster
```
dkp create cluster gcp \
--cluster-name=${GCP_CLUSTER_NAME} \
--additional-tags=owner=mshayda,expiration=3d \
--with-gcp-bootstrap-credentials=true \
--project=${GCP_PROJECT} \
--image=${IMAGE_NAME} \
--network=${NETWORK_NAME} \
--registry-mirror-url ${REGISTRY_MIRROR_URL} \
--registry-mirror-username ${REGISTRY_MIRROR_USERNAME} \
--registry-mirror-password ${REGISTRY_MIRROR_PASSWORD} \
--dry-run -o yaml > ${GCP_CLUSTER_NAME}.yaml
```
```
kubectl create -f ${GCP_CLUSTER_NAME}.yaml
```
```
watch -n .5 dkp describe cluster -c ${GCP_CLUSTER_NAME}
```
```
dkp get kubeconfig -c ${GCP_CLUSTER_NAME} > ${GCP_CLUSTER_NAME}.conf
```
```
dkp create capi-components --kubeconfig ${GCP_CLUSTER_NAME}.conf
```
```
dkp move capi-resources --to-kubeconfig ${GCP_CLUSTER_NAME}.conf
```






Create a Managed Cluster from DKP CLI

* Create a Workspace in the Management cluster if one is not yet created
```
kubectl get workspace -A
```
Set the Workspace Name variable to the Workspace you want to attach the cluster [WORKSPACE NAMESPACE]
```
export WORKSPACE_NAMESPACE=<workspace_namespace>
```
Set the cluster name variable to a unique name that DKP will use for the cluster name in the Workspace
```
export MANAGED_CLUSTER_NAME=
```
```
dkp create cluster gcp \
--cluster-name=${GCP_CLUSTER_NAME} \
--additional-tags=owner=mshayda,expiration=3d \
--namespace ${WORKSPACE_NAMESPACE} \
--project=${GCP_PROJECT} \
--image=${IMAGE_NAME} \
--network=${NETWORK_NAME} \
--registry-mirror-url ${REGISTRY_MIRROR_URL} \
--registry-mirror-username ${REGISTRY_MIRROR_USERNAME} \
--registry-mirror-password ${REGISTRY_MIRROR_PASSWORD} \
--kubeconfig=<management-cluster-kubeconfig-path>
```




### Create a GKE Cluster from the UI 
* Note: We cannot create a GKE cluster from the DKP CLI as GKE does not have CAPI Provider for it

Create GKE Cluster from the GCP-GKE Portal
* Go to the Kubernetes Engine section of GCP
* Select to Create a new cluster
* Select Standard cluster to configure
* Name the cluster
* Select the Zone or Region
* Select `Static version` for the Control Plan Version and choose 1.26.x
* Create


### Configure GKE cluster kubeconfig to be DKP ready

#### Attach New GKE Cluster to DKP

Generate and download the kubeconfig for the new cluster
```
unset KUBECONFIG
```
```
export GKE_CLUSTER_NAME=
```
```
export KUBECONFIG=${GKE_CLUSTER_NAME}.conf
```
```
gcloud components install gke-gcloud-auth-plugin
```
```
gcloud container clusters get-credentials ${GKE_CLUSTER_NAME} --region=${GCP_REGION}
```

Get the SVC endpoint
```
kubectl get svc --kubeconfig ${GKE_CLUSTER_NAME}.conf
```

#### Create necessary service account (do for each GKE cluster)
```
kubectl -n kube-system create serviceaccount kommander-cluster-admin --kubeconfig ${GKE_CLUSTER_NAME}.conf
```

Create token secret for the serviceaccount (do for each GKE cluster)
```
kubectl -n kube-system create --kubeconfig ${GKE_CLUSTER_NAME}.conf -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: kommander-cluster-admin-sa-token
  annotations:
    kubernetes.io/service-account.name: kommander-cluster-admin
type: kubernetes.io/service-account-token
EOF
```

Verify service token is running
```
kubectl -n kube-system get secret kommander-cluster-admin-sa-token --kubeconfig ${GKE_CLUSTER_NAME}.conf -o yaml
```

Configure new service account for cluster-admin permissions (do for each GKE cluster)
```
cat << EOF | kubectl apply --kubeconfig ${GKE_CLUSTER_NAME}.conf -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kommander-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kommander-cluster-admin
  namespace: kube-system
EOF
```

#### Setup the following variables AND generate a new kubeconfig (do for each GKE cluster)
```
export USER_TOKEN_VALUE=$(kubectl --kubeconfig ${GKE_CLUSTER_NAME}.conf -n kube-system get secret/kommander-cluster-admin-sa-token -o=go-template='{{.data.token}}' | base64 --decode)
export CURRENT_CONTEXT=$(kubectl --kubeconfig ${GKE_CLUSTER_NAME}.conf config current-context)
export CURRENT_CLUSTER=$(kubectl --kubeconfig ${GKE_CLUSTER_NAME}.conf config view --raw -o=go-template='{{range .contexts}}{{if eq .name "'''${CURRENT_CONTEXT}'''"}}{{ index .context "cluster" }}{{end}}{{end}}')
export CLUSTER_CA=$(kubectl --kubeconfig ${GKE_CLUSTER_NAME}.conf config view --raw -o=go-template='{{range .clusters}}{{if eq .name "'''${CURRENT_CLUSTER}'''"}}"{{with index .cluster "certificate-authority-data" }}{{.}}{{end}}"{{ end }}{{ end }}')
export CLUSTER_SERVER=$(kubectl --kubeconfig ${GKE_CLUSTER_NAME}.conf config view --raw -o=go-template='{{range .clusters}}{{if eq .name "'''${CURRENT_CLUSTER}'''"}}{{ .cluster.server }}{{end}}{{ end }}')
```
Verify variables were set correctly
```
export -p | grep -E 'USER_TOKEN_VALUE|CURRENT_CONTEXT|CURRENT_CLUSTER|CLUSTER_CA|CLUSTER_SERVER'
```

Generate a new kubeconfig file with the environment variables just set
```
cat << EOF > kommander-cluster-admin-config-${GKE_CLUSTER_NAME}.conf
apiVersion: v1
kind: Config
current-context: ${CURRENT_CONTEXT}
contexts:
- name: ${CURRENT_CONTEXT}
  context:
    cluster: ${CURRENT_CONTEXT}
    user: kommander-cluster-admin
    namespace: kube-system
clusters:
- name: ${CURRENT_CONTEXT}
  cluster:
    certificate-authority-data: ${CLUSTER_CA}
    server: ${CLUSTER_SERVER}
users:
- name: kommander-cluster-admin
  user:
    token: ${USER_TOKEN_VALUE}
EOF
```

Verify the new config file can access the cluster (do this for each GKE cluster)
```
kubectl --kubeconfig $(pwd)/kommander-cluster-admin-config-${GKE_CLUSTER_NAME}.conf get all --all-namespaces
```

### Use `kommander-cluster-admin-config-${GKE_CLUSTER_NAME}.conf` Config File to Upload to DKP to Attach Cluster

* Create a Workspace in the Management cluster if one is not yet created
```
kubectl get workspace -A
```
Set the Workspace Name variable to the Workspace you want to attach the cluster [WORKSPACE NAMESPACE]
```
export WORKSPACE_NAME=<workspace_name>
```

```
dkp attach cluster -n ${GKE_CLUSTER_NAME} --attached-kubeconfig kommander-cluster-admin-config-${GKE_CLUSTER_NAME}.conf -w ${WORKSPACE_NAME}
```