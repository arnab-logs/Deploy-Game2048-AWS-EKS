# Deploying the 2048 Game on AWS EKS (with Fargate)

This guide walks you through **deploying the popular 2048 game** on an **Amazon EKS cluster** using **AWS Fargate**.  
We’ll go step-by-step — from creating the cluster to verifying the load balancer — with explanations for each command so you understand what’s happening at every stage.

---

## 1. Create an EKS Cluster

We’ll begin by creating our Kubernetes cluster using `eksctl`.

```bash
eksctl create cluster --name demo-cluster-1 --region ap-south-1 --fargate
```

<img width="2940" height="1724" alt="image" src="https://github.com/user-attachments/assets/a3ea4ca6-df3e-4a12-9689-5fe5b4695859" />

This command:

Creates an Amazon EKS cluster named demo-cluster-1.

Sets the region to ap-south-1.

Uses AWS Fargate as the compute option instead of EC2 instances.
(Fargate allows you to run pods without managing servers — AWS handles the underlying infrastructure.)


## 2. Understanding the Cluster Overview
Once the cluster is created, navigate to the EKS console.
You’ll see that the cluster overview displays an OpenID Connect Provider URL.

<img width="2286" height="1184" alt="image" src="https://github.com/user-attachments/assets/fe43815d-f3c0-4424-825e-71174e3458f3" />

This URL is essential because it allows integration with identity providers such as Okta, Keycloak, or LDAP.
That means we can create and manage users for our organization and connect multiple services under one identity system — similar to how websites offer “Login with Google,” “Login with Facebook,” or “Login with GitHub.”

AWS supports attaching an identity provider to your cluster, enabling pods to securely communicate with AWS services.

If we don’t attach an IAM identity provider and create a Kubernetes pod that needs to talk to other AWS services (like S3 or the EKS control plane), it won’t have permission to do so.
By integrating IAM roles with Kubernetes service accounts, we ensure that pods can securely access AWS APIs.



## 3. Verify Compute Type (Fargate)
Go to the Compute section in your EKS console.
You’ll notice that we’re using Fargate — not EC2 instances.

<img width="2286" height="860" alt="image" src="https://github.com/user-attachments/assets/1bf7d80e-62c6-4fcf-9769-e6fd98a56d79" />

A Fargate profile defines which pods run on Fargate.
The default profile attaches to the default and kube-system namespaces.

<img width="2246" height="416" alt="image" src="https://github.com/user-attachments/assets/d6a085d4-0a95-448c-83b5-8d9a139c9a77" />

If we want to deploy pods in a new namespace, we must create an additional Fargate profile.


## 4. Update Kubeconfig
Next, download your kubeconfig file so kubectl can communicate with your cluster.

```bash
aws eks update-kubeconfig --name demo-cluster-1 --region ap-south-1
```

<img width="2378" height="142" alt="image" src="https://github.com/user-attachments/assets/17d8b0a1-d54e-4d10-a2a4-84ea26a30f18" />

This command:

Retrieves the cluster credentials.

Updates your local kubeconfig file.

Enables you to run kubectl commands against your EKS cluster.

## 5. Create a Fargate Profile for the Game Namespace
We’ll now create a Fargate profile to deploy the 2048 game in a dedicated namespace.

```bash
eksctl create fargateprofile \
    --cluster demo-cluster-1 \
    --region ap-south-1 \
    --name alb-sample-app \
    --namespace game-2048
```

<img width="2378" height="366" alt="image" src="https://github.com/user-attachments/assets/2bebc283-20bf-4912-ba5f-929286268a71" />

Explanation:

The profile is named alb-sample-app.

It’s attached to the namespace game-2048.

This ensures that any pods created in this namespace will run on Fargate.

In the Compute section, you’ll now see that instances can be created in the following namespaces:

default

kube-system

game-2048

<img width="2248" height="500" alt="image" src="https://github.com/user-attachments/assets/8fcc5a49-680e-4cae-9696-e212d867a4dd" />

## 6. Deploy the 2048 Application
The following command applies the deployment, service, and ingress configuration in one go:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

<img width="1896" height="1618" alt="image" src="https://github.com/user-attachments/assets/9c3c5923-fbae-4b1c-826f-abe635c2c2c7" />

This manifest:

Creates the game-2048 namespace (if it doesn’t exist).

Deploys the 2048 application.

Exposes it using a service.

Defines an ingress resource that uses the ALB Ingress Controller.

<img width="2940" height="284" alt="image" src="https://github.com/user-attachments/assets/3143c869-7a05-401b-b0b8-34826f83e1ca" />

Explanation of Ingress:

The ingress class is alb.

The ALB controller reads ingress resources and sets up Application Load Balancers to route external requests to the right service (service-2048), which forwards them to the pods.

At this stage, we don’t yet have an ingress controller — so the ingress resource won’t function.

## 7. Checking the Services and Ingress

```bash
kubectl get svc -n game-2048
```

