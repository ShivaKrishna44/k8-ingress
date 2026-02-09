# Ingress Controller

REGION_CODE=us-east-1
CLUSTER_NAME=expense
ACC_ID=658643245214

### Permissions

* OIDC provider
```
eksctl utils associate-iam-oidc-provider \
    --region $REGION_CODE \
    --cluster $CLUSTER_NAME \
    --approve
```
* IAM Policy
```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.12.0/docs/install/iam_policy.json
```
* Create IAM Policy
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```
* Provide access to EKS through IAM Policy
```
eksctl create iamserviceaccount \
--cluster=$CLUSTER_NAME \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::$ACC_ID:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region $REGION_CODE \
--approve
```

### Install Drivers

* Add the EKS chart repo to Helm
* first install helm

curl -fsSL https://get.helm.sh/helm-v3.14.4-linux-amd64.tar.gz -o helm.tar.gz 
tar -xzf helm.tar.gz 
linux-amd64/helm 
sudo mv linux-amd64/helm /usr/local/bin/helm 
helm version 

```
helm repo add eks https://aws.github.io/eks-charts
helm repo update

```

* Install AWS Load Balancer Controller
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=$CLUSTER_NAME --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```
