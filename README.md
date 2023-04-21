# Effortlessly-Deploy-an-Application-Load-Balancer-on-AWS-EKS
Effortlessly Deploy an Application Load Balancer on AWS EKS using HELM.


If you’re looking to expose a Kubernetes deployment using a Kubernetes service and an Elastic Load Balancer (ELB), you may find yourself limited by the classic load balancer’s outdated technology. Fortunately, AWS provides a newer Application Load Balancer (ALB) with advanced features and performance benefits

I have also created a bash script to automate this whole process.

Hanz-ala1/EKS-Deployment: Simple EKS-Deployment automated script. (github.com)


To create an Application Load Balancer, we will have to provision:

EKS Cluster

IAM OIDC ( OpenID Connect )

AWS Load Balancer Controller

IAM Policies

Ingress Manifests


Requirements:

Eksctl CLI

<img width="876" alt="Screenshot 2023-04-21 at 22 09 52" src="https://user-images.githubusercontent.com/67044030/233742829-3c2398e1-2627-442d-a758-a35c8ed9057d.png">

Kubectl CLI

<img width="856" alt="Screenshot 2023-04-21 at 22 12 22" src="https://user-images.githubusercontent.com/67044030/233743027-07e29497-b408-4700-8c7d-55a06689615b.png">

Helm CLI

AWS CLI

Sufficient Admin Permissions for the AWS account


The AWS Load Balancer Controller allows you to use an ALB or NLB with your Kubernetes services. By creating a Kubernetes ingress resource and associating it with a service, the controller automatically creates and manages an ALB or NLB for you. This allows you to take advantage of the advanced features and performance benefits of the ALB or NLB while still using Kubernetes services to manage your backend pods.

Enter the following command below:

eksctl create cluster --name demo --region us-east-1

You can change the name “demo” and region “us-east-1” to your own preferences.

Within that one command you shall create an entire cluster. This can take up to 10 minutes.

Once the cluster has created use this command below to update your ./kubeconfig

aws eks update-kubeconfig --region us-east-1 --name demo
You should be able to now access your cluster. Try the following below to see your worker nodes.

kubectl get nodes

Now that the cluster is active, we need to setup the IAM Policies for the Load Balancer Controller.

Download an IAM policy for the AWS Load Balancer Controller that allows it to make calls to AWS APIs on your behalf.

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/install/iam_policy.json

Create the IAM policy

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
    
We also need an IAM OIDC because it provides a solution by allowing the EKS clusters to use OIDC for authentication and authorization of Kubernetes resources.

This means that EKS clusters can leverage existing IAM policies and roles to control access to Kubernetes resources, making it easier to manage access to Kubernetes clusters and resources in a secure and scalable manner.

To create/associate the IAM-OIDC Provider. (change “us-east-1” to your own region & “demo” the name of your cluster)
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster demo \
    --approve
    
Now we will take our IAM policy and create an IAM role. Using this role attach a service account name for the load balancer controller.

Replace my-cluster & 111122223333 with your information. check using “kubectl config view”

eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
  
  
Now we can install the AWS Load Balancer controller using HELM

Helm is a tool that helps with the deployment and management of applications on Kubernetes by using pre-configured “charts” that define application dependencies and resources.

Add the eks-charts repository

helm repo add eks https://aws.github.io/eks-charts

Update the repo

helm repo update


Replace my-cluster with your cluster name. This shall install it.

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=Dev\
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller 
  
  
To check the installation, view the deployment

kubectl get deployment -n kube-system aws-load-balancer-controller

This what it should look like :


Install a basic app using Application Load Balancer.

Using the ingress, we will define a define deployment to use the newly created load balancer controller to provision ALB for our app.

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/examples/2048/2048_full.yaml
Open your App. By copying the URL and pasting it into your browser.


What it should look like:


Please remember to delete cluster

 eksctl delete cluster --region=us-east-1 --name=Dev
