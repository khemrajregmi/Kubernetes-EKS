
# Prerequisites

kubectl – A command line tool for working with Kubernetes clusters. For more information, see [Installing or updating kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html).

eksctl – A command line tool for working with EKS clusters that automates many individual tasks. For more information, see [Installing or updating eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html).

AWS CLI – A command line tool for working with AWS services, including Amazon EKS. For more information, see [Installing, updating, and uninstalling the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) in the AWS Command Line Interface User Guide. After installing the AWS CLI, we recommend that you also configure it. For more information, see [Quick configuration with aws configure](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config) in the AWS Command Line Interface User Guide.


# Install EKS

Please follow the prerequisites doc before this.

## Install using Fargate

```
eksctl create cluster --name demo-cluster --region eu-central-1 --fargate
```

## Delete the cluster

```
eksctl delete cluster --name demo-cluster --region eu-central-1
```


# 2048 App

## Create Fargate profile

```
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region eu-central-1 \
    --name alb-sample-app \
    --namespace game-2048
```

## Deploy the deployment, service and Ingress

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

we don't have ingress controller yet lets check

``` 
kubectl get pods -n game-2048
```

lets watch the pods with the command below and you can quite ctrl + c to quite after you watch

``` kubectl get pods -n game-2048 -w ```

you can also serch for the services

``` kubectl get svc -n game-2048 ```

screenshots

As you see now have cluster IP and type ins NodePort but there is no external IP that means anybody within the AWS VPC or anybody who has access to VPC they talk to this pod using the
IP address followed by Port but our goal is make availabe to outside the aws or some who is your customer should access this , so for that we have created the ingress right ?
Check with the follwoing command

```
kubectl get ingress -n game-2048
```
Once we deply the ingress controller we can see the address here , then every one can access from the outside the world

Screenshot
Next steps will be to create ingress controller that ingress controller will read the resource called ingress-2048 , and it will create a loadbalancer for us
And we also need to create IAM OIDC provider

# commands to configure IAM OIDC provider 

```
export cluster_name=demo-cluster
```

```
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5) 
```

## Check if there is an IAM OIDC provider configured already

- aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4\n 

If not, run the below command

```
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

Now ALB controller ->simply known as Pod -> Access to AWS services such as ALB

# How to setup alb add on

Download IAM policy

```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

Create IAM Policy

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

Create IAM Role

```
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

## Deploy ALB controller
Make sure you have install help befor you procced in mac follow this command
```
brew install helm
```

Add helm repo

```
helm repo add eks https://aws.github.io/eks-charts
```
This Helm chart will create the actual controller and it will use this service account for running the pod
Update the repo

```
helm repo update eks
```

Install helm chart

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```

Verify that the deployments are running.

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Screenshot

you can do debugging with
```
kubectl edit deploy/aws-load-balancer-controller -n kube-system
```

fix the issue if there is

else run the command 

```
kubectl get ingress -n game-2048
```

Now you will able to see the address

Conclution This address is the load balancer that the ingress controller has created watching this ingress resource. So ingress controller will watch for the ingress resource configuration provided in ingress resource and create a load balancer

For each micro service it is the responsiblity of devops engineer to write a pod , deployment.yml service.yml and 1 time responsiblity to create the ingress controller


Enjoy !! 





![Screenshot 2023-08-03 at 7 57 15 PM](later)
