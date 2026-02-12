AWS Load Balancer Controller on EKS – Complete Step-by-Step Guide
This document explains end-to-end setup of AWS Load Balancer Controller (ALB Controller) on an EKS cluster, exactly in the order followed during hands-on practice. It also explains why each step is required, which is important for interviews and production understanding.
1. What is AWS Load Balancer Controller?
AWS Load Balancer Controller is a Kubernetes controller that:
•	Watches Ingress and Service resources
•	Automatically creates and manages:
o	Application Load Balancer (ALB)
o	Target Groups
o	Listeners and rules
It integrates Kubernetes networking with AWS ALB natively.
2. Prerequisites
Before starting, ensure:
•	EKS cluster is running
•	kubectl configured and working
•	AWS CLI configured
•	eksctl installed
•	Helm installed
Verify:  kubectl get nodes  , aws sts get-caller-identity
3. Associate IAM OIDC Provider with EKS
Why this step is needed
This enables IRSA (IAM Roles for Service Accounts) so that pods can securely access AWS APIs without static credentials.
Command
eksctl utils associate-iam-oidc-provider --region us-east-1 --cluster expense-v2 --approve
Output
•	Creates an IAM OIDC identity provider
•	Required for AWS Load Balancer Controller
4. Create IAM Policy for Load Balancer Controller
Why this step is needed
The controller needs permissions to:
•	Create ALBs
•	Create listeners and rules
•	Manage target groups
•	Read EC2, ELB, IAM metadata
Policy creation:
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam-policy.json
Note: If the policy already exists, reuse it.
5. Create IAM Service Account (IRSA)
Why this step is needed ,This step links:
•	Kubernetes ServiceAccount
•	AWS IAM Role
•	IAM policy created earlier
This allows the controller pod to call AWS APIs securely.
Command
eksctl create iamserviceaccount  --cluster expense-v2  --region us-east-1 \
--namespace kube-system  --name aws-load-balancer-controller \
--attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve
What this creates
•	ServiceAccount: kube-system/aws-load-balancer-controller
•	IAM Role with trust relationship via OIDC
6. Install AWS Load Balancer Controller using Helm
Add Helm repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update
Install controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=expense-v2 \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller
Verify pods :   kubectl get pods -n kube-system | grep load-balancer
Expected: controller pods in Running state.
7. Verify CRDs Installed
The controller installs required CRDs.
kubectl get crd | grep elbv2
Examples:
•	targetgroupbindings.elbv2.k8s.aws
•	ingressclassparams.elbv2.k8s.aws
8. Deploy Application (Deployment + Service)
Deployment
•	Runs application pods
Service
•	Exposes pods internally
•	Acts as backend for Ingress
Service type used:
•	ClusterIP (recommended with ALB ingress)
9. Create Ingress Resource
Purpose 
Ingress defines:
•	Host-based routing
•	TLS termination
•	Which service receives traffic
Example annotations explained
alb.ingress.kubernetes.io/scheme: internet-facing
•	Makes ALB public
alb.ingress.kubernetes.io/target-type: ip
•	Routes traffic directly to pod IPs
alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
•	Creates HTTPS listener
alb.ingress.kubernetes.io/certificate-arn: <ACM_ARN>
•	Attaches TLS certificate
alb.ingress.kubernetes.io/group.name: shivakrishna44
•	Enables shared ALB for multiple ingresses
10. TLS Certificate (ACM)
Important rules
•	Certificate must be in same region as ALB
•	Certificate must be in ISSUED state
•	DNS validation must be completed
Verify certificate
aws acm describe-certificate  --certificate-arn <CERT_ARN>  --region us-east-1
Status must be: ISSUED
11. DNS Configuration (Route53)
Steps
1.	Domain NS must point to Route53 nameservers
2.	Create Alias record:
o	Type: A
o	Target: ALB DNS name
Verify: dig app1.example.com
12. Verify Ingress and ALB
kubectl get ingress
Expected:
•	ADDRESS populated with ALB DNS
Test:
curl https://app1.example.com
13. Common Issues Faced & Fixes
Issue: ADDRESS empty
•	Controller not running
•	IAM permissions missing
Issue: CertificateNotFound
•	ACM cert not validated
•	Wrong region
Issue: NXDOMAIN
•	NS records incorrect
•	DNS not propagated
14. Cleanup (Cost Saving)
kubectl delete ingress --all
kubectl delete svc --all
kubectl delete deploy --all