<img width="1624" height="178" alt="image" src="https://github.com/user-attachments/assets/2e7c514b-ab34-45d5-8878-330e082ce40b" />

You’ll see that the service type is NodePort and there’s no external IP.
This means only resources within the VPC can access the pods.
Our goal is to make the application externally accessible, which requires a functioning Ingress Controller.

Next, check the ingress:

```bash
kubectl get ingress -n game-2048
```

<img width="1624" height="178" alt="image" src="https://github.com/user-attachments/assets/646adffd-1383-4447-8d8c-66225c950545" />

The ingress is created but currently lacks an external address.
Once the ingress controller is deployed, it will automatically create an Application Load Balancer (ALB) and populate this address field.

## 8. Configure IAM OIDC Provider
Before installing the ALB ingress controller, we must configure an IAM OIDC provider.
Without it, the ALB controller won’t be able to interact with AWS services.

```bash
eksctl utils associate-iam-oidc-provider --cluster demo-cluster-1 --approve
```

<img width="2440" height="136" alt="image" src="https://github.com/user-attachments/assets/2cff39d6-44b0-4094-9b85-8de1dffc7feb" />

This command associates an OIDC identity provider with the cluster, enabling IAM roles to be used by Kubernetes service accounts.

## 9. Create IAM Policy for the ALB Controller
The ALB Controller needs permissions to create and manage AWS load balancers.

Download the IAM policy:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

<img width="2938" height="218" alt="image" src="https://github.com/user-attachments/assets/572e5aef-a0b2-43f8-8d5b-8a04909d7db5" />


Create the IAM policy in AWS:

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

<img width="1904" height="788" alt="image" src="https://github.com/user-attachments/assets/ff33ec48-ec6d-4e84-9966-e4fcbdf397e2" />


## 10. Create IAM Service Account
We’ll now create a Kubernetes service account for the ALB controller and attach the IAM policy to it.

```bash
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

<img width="2940" height="744" alt="image" src="https://github.com/user-attachments/assets/0046ec17-fa88-45f9-99ae-e3d3a2fff771" />

Explanation:

This command creates a service account in the kube-system namespace.

It associates the IAM role AmazonEKSLoadBalancerControllerRole with the service account.

The attached policy gives it permission to manage load balancers in AWS.

## 11. Install the ALB Controller Using Helm
We’ll use Helm (a Kubernetes package manager) to install the controller.

Add the EKS Helm repository:
```bash
helm repo add eks https://aws.github.io/eks-charts
```

<img width="1812" height="144" alt="image" src="https://github.com/user-attachments/assets/d2445b45-d28a-471b-b0d0-f14aae356e11" />

Update the repo:

```bash
helm repo update eks
```

<img width="1812" height="182" alt="image" src="https://github.com/user-attachments/assets/88945a96-9184-4743-b74e-6c2ef9a7fd4d" />

Install the controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<your-region> \
  --set vpcId=<your-vpc-id>
```

<img width="968" height="370" alt="image" src="https://github.com/user-attachments/assets/78a29f31-834e-401d-ba3a-58c499bef3c9" />

Explanation:

serviceAccount.create=false ensures Helm uses the service account we already created.

The controller will now manage ALBs for your cluster.


## 12. Verify Deployment
Confirm that the controller is running with two replicas (one in each availability zone):

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

<img width="2156" height="196" alt="image" src="https://github.com/user-attachments/assets/b4223879-f4fa-41b1-8cbc-bc4e731075d8" />

## 13. Confirm Load Balancer Creation
Go to the AWS Management Console → EC2 → Load Balancers.
You should now see a new ALB created automatically by the controller using the ingress resource.

<img width="2940" height="566" alt="image" src="https://github.com/user-attachments/assets/ac8d65fe-f831-47d3-8c7a-1d9a46aa508c" />

Check again via kubectl:

```bash
kubectl get ingress -n game-2048
```

<img width="2626" height="138" alt="image" src="https://github.com/user-attachments/assets/b2a5aead-1ee0-45fb-a328-fdc78832cc80" />

This command will now display the external address, which corresponds to the DNS name of your newly created load balancer.

<img width="2374" height="978" alt="image" src="https://github.com/user-attachments/assets/f7240b4f-d80d-4d1f-bb18-58378efe2aeb" />

## 14. Access the Game
Copy the DNS name from the load balancer and open it in your browser.

<img width="2940" height="1670" alt="image" src="https://github.com/user-attachments/assets/463eb029-1363-410c-8305-cf3006d0ab3a" />


You’ll now see the 2048 game running on your EKS cluster — deployed entirely on AWS Fargate!

<img width="2940" height="1670" alt="image" src="https://github.com/user-attachments/assets/fa8dfe16-853d-490b-856a-43a622b3718f" />



## 15. Cleanup Resources
When you’re done, delete the cluster to avoid ongoing charges:

```bash
eksctl delete cluster --name demo-cluster-1 --region ap-south-1
```

This command deletes:

The EKS cluster

All associated Fargate profiles

Load balancers, pods, and IAM roles created for the project
