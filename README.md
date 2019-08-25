# k8s-aws-kops
Deploy private k8s cluster using kops on AWS

## Pre-requisites
- Development instance
- AWS Account
- Install aws-cli on development instance
- Setup aws creds on development instance 
-- https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html#cli-quick-configuration

## Defaults
- VPC CIDR : 10.240.0.0/16
- AWS Region : eu-west-1
- k8s Cluster Name : tes-cluster.k8s.local ## For a gossip-based cluster, make sure the name ends with k8s.local
- s3 bucket name : k8s-test-state-store-ubuntu 	## make sure to use a unique bucket name
- k8s version : 1.13.0

## Environment Variables
```
export NODE_SIZE=${NODE_SIZE:-t2.medium}
export MASTER_SIZE=${MASTER_SIZE:-t2.medium}
export ZONES=${ZONES:-"eu-west-1a,eu-west-1b,eu-west-1c"}
export KOPS_CLUSTER_NAME=${KOPS_CLUSTER_NAME:-"test-cluster.k8s.local"}
export KOPS_STATE_STORE="s3://k8s-test-state-store-ubuntu"		#replace it with your s3 bucket name
```

## Steps
#### Install kops v1.13.0
```
wget https://github.com/kubernetes/kops/releases/download/1.13.0/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
```

#### Create ssh keygen with empty passphrase
```
ssh-keygen -t rsa -C "k8s" -f ~/.ssh/k8s -P ""
```

#### Create AWS VPC to deploy k8s cluster
```
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.240.0.0/16 --amazon-provided-ipv6-cidr-block --query 'Vpc.VpcId' --output text)
aws ec2 create-tags --resources ${VPC_ID} --tag Key=Name,Value=k8s-test
export VPC_ID=${VPC_ID}
```

#### Create and Attach Internet Gateway to newly created VPC
```
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id ${IGW_ID} --vpc-id ${VPC_ID}
```

#### Create s3 bucket to store k8s state
```
aws s3 mb s3://k8s-test-state-store-ubuntu		#replace it with your bucket name
```

#### Create private hosted zone
```
HOSTED_ZONE=$(aws route53 create-hosted-zone --name cluster.k8s.local --caller-reference $(date +"%D:%T") --vpc VPCRegion="eu-west-1",VPCId=${VPC_ID} --query 'HostedZone.Id' --output text)
HOSTED_ZONE=$(echo $HOSTED_ZONE | cut -d'/' -f3)
```

#### Cluster Creation
```
kops create cluster \
  --node-count 3 \
  --master-count 3 \
  --zones ${ZONES} \
  --master-zones ${ZONES} \
  --master-volume-size=100 \
  --vpc ${VPC_ID} \
  --network-cidr 10.240.0.0/16 \
  --node-size ${NODE_SIZE} \
  --master-size ${MASTER_SIZE} \
  --node-volume-size=100 \
  --kubernetes-version="1.13.0" \
  --cloud-labels "Creator=Kops,CostCenter=PROD" \
  --state=${KOPS_STATE_STORE} \
  --authorization RBAC \
  --networking weave \
  --topology private \
  --ssh-public-key ~/.ssh/k8s.pub \
  --name=${KOPS_CLUSTER_NAME} \
  --output=yaml
```

Above command will produce lot of console output mentioning what all objects kops will create, ending with something like this
```
Must specify --yes to apply changes

Cluster configuration has been created.

Suggestions:
 * list clusters with: kops get cluster
 * edit this cluster with: kops edit cluster test-cluster.k8s.local
 * edit your node instance group: kops edit ig --name=test-cluster.k8s.local nodes
 * edit your master instance group: kops edit ig --name=test-cluster.k8s.local master-eu-west-1a

Finally configure your cluster with: kops update cluster --name test-cluster.k8s.local --yes
```
**NOTE :** Above command will not create actual k8s cluster but only manifest file.

#### Edit cluster manifest to make it secure
```
kops edit cluster --name test-cluster.k8s.local
```
- Above command will open k8s manifest file. Now edit below config options to access your k8s cluster from your network only.
```
  kubernetesApiAccess:
  - 0.0.0.0/0           ### Delete this entry and add your networks public ip's

  sshAccess:
  - 0.0.0.0/0           ### Delete this entry and add your networks public ip's
```

#### Edit Node Instance Group
```
kops edit ig --name=test-cluster.k8s.local nodes
```
- Above command will open node instance manifest file. Add below 3 lines under 'spec' section. 
```
	cloudLabels:
      k8s.io/cluster-autoscaler/test-cluster.k8s.local: "true"
      k8s.io/cluster-autoscaler/enabled: "true"
```
**NOTE :** This is required to locate your node auto-scaling object by k8s Cluster Autoscaler

