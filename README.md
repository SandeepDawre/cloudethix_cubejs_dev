#CUBE.DEV INSTALLATION ON EKS 

========================================================
● TO INSTALL OR UPDATE EKSCTL ON LINUX
========================================================
	a. Download and extract the latest release of eksctl with the following command.

		curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

	b. Move the extracted binary to /usr/local/bin.

		sudo mv /tmp/eksctl /usr/local/bin

	c. Test that your installation was successful with the following command.

		eksctl version

========================================================
● INSTALL KUBECTL BINARY FOR KUBERNETES CLIENT TO COMMUNICATE WITH THE KUBERNETES API SERVER.
========================================================

		curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
		
		chmod +x ./kubectl
		
		mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
		
		echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
		
		kubectl version --client


========================================================
● CREATE FIRST EKS CLUSTER
========================================================

	---------------------------
	vim eks-cube-cluster.yaml
	---------------------------
		apiVersion: eksctl.io/v1alpha5
		kind: ClusterConfig

		metadata:
		  name: dev-atg-cube-eks
		  region: us-east-2

		nodeGroups:
		  - name: dev-atg-cube-eks-ng-01
			instanceType: t2.medium
			desiredCapacity: 2
			volumeSize: 20
			ssh:
			  allow: true # will use ~/.ssh/id_rsa.pub as the default ssh key
			  publicKeyPath: /root/atg_eks_ssh_key/id_rsa.pub

		  - name: dev-atg-cube-eks-ng-02
			instanceType: t2.small
			desiredCapacity: 2
			volumeSize: 20
			ssh:
			  publicKeyPath: /root/atg_eks_ssh_key/id_rsa.pub

========================================================
● NEXT, RUN THIS COMMAND:
========================================================

		eksctl create cluster -f eks-cube-cluster.yaml

========================================================
● TO GET THE KUBE-CONFIG TO ACCESS THE EKS CLUSTER
========================================================

		aws eks --region us-east-2 update-kubeconfig --name dev-atg-cube-eks01



========================================================
● TRY ACCESSING THE EKS CLUSTER
========================================================

	kubectl get nodes
	kubectl get pod
	kubectl get ns
	

