# Install eksctl
https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html

To install or upgrade eksctl on Linux using curl

1. Download and extract the latest release of eksctl with the following command.
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```
2. Move the extracted binary to /usr/local/bin.
```
sudo mv /tmp/eksctl /usr/local/bin
```
3. Test that your installation was successful with the following command.
```
eksctl version
```
> Note:  
The GitTag version should be at least 0.47.0. If not, check your terminal output for any installation or upgrade errors, or replace the address in step 1 with https://github.com/weaveworks/eksctl/releases/download/0.47.0/eksctl_Linux_amd64.tar.gz and complete steps 1-3 again.

# Install AWS CLI
https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html

For the latest version of the AWS CLI, use the following command block:

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

# Configure AWS CLI
https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-format
```
aws configure
```
> You will need an Access Key and Access Key Secret for this step
```
AWS Access Key ID [None]: ********************
AWS Secret Access Key [None]: ****************************************
Default region name [None]: **-****-*
Default output format [None]: json
```

## Create a private key
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#prepare-key-pair
> I created a key on Amazon EC2

## Some environment variables which may be helpful to set
```
export eks_account_id=************
export eks_cluster_name=*************
export eks_private_key=************
export eks_region=**-****-*
export eks_lbc_version="v2.0.0"
export eks_node_group=ng-********
export eks_vpc_id=vpc-*****************
```

## Authenticate Docker
Authenticate Docker to an Amazon ECR private registry with get-login-password
```
aws ecr get-login-password --region ${eks_region} | docker login --username AWS --password-stdin ${eks_account_id}.dkr.ecr.${eks_region}.amazonaws.com
```

# Create cluster

https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html
## Create cluster
```
eksctl create cluster \
--name ${eks_cluster_name} \
--region ${eks_region} \
--with-oidc \
--ssh-access \
--ssh-public-key ${eks_private_key} \
--node-type t3.medium \
--managed
```
> This took 13 minutes to complete for me

## Change min/max/desired node group sizes
https://console.aws.amazon.com/eks/home?region=${eks_region}#/clusters/${eks_cluster_name}/nodegroups/${eks_node_group}

## Add a node
https://eksctl.io/usage/managing-nodegroups/#scaling
```
eksctl scale nodegroup --cluster=${eks_cluster_name} --nodes=3 ${eks_node_group}
```

# Push image to Amazon ECR private registry
## Tag your image
Tag your image with the Amazon ECR registry, repository, and optional image tag name
```
docker tag <image-tag> ${eks_account_id}.dkr.ecr.${eks_region}.amazonaws.com/<image-name>
```
## Push the image
Push the image using the docker push command:
```
docker push ${eks_account_id}.dkr.ecr.${eks_region}.amazonaws.com/<image-name>
```

# Deploy the AWS Load Balancer (Ingress) Controller
https://www.eksworkshop.com/beginner/130_exposing-service/ingress_controller_alb/

## Verify if the AWS Load Balancer Controller version has been set

```
if [ ! -x ${eks_lbc_version} ]
then
echo '${eks_lbc_version} has been set.'
else
echo '${eks_lbc_version} has NOT been set.'
fi
```
If the result is ${eks_lbc_version} has NOT been set, go here - https://www.eksworkshop.com/020_prerequisites/k8stools/#set-the-aws-load-balancer-controller-version - for instructions on setting it.

## Create IAM OIDC provider
```
eksctl utils associate-iam-oidc-provider \
--region ${eks_region} \
--cluster ${eks_cluster_name} \
--approve
```

## Create a policy called AWSLoadBalancerControllerIAMPolicy
```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```
```
aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam-policy.json
```
Response:
```
{
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
        "PolicyId": "*********************",
        "Arn": "arn:aws:iam::************:policy/AWSLoadBalancerControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2021-05-20T14:54:52+00:00",
        "UpdateDate": "2021-05-20T14:54:52+00:00"
    }
}
```

## Create a IAM role and ServiceAccount
```
eksctl create iamserviceaccount \
--cluster ${eks_cluster_name} \
--namespace kube-system \
--name aws-load-balancer-controller \
--attach-policy-arn arn:aws:iam::${eks_account_id}:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--approve
```

## Install the TargetGroupBinding CRDs
```
kubectl apply -k github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master
```
```
kubectl get crd
```

##  Tag subnets
https://console.aws.amazon.com/vpc/home?region=${eks_region}#subnets:

|key|value|note|
|---|---|---|
|kubernetes.io/cluster/[cluster-name]|owned (or shared)| Replace [cluster-name] with your cluster name.|
|kubernetes.io/role/internal-elb|1||

## Deploy the Helm chart
The helm chart will deploy from the eks repo
```
helm repo add eks https://aws.github.io/eks-charts
```
```
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=${eks_cluster_name} \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller \
--set region=${eks_region}
```
```
kubectl -n kube-system rollout status deployment aws-load-balancer-controller
```

## Sample deployment
```
curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/examples/2048/2048_full.yaml \
| sed 's#replicas: 5#replicas: 1#g' \
| kubectl apply -f -
```

## Setting up persistent storage
https://docs.aws.amazon.com/eks/latest/userguide/storage-classes.html


# Run a temp busybox pod
```
kubectl run -it temp --image=busybox /bin/sh
```

# Cleaning up
To delete the resources 

```
curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/examples/2048/2048_full.yaml \
| sed 's=alb.ingress.kubernetes.io/target-type: ip=alb.ingress.kubernetes.io/target-type: instance=g' \
| kubectl delete -f -
```
```
helm uninstall aws-load-balancer-controller \
-n kube-system
```
```
kubectl delete -k github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master
```
```
eksctl delete iamserviceaccount \
--cluster ${eks_cluster_name} \
--name aws-load-balancer-controller \
--namespace kube-system \
--wait
```
```
aws iam delete-policy \
--policy-arn arn:aws:iam::${eks_account_id}:policy/AWSLoadBalancerControllerIAMPolicy
```

# Delete the cluster and nodes
## Stop all helm charts
```
helm uninstall $(helm list -q)
```
## Delete all PVCs
```
kubectl delete pvc $(kubectl get pvc --template '{{range .items}}{{.metadata.name}}{{" "}}{{end}}')
```
### Make sure the PVs have been deleted as well
```
kubectl get pv
```
## Delete the cluster
```
eksctl delete cluster --name ${eks_cluster_name} --region ${eks_region}
```

# Sample cluster creation
## Set up the cluster
```
export eks_account_id=************
export eks_cluster_name=*********
export eks_private_key=************
export eks_region=**-****-*
export eks_lbc_version="v2.0.0"
export eks_node_group=ng-********
export eks_vpc_id=vpc-*****************
```
```
eksctl create cluster --name ${eks_cluster_name} --region ${eks_region} --with-oidc --ssh-access --ssh-public-key ${eks_private_key} --node-type t3.medium --managed
```
> https://console.aws.amazon.com/eks/home?region=${eks_region}#/clusters/${eks_cluster_name}/nodegroups/${eks_node_group}

```
eksctl utils associate-iam-oidc-provider --region ${eks_region} --cluster ${eks_cluster_name} --approve
```
```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```
```
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam-policy.json
```
```
eksctl create iamserviceaccount --cluster ${eks_cluster_name} --namespace kube-system --name aws-load-balancer-controller --attach-policy-arn arn:aws:iam::${eks_account_id}:policy/AWSLoadBalancerControllerIAMPolicy --override-existing-serviceaccounts --approve
```
```
kubectl apply -k github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master
```
Test crd was created
```
kubectl get crd
```

Got to this URL and enter the following 2 tags  
https://console.aws.amazon.com/vpc/home?region=${eks_region}#subnet:  

|tag|value|
|---|---|
|kubernetes.io/cluster/${cluster_name}|owned|
|kubernetes.io/role/internal-elb|1|

```
helm repo add eks https://aws.github.io/eks-charts
```
```
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=${eks_cluster_name} --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=${eks_region}
```
```
kubectl -n kube-system rollout status deployment aws-load-balancer-controller
```

## Install applications...
