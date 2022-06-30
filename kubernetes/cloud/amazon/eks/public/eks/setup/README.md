# Setup EKS Cluster

## Create An AWS Free Account and Its Security Credentials
Create a free account of AWS Free Tier from [here](https://aws.amazon.com/cn/free/).
Then login and click your account in the top-right corner of the page, choose "Security credentials",
switch to the "Access Keys" tab and click the "Create New Access Key" button.

## Create New DevOps Environment
```shell
# move into "kubernetes/cloud/amazon/eks" folder and then start container for current devops environment
docker run --name eks-devops -it -v ${PWD}:/work aws-eks-dev:latest

source ./setup/env
```

## Configure AWS-Cli
For more details, refer to [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config).

```shell
aws configure
# AWS Access Key ID [None]: <YOUR_ACCESS_KEY_ID>
# AWS Secret Access Key [None]: <YOUR_SECRET_ACCESS_KEY>
# Default region name [None]: <CHOSEN_REGION>
# Default output format [None]: yaml

REGION=us-west-1
CLUSTER_ID=eks-devops
VPC_STACK_NAME=${CLUSTER_ID}-vpc-stack
ROLE_NAME=${CLUSTER_ID}-role
NODE_ROLE_NAME=${CLUSTER_ID}-node-role
NODE_GROUP_ID=${CLUSTER_ID}-node-group
```

## Create VPC and Role for EKS
For more details, refer to [here](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html#eks-create-cluster)

```shell
# create VPC stack via template-url
aws cloudformation create-stack \
  --region ${REGION} \
  --stack-name ${VPC_STACK_NAME} \
  --template-url https://amazon-eks.s3.us-west-2.amazonaws.com/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml

# create eks role policy
cat << EOF > eks-role-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# create EKS cluster role and attach role policy
ROLE_ARN=$(aws iam create-role --role-name ${ROLE_NAME} --assume-role-policy-document file://eks-role-policy.json | yq .Role.Arn | sed s/\"//g)
aws iam attach-role-policy --role-name ${ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
rm -f eks-role-policy.json

# create node role and attach role policies
# 1) create node role policy
cat << EOF > node-role-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# 2) create our role for nodes
NODE_ROLE_ARN=$(aws iam create-role --role-name ${NODE_ROLE_NAME} --assume-role-policy-document file://node-role-policy.json | yq .Role.Arn | sed s/\"//g)
rm -f node-role-policy.json

# 3) attach role policies
aws iam attach-role-policy --role-name ${NODE_ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name ${NODE_ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam attach-role-policy --role-name ${NODE_ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

# get subnets and security group
aws cloudformation list-stack-resources --stack-name ${VPC_STACK_NAME} > vpc-stack.yaml

# get network environment variables
VPC_ID=$(yq '.StackResourceSummaries[] | select(.ResourceType=="AWS::EC2::VPC") | .PhysicalResourceId' vpc-stack.yaml | sed s/\"//g)
SECURITY_GROUP_IDS=$(yq '.StackResourceSummaries[] | select(.ResourceType=="AWS::EC2::SecurityGroup") | .PhysicalResourceId' vpc-stack.yaml | sed s/\"//g | tr -s "\n" ",")
NODE_SUBNET_ID=$(yq '.StackResourceSummaries[] | select(.LogicalResourceId=="PrivateSubnet01") | .PhysicalResourceId' vpc-stack.yaml | sed s/\"//g)
POD_SUBNET_IDS=$(yq '.StackResourceSummaries[] | select(.ResourceType=="AWS::EC2::Subnet" and .LogicalResourceId!="PrivateSubnet01") | .PhysicalResourceId' vpc-stack.yaml | sed s/\"//g | tr -s "\n" ",")

# confirm that all required environment variables are ready
echo "CLUSTER_ID:${CLUSTER_ID}
REGION:${REGION}
ROLE_ARN:${ROLE_ARN}
NODE_ROLE_ARN:${NODE_ROLE_ARN}
VPC_ID:${VPC_ID}
SECURITY_GROUP:${SECURITY_GROUP}
POD_SUBNET_IDS:${POD_SUBNET_IDS}
NODE_SUBNET_ID:${NODE_SUBNET_ID}"
```

## Create EKS Cluster

```shell
# create EKS cluster: https://docs.aws.amazon.com/cli/latest/reference/eks/create-cluster.html
aws eks create-cluster \
  --name ${CLUSTER_ID} \
  --region ${REGION} \
  --role-arn ${ROLE_ARN} \
  --resources-vpc-config subnetIds=${POD_SUBNET_IDS}securityGroupIds=${SECURITY_GROUP_IDS}endpointPublicAccess=true,endpointPrivateAccess=true

aws eks update-kubeconfig --name ${CLUSTER_ID} --region ${REGION}

# add nodes: https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/getting-started-console.html#eks-launch-workers
# create node group: https://docs.aws.amazon.com/cli/latest/reference/eks/create-nodegroup.html
# ec2 instances: https://aws.amazon.com/cn/ec2/instance-types/
aws eks create-nodegroup \
  --cluster-name ${CLUSTER_ID} \
  --nodegroup-name ${NODE_GROUP_ID} \
  --node-role ${NODE_ROLE_ARN} \
  --subnets ${NODE_SUBNET_ID} \
  --disk-size 10 \
  --scaling-config minSize=1,maxSize=2,desiredSize=1 \
  --instance-types t2.small

```

## Updates

```shell
# update network mode (or public access CIDR: publicAccessCidrs="203.0.113.5/32")
aws eks update-cluster-config \
    --region ${REGION} \
    --name ${CLUSTER_ID} \
    --resources-vpc-config endpointPublicAccess=true,endpointPrivateAccess=true
```

## Verifications

```shell

# list VPC stacks: https://us-west-1.console.aws.amazon.com/vpc/home
aws cloudformation list-stacks

# check eks role and attached policy
echo "ROLE_ARN:${ROLE_ARN}"
aws iam get-role --role-name ${ROLE_NAME}
aws iam list-attached-role-policies --role-name ${ROLE_NAME}

# check node role and attached policies
echo "NODE_ROLE_ARN:${NODE_ROLE_ARN}"
aws iam get-role --role-name ${NODE_ROLE_NAME}
aws iam list-attached-role-policies --role-name ${NODE_ROLE_NAME}

# check cluster status
aws eks list-clusters
aws eks describe-cluster --name ${CLUSTER_ID}

eksctl get cluster ${CLUSTER_ID}

aws eks list-nodegroups --cluster-name=${CLUSTER_ID}
aws eks describe-nodegroup --cluster-name=${CLUSTER_ID} --nodegroup-name=${NODE_GROUP_ID}

# check number of ec2 network interfaces and ips
aws ec2 describe-instance-types --filters "Name=instance-type,Values=t3.*" --query "InstanceTypes[].{Type: InstanceType, MaxENI: NetworkInfo.MaximumNetworkInterfaces, IPv4addr: NetworkInfo.Ipv4AddressesPerInterface}" --output table

```

## Cleanup

```shell
# delete cluster
aws eks delete-nodegroup --cluster-name ${CLUSTER_ID} --nodegroup-name ${NODE_GROUP_ID}
aws eks delete-cluster --name ${CLUSTER_ID}

# cleanup node role
aws iam detach-role-policy --role-name ${NODE_ROLE_NAME} --policy-arn  arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam detach-role-policy --role-name ${NODE_ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam detach-role-policy --role-name ${NODE_ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
aws iam delete-role --role-name ${NODE_ROLE_NAME}

# cleanup cluster role
aws iam detach-role-policy --role-name ${ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
aws iam delete-role --role-name ${ROLE_NAME}

# delete VPC stack
aws cloudformation delete-stack --stack-name ${VPC_STACK_NAME}

```