========================================================
● TO CREATE AN IAM OIDC IDENTITY PROVIDER FOR YOUR CLUSTER WITH EKSCTL
========================================================

	1. Determine whether you have an existing IAM OIDC provider for your cluster.
	   Retrieve your cluster's OIDC provider ID and store it in a variable.
	   

		oidc_id=$(aws eks describe-cluster --name dev-atg-cube-eks --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
		

	2. Determine whether an IAM OIDC provider with your cluster's ID is already in your account.

		aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4



	3. If output is returned, then you already have an IAM OIDC provider for your cluster and you can skip the next step. 
	   If no output is returned, then you must create an IAM OIDC provider for your cluster.
	


	4. Create an IAM OIDC identity provider for your cluster with the following command. Replace dev-atg-cube-eks with your own value.

		eksctl utils associate-iam-oidc-provider --cluster dev-atg-cube-eks --approve



	5. Check again whether an IAM OIDC provider with your cluster's ID is created in your account.

		aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4



========================================================
● CREATE A SERVICE ACCOUNT POLICY, AND ROLE-BASED ACCESS CONTROL (RBAC) POLICIES
========================================================


		1. To download an IAM policy that allows the AWS Load Balancer Controller to make calls to AWS APIs on your behalf, run the following command:

			curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-loadbalancer-controller/v2.4.4/docs/install/iam_policy.json

				{
					"Policy": {
						"PolicyName": "AWSLoadBalancerControllerIAMPolicy",
						"PolicyId": "ANPAX5PDNWKDISHPUZS77",
						"Arn": "arn:aws:iam::544327185030:policy/AWSLoadBalancerControllerIAMPolicy",
						"Path": "/",
						"DefaultVersionId": "v1",
						"AttachmentCount": 0,
						"PermissionsBoundaryUsageCount": 0,
						"IsAttachable": true,
						"CreateDate": "2023-02-08T08:58:38+00:00",
						"UpdateDate": "2023-02-08T08:58:38+00:00"
					}
				}

		2. To create an IAM policy using the policy that you downloaded in step 1, run the following command:

			aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
			

		3. To create a service account named aws-load-balancer-controller in the kube-system namespace for the AWS Load Balancer Controller, run the following command:

			eksctl create iamserviceaccount \
			--cluster=dev-atg-cube-eks \
			--namespace=kube-system \
			--name=aws-load-balancer-controller \
			--attach-policy-arn=arn:aws:iam::544327185030:policy/AWSLoadBalancerControllerIAMPolicy \
			--override-existing-serviceaccounts \
			--approve


		4. To verify that the new service role is created, run one of the following commands:

			eksctl get iamserviceaccount --cluster dev-atg-cube-eks --name aws-load-balancer-controller --namespace kube-system


		   or

			kubectl get serviceaccount aws-load-balancer-controller --namespace kube-system


========================================================
● INSTALL THE AWS LOAD BALANCER CONTROLLER USING HELM   

( NOTE:- THE ALB INGRESS CONTROLLER IS NOW THE AWS LOAD BALANCER CONTROLLER) 
========================================================
		
		1. Run below command to check helm version, IF NOT INSTALL
		
			curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
			
			chmod 700 get_helm.sh
			
			./get_helm.sh
		
			helm version


		2. To add the Amazon EKS chart repo to Helm, run the following command:

			helm repo add eks https://aws.github.io/eks-charts
			
			helm repo list


		3. To install the TargetGroupBinding custom resource definitions (CRDs), run the following command:

			kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"


		4. To install the Helm chart, run the following command:

		    NOTE : HERE MAKE SURE YOU ADD EKS VPC ID

			helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
			--set clusterName=dev-atg-cube-eks \
			--set serviceAccount.create=false \
			--set region=us-east-2 \
			--set vpcId=vpc-0d99fcbea646e7a44 \
			--set serviceAccount.name=aws-load-balancer-controller \
			-n kube-system


========================================================
● AMAZON EFS CSI DYNAMIC PROVISIONING
========================================================

	1. Download the IAM policy document 
	
		curl -S https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/v1.2.0/docs/iam-policy-example.json -o iam-policy.json

	2. Create an IAM policy 
	
		aws iam create-policy \ 
		  --policy-name EFSCSIControllerIAMPolicy \ 
		  --policy-document file://iam-policy.json 

	3. Create a Kubernetes service account 
	
		eksctl create iamserviceaccount \ 
		  --cluster=dev-atg-cube-eks \  
		  --region=us-east-2 \ 
		  --namespace=kube-system \ 
		  --name=efs-csi-controller-sa \ 
		  --override-existing-serviceaccounts \ 
		  --attach-policy-arn=arn:aws:iam::<AWS account ID>:policy/EFSCSIControllerIAMPolicy \ 
		  --approve
		  
	4. To verify that the new service role is created, run one of the following commands:

		eksctl get iamserviceaccount --cluster dev-atg-cube-eks --name efs-csi-controller-sa --namespace kube-system


	   or

		kubectl get serviceaccount --namespace kube-system efs-csi-controller-sa
		
		
	
	5. Now install AWS EFS Storage Controller driver. 
	
		helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver

		helm repo update

		helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
		--namespace kube-system \
		--set image.repository=602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/aws-efs-csi-driver \
		--set controller.serviceAccount.create=false \
		--set controller.serviceAccount.name=efs-csi-controller-sa

		
	6. To verify that aws-efs-csi-driver has started, run:


		kubectl get pod -n kube-system -l "app.kubernetes.io/name=aws-efs-csi-driver,app.kubernetes.io/instance=aws-efs-csi-driver"