eksctl delete cluster --name expense-v2 --region us-east-1
15. Interview Summary 
"I implemented AWS Load Balancer Controller on EKS using IRSA. I associated OIDC, created IAM policies, installed the controller via Helm, and exposed applications using ALB ingress with TLS and host-based routing. I handled DNS, ACM validation, and cost optimization using shared ALBs."
What’s covered clearly
•	OIDC & IRSA (very important for interviews)
•	IAM policy + service account mapping
•	Helm installation (with exact flags and reasoning)
•	Ingress annotations explained line-by-line
•	ACM certificate issues you faced (PENDING → ISSUED)
•	DNS & Route53 mistakes + fixes
•	Shared ALB using group.name
•	Real production failure scenarios
•	Clean teardown to avoid AWS billing
ALB Debugging:
Ingress created but ADDRESS is empty
kubectl get ingress
ADDRESS   <empty>
Possible causes
•	AWS Load Balancer Controller not running
•	Wrong ingressClassName
•	Missing IAM permissions (IRSA issue)
•	Controller crashlooping
Debug steps
kubectl get pods -n kube-system | grep load-balancer
kubectl logs -n kube-system deployment/aws-load-balancer-controller
What to say in interview
“If ADDRESS is empty, I first check whether the AWS Load Balancer Controller pods are running and then inspect controller logs. Most of the time it’s either IRSA permission issues or a wrong ingress class.”
Controller running but ALB not created
Controller logs show errors like:
AccessDenied
UnauthorizedOperation
Root cause
•	IAM policy missing ELB / EC2 permissions
•	Wrong service account annotation
•	OIDC provider not associated
Debug checklist
eksctl utils associate-iam-oidc-provider --cluster <name>
kubectl describe sa aws-load-balancer-controller -n kube-system
Ensure:
annotations:   eks.amazonaws.com/role-arn: arn:aws:iam::<acc-id>:role/...
Interview-ready line
“ALB creation failures almost always trace back to IRSA. I validate the OIDC provider, service account annotations, and IAM role trust policy.”
HTTPS listener fails – CertificateNotFound
You saw this exact error 👇
CertificateNotFound: arn:aws:acm:...
Why it happens
•	Certificate in PENDING_VALIDATION
•	Certificate in wrong AWS region
•	Wrong AWS account ARN
Debug steps
aws acm describe-certificate --certificate-arn <arn>
Check:
•	Status = ISSUED
•	Region = same as ALB (us-east-1)
Interview explanation
“ALB HTTPS listeners require ACM certificates in the same region and in ISSUED state. If the cert is pending validation, the controller keeps retrying and fails.”
DNS not resolving (NXDOMAIN)
nslookup app1.example.com
NXDOMAIN
Real causes
•	Domain nameservers still pointing to old DNS provider
•	Route53 hosted zone exists but registrar not updated
•	Wrong record type (CNAME vs ALIAS)
Debug commands
dig example.com NS +short
aws route53 list-hosted-zones
Fix pattern
•	Update domain registrar NS
•	Use ALIAS A record → ALB DNS
•	Never use CNAME at root domain
Interview line
“DNS issues are usually not Kubernetes problems. I always verify authoritative name servers before debugging Route53 records.”
Ingress works internally but not externally
Symptoms:
•	Pod is healthy
•	Service works via ClusterIP
•	ALB returns 404 / timeout
Things to check
kubectl describe ingress <name>
kubectl get targetgroupbindings
Common issues:
•	Wrong target-type
•	Security group missing inbound rules
•	Health check path incorrect
Production insight
“ALB health checks must match application paths. A wrong health check silently keeps targets unhealthy.”
Multiple Ingress → Multiple ALBs (unexpected cost 💸)
Why it happens
•	Missing alb.ingress.kubernetes.io/group.name
Debug
kubectl get ingress -o yaml | grep group.name
Fix
alb.ingress.kubernetes.io/group.name: shared-alb
Interview phrasing
“In production we group multiple Ingress resources under one ALB to reduce cost and simplify SSL management.”
Git errors during infra work (you faced this too)
git push rejected (fetch first)
git pull → 500 error
Why
•	Remote repo has commits not present locally
•	Temporary GitHub backend issue
Safe approach
git fetch origin
git rebase origin/main
Interview framing:
“I prefer rebase over merge to keep infrastructure repos clean and linear.”
Cluster deletion stuck (unevictable pods)
You hit this during teardown 👇
Cause
•	DaemonSets
•	Pods with podDisruptionBudget
•	Controller pods still running
Fix
kubectl get daemonsets -A
kubectl delete daemonset aws-load-balancer-controller -n kube-system
Then retry: eksctl delete cluster
Interview angle
“Before deleting clusters, I remove system add-ons to avoid drain failures and orphaned AWS resources.”
Cost-awareness debugging (very important!)
Resources that keep charging:
•	ALB (hourly)
•	NAT Gateway
•	EKS control plane
•	Elastic IPs
Habit
kubectl get ingress
aws elbv2 describe-load-balancers
Interview-ready:
“I always verify ALBs and NAT gateways before ending a lab session to avoid unnecessary AWS costs.”
Golden Debugging Framework (memorize this)
When anything fails, follow this order:
1 Kubernetes object state :  kubectl get / describe
2 Controller logs : kubectl logs
3 AWS resource existence: (ALB, Target Groups, Certs)
4 IAM / IRSA validation
5 DNS last (never first)
How this helps you in interviews
You can confidently say:
“I’ve debugged ALB Ingress issues across IAM, ACM, DNS, and Kubernetes layers, not just YAML problems.”
