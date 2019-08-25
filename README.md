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
ssh-keygen -t rsa -C "k8s" -f ${HOME}/.ssh/k8s -P ""
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

#### Configure bastion server to ssh into your cluster nodes
```
kops create instancegroup bastions --role Bastion --subnet utility-eu-west-1a
```
- Above command will open a new manifest file in you default editor with details of bastion instance group. Save and exit.

#### Update k8s cluster
- This command will create all necessary objects for your cluster in AWS
```
kops update cluster --name test-cluster.k8s.local --yes
```

- Command will take around 10minutes to finish. It should end up with console output something like this :
```
kops has set your kubectl context to test-cluster.k8s.local

Cluster is starting.  It should be ready in a few minutes.

Suggestions:
 * validate cluster: kops validate cluster
 * list nodes: kubectl get nodes --show-labels
 * to ssh to the bastion, you probably want to configure a bastionPublicName.
 * the admin user is specific to Debian. If not using Debian please use the appropriate user based on your OS.
 * read about installing addons at: https://github.com/kubernetes/kops/blob/master/docs/addons.md.
```

#### Update bastion instance security group to ssh into instance
```
BASTION_SG_1=$(aws ec2 describe-security-groups --filter Name=vpc-id,Values=${VPC_ID} Name=group-name,Values=bastion-elb.test-cluster.k8s.local --query 'SecurityGroups[*].[GroupId]' --output text)
BASTION_SG_2=$(aws ec2 describe-security-groups --filter Name=vpc-id,Values=${VPC_ID} Name=group-name,Values=bastion.test-cluster.k8s.local --query 'SecurityGroups[*].[GroupId]' --output text)

-- Get bastion instance public ip
BASTION_PUB_IP=$(aws ec2 describe-instances --filter "Name=tag:Name,Values=bastions.test-cluster.k8s.local" --query 'Reservations[].Instances[].PublicIpAddress' --output text)
BASTION_ID=$(aws ec2 describe-instances --filter "Name=tag:Name,Values=bastions.test-cluster.k8s.local" --query 'Reservations[].Instances[].InstanceId' --output text)

-- Attach BASTION_SG_ID to BASTION_ID
aws --region eu-west-1 ec2 modify-instance-attribute --instance-id ${BASTION_ID} --groups ${BASTION_SG_1} ${BASTION_SG_2}
```

#### Copy ssh keys used to brought up our k8s cluster to bastion instance
```
scp -i ${HOME}/.ssh/k8s ${HOME}/.ssh/k8s admin@${BASTION_PUB_IP}:~/.ssh/.
```
- Check if we can ssh into bastion instance.
```
ssh -i ${HOME}/.ssh/k8s admin@${BASTION_PUB_IP}
```
- Exit ssh session

#### Validate Cluster
- Time to check validity of cluster
```
kops validate cluster
```
- If cluster setup is completed by kops, you should see console output something like this.
```
Validating cluster test-cluster.k8s.local

INSTANCE GROUPS
NAME                    ROLE    MACHINETYPE     MIN     MAX     SUBNETS
bastions                Bastion t2.micro        1       1       utility-eu-west-1a
master-eu-west-1a       Master  t2.medium       1       1       eu-west-1a
master-eu-west-1b       Master  t2.medium       1       1       eu-west-1b
master-eu-west-1c       Master  t2.medium       1       1       eu-west-1c
nodes                   Node    t2.medium       3       3       eu-west-1a,eu-west-1b,eu-west-1c

NODE STATUS
NAME                                            ROLE    READY
ip-10-240-115-201.eu-west-1.compute.internal    node    True
ip-10-240-121-250.eu-west-1.compute.internal    master  True
ip-10-240-36-14.eu-west-1.compute.internal      node    True
ip-10-240-45-220.eu-west-1.compute.internal     master  True
ip-10-240-80-231.eu-west-1.compute.internal     node    True
ip-10-240-94-126.eu-west-1.compute.internal     master  True

Your cluster test-cluster.k8s.local is ready
```
**NOTE :** Do not proceed untill this command is successful

#### Deploy Cluster Autoscaler
```
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```
