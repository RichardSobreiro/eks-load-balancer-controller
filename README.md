# eks-load-balancer-controller
Creation of a eks cluster with load balancer controller and api gateway resources and methods

## Steps
export AGW_AWS_REGION=sa-east-1
export AGW_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
export AGW_EKS_CLUSTER_NAME=EksCluster01

# 1 - Create the EC2 key pair (manual step only - Name = eks-key-pair)
# 2 - Run yml file
# 3 - Create & Associate IAM OIDC Provider for our EKS Cluster
eksctl utils associate-iam-oidc-provider --region $AGW_AWS_REGION --cluster $AGW_EKS_CLUSTER_NAME --approve
# 4 - Download the IAM policy document
curl -S https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json -o iam-policy.json
# 5 - Create an IAM policy 
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy-APIGWEksCluster01 --policy-document file://iam-policy.json 2> /dev/null

# 6 - Create a service account 
eksctl create iamserviceaccount \
  --cluster=$AGW_EKS_CLUSTER_NAME \
  --region $AGW_AWS_REGION \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --override-existing-serviceaccounts \
  --attach-policy-arn=arn:aws:iam::${AGW_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy-APIGWEksCluster01 \
  --approve
  
# 7 - Get EKS cluster VPC ID
export AGW_VPC_ID=$(aws eks describe-cluster --name $AGW_EKS_CLUSTER_NAME --region $AGW_AWS_REGION  --query "cluster.resourcesVpcConfig.vpcId" --output text)

# 8 - Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts && helm repo update
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=$AGW_EKS_CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=$AGW_VPC_ID\
  --set region=$AGW_AWS_REGION 
