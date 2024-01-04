# Three-tier-architecture-Kubernetes

**Step 1: IAM User Creation in AWS**

1.	Log in to the AWS console using your credentials.
2.	In the search bar, enter ‘IAM’ to access the IAM Dashboard.
3.	Navigate to the ‘Users’ section and select ‘Create User’.
4.	Enter a Name, Check the Desired Options, and Proceed to Next Step
5.	Attach Policies, select Administrator access
6.	Click next and create user.
7.	Select View User to Access User Details
8.	Access Security Credentials
9.	Now, within security credentials, navigate to Access keys and proceed to Create a new access key.
10.	Download the .csv File and Click ‘Done’
    
**Step2: Create EC2 Instance**

1. Sign in to AWS Console:
— Log in to your AWS Management Console.
2. Navigate to EC2 Dashboard:
— Access the EC2 Dashboard by selecting “Services” in the top menu.
— Choose “EC2” under the Compute section.
3. Launch Instance:
— Click on the “Launch Instance” button to initiate the creation process.
4. Choose an Amazon Machine Image (AMI):
— Select a suitable AMI (e.g., Ubuntu) for your instance.
5. Choose an Instance Type:
— In the “Choose Instance Type” step, opt for t2.medium.
— Proceed by clicking “Next: Configure Instance Details.”
1.	Configure Instance Details:
— Set “Number of Instances” to 1 (adjust if necessary).
— Configure additional settings such as network, subnets, IAM role, etc.
— For “Storage,” add a new volume and set the size to 8GB (or modify existing storage to 16GB).
— Click “Next: Add Tags” when configuration is complete.
7. Add Tags (Optional):
— Optionally, add tags to organize your instance.
8. Configure Security Group:
— Choose an existing security group or create a new one.
— Ensure the security group has necessary inbound/outbound rules for required access.
9. Review and Launch:
— Review the configuration details to ensure they are as desired.
10. Select Key Pair:
— Choose “Choose an existing key pair” from the dropdown.
— Acknowledge access to the selected private key file.
11. Launch Instances:
— Click “Launch Instances” to create the EC2 instance.
12. Access the EC2 Instance:
— Once the instance is launched, access it using the selected key pair and the instance’s public IP or DNS.

**Step3: Connect to Instance and Install Required Packages**

**Install AWS CLI:**
•	curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -"awscliv.zip"
•	unzip -q awscli.zip
•	./aws/install
•	aws –version
•	aws configure – We need to configure all necessary information from the access key

**Install Kubectl:**
•	curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.3/2023-11-14/bin/linux/amd64/kubectl
•	sudo chmod +x ./kubectl
•	mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
•	kubectl version --client

**Install Eksctl:**
•	curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
•	sudo mv /tmp/eksctl /usr/local/bin
•	eksctl version

**Install Helm:**
•	curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
•	chmod 700 get_helm.sh
•	./get_helm.sh

**Step4: EKS Setup**

1.	Clone the GitHub Repository: A Step-by-Step Guide
git clone https://github.com/mudit097/three-tier-architecture-demo.git
cd 3TierDB

2.	Establish Cluster
eksctl create cluster --name demo-cluster-three-tier-1 --region us-east-1
export cluster_name=<CLUSTER-NAME>
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve

3.	Setting Up ALB Add-On:
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

4.	Create IAM Role with Cluster Name and AWS Account ID
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
5. Implement ALB Controller
  Add Helm Repository for Deployment
  helm repo add eks https://aws.github.io/eks-charts
  Repository Refresh: Latest Updates
  helm repo update eks
  Update the VPC_ID in the following command after retrieving the VPC ID from EKS
  helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=demo-cluster-three-tier-1 --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=us-east-1 --set vpcId=<vpc-id>
  Ensure Operational Deployment Success
  kubectl get deployment -n kube-system aws-load-balancer-controller

6.	EBS CSI Plugin Setup and Configuration
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster <YOUR-CLUSTER-NAME> \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
  Execute the following command, replacing ‘YOUR_CLUSTER_NAME’ with the actual name of your cluster and ‘YOUR_ACCOUNT_ID’ with your account ID.
  eksctl create addon --name aws-ebs-csi-driver --cluster <YOUR-CLUSTER-NAME> --service-account-role-arn arn:aws:iam::<AWS-ACCOUNT-ID>:role/AmazonEKS_EBS_CSI_DriverRole –force

8. Navigate into the Helm and Establish a New Namespace
  cd helm
  kubectl create ns robot-shop
  helm install robot-shop --namespace robot-shop .

10. Time for Pod Check
  kubectl get pods -n robot-shop

11. Check service
  kubectl get svc -n robot-shop
  
12. Now Accepting Ingress Applications
  kubectl apply -f ingress.yaml
  
13. Navigate to AWS Console, Locate EC2, and Access Load Balancers — Copy DNS
  k8s-robotsho-robotsho-55094ff83e-535495866.us-east-1.elb.amazonaws.com

**Step5: DELETE CLUSTER**

eksctl delete cluster --name demo-cluster-three-tier-1 --region us-east-1
