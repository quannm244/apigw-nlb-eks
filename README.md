# apigw-nlb-eks

Deploy EKS cluster with API Gateway and private integration with NLB to expose a sample Hello World

# 1. Setup environment
## a. Install kubectl (v1.20)
```
sudo curl --silent --location -o /usr/local/bin/kubectl https://dl.k8s.io/release/v1.20.0/bin/linux/amd64/kubectl
sudo chmod +x /usr/local/bin/kubectl
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

## b. Install eksctl
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

- Remove credentials (optional for Cloud9)
```
rm -vf ${HOME}/.aws/credentials
```

## c. Uninstall awscli v1 if present and install awscli v2
```
export AWS_PATH=$(which aws)
sudo rm -rf $AWS_PATH
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin
aws --version
rm awscliv2.zip
rm -rf aws
```

## d. Install helm
```
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)
```

# 2. Create EKS cluster, deployment and service
## a. Create EKS cluster
```
eksctl create cluster -f cluster.yaml
```
- Export env variable for later use, replace <cluster_name> as in cluster.yaml file 
```
export AGW_AWS_REGION=us-east-2
export AGW_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
export AGW_EKS_CLUSTER_NAME=<cluster_name>
```
- Add user to master group to see EKS resources (optional)
```
eksctl create iamidentitymapping \
  --cluster $AGW_EKS_CLUSTER_NAME \
  --region $AGW_AWS_REGION \
  --arn arn:aws:iam::$AGW_ACCOUNT_ID:user/<user_name> \
  --username <user_name> \
  --group system:masters
```

## b. Deploy AWS Load Balancer Controller
- Associate OIDC provider:
```
eksctl utils associate-iam-oidc-provider \
  --region $AGW_AWS_REGION \
  --cluster $AGW_EKS_CLUSTER_NAME \
  --approve
```

- Download the IAM policy document for AWS Load Balancer Controller:
```
curl -S https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json -o iam-policy.json
```

- Create an IAM policy for AWS Load Balancer Controller:
```
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy-APIGW \
  --policy-document file://iam-policy.json 2> /dev/null
```

- Create a service account for AWS Load Balancer Controller:
```
eksctl create iamserviceaccount \
  --cluster=$AGW_EKS_CLUSTER_NAME \
  --region=$AGW_AWS_REGION \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --override-existing-serviceaccounts \
  --attach-policy-arn=arn:aws:iam::${AGW_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy-APIGW \
  --approve
```

- Get EKS cluster VPC ID:
```
export AGW_VPC_ID=$(aws eks describe-cluster \
  --name $AGW_EKS_CLUSTER_NAME \
  --region $AGW_AWS_REGION  \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text)
```

- Install AWS Load Balancer Controller using helm:
```
helm repo add eks https://aws.github.io/eks-charts && helm repo update
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=$AGW_EKS_CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=$AGW_VPC_ID\
  --set region=$AGW_AWS_REGION
```

## c. Deploy ACK Controller for API Gateway

- Download policy file for ACK:
```
curl -O https://raw.githubusercontent.com/aws-samples/amazon-apigateway-ingress-controller-blog/Mainline/apigw-ingress-controller-blog/ack-iam-policy.json
```

- Create IAM policy from file above:
```
aws iam create-policy \
  --policy-name ACKIAMPolicy \
  --policy-document file://ack-iam-policy.json
```

- Create IAM service account for ACK:
```
eksctl create iamserviceaccount \
  --attach-policy-arn=arn:aws:iam::${AGW_ACCOUNT_ID}:policy/ACKIAMPolicy \
  --cluster=$AGW_EKS_CLUSTER_NAME \
  --namespace=kube-system \
  --name=ack-apigatewayv2-controller \
  --override-existing-serviceaccounts \
  --region $AGW_AWS_REGION \
  --approve
```

- Install ACK using helm: 

```
export HELM_EXPERIMENTAL_OCI=1
export SERVICE=apigatewayv2
export RELEASE_VERSION=v0.0.2
export CHART_EXPORT_PATH=/tmp/chart
export CHART_REPO=public.ecr.aws/aws-controllers-k8s/chart
export CHART_REF=$CHART_REPO:$SERVICE-$RELEASE_VERSION
mkdir -p $CHART_EXPORT_PATH

helm chart pull $CHART_REF
helm chart export $CHART_REF --destination $CHART_EXPORT_PATH

helm install ack-${SERVICE}-controller \
  $CHART_EXPORT_PATH/ack-${SERVICE}-controller \
  --namespace kube-system \
  --set serviceAccount.create=false \
  --set aws.region=$AGW_AWS_REGION
```

## d. Deploy application and service:
```
kubectl apply -f helloworld-app.yaml
```

## e. Create VPC Link
- Create SG for VPC Link
```
AGW_VPCLINK_SG=$(aws ec2 create-security-group \
  --description "SG for VPC Link" \
  --group-name SG_VPC_LINK \
  --vpc-id $AGW_VPC_ID \
  --region $AGW_AWS_REGION \
  --output text \
  --query 'GroupId')
```

- Create VPC Link for internal NLB
```
cat > vpclink.yaml<<EOF
apiVersion: apigatewayv2.services.k8s.aws/v1alpha1
kind: VPCLink
metadata:
  name: nlb-internal
spec:
  name: nlb-internal
  securityGroupIDs: 
    - $AGW_VPCLINK_SG
  subnetIDs: 
    - $(aws ec2 describe-subnets \
          --filter Name=tag:kubernetes.io/role/internal-elb,Values=1 \
          --query 'Subnets[0].SubnetId' \
          --region $AGW_AWS_REGION --output text)
    - $(aws ec2 describe-subnets \
          --filter Name=tag:kubernetes.io/role/internal-elb,Values=1 \
          --query 'Subnets[1].SubnetId' \
          --region $AGW_AWS_REGION --output text)
    - $(aws ec2 describe-subnets \
          --filter Name=tag:kubernetes.io/role/internal-elb,Values=1 \
          --query 'Subnets[2].SubnetId' \
          --region $AGW_AWS_REGION --output text)
EOF
kubectl apply -f vpclink.yaml
```

- Check if VPC Link is AVAILABLE:
```
aws apigatewayv2 get-vpc-links --region $AGW_AWS_REGION
```

- Create API with VPC Link integration:
```
cat > apigw-api.yaml<<EOF
apiVersion: apigatewayv2.services.k8s.aws/v1alpha1
kind: API
metadata:
  name: apitest-private-nlb
spec:
  body: '{
              "openapi": "3.0.1",
              "info": {
                "title": "ack-apigwv2-import-test-private-nlb",
                "version": "v1"
              },
              "paths": {
              "/hello": {
                  "get": {
                    "x-amazon-apigateway-integration": {
                       "uri" : "$(aws elbv2 describe-listeners \
  --load-balancer-arn $(aws elbv2 describe-load-balancers \
  --region $AGW_AWS_REGION \
  --query "LoadBalancers[?contains(DNSName, '$(kubectl get service hello-service \
  -o jsonpath="{.status.loadBalancer.ingress[].hostname}")')].LoadBalancerArn" \
  --output text) \
  --region $AGW_AWS_REGION \
  --query "Listeners[0].ListenerArn" \
  --output text)",
                      "httpMethod": "GET",
                      "connectionId": "$(kubectl get vpclinks.apigatewayv2.services.k8s.aws \
  nlb-internal \
  -o jsonpath="{.status.vpcLinkID}")",
                      "type": "HTTP_PROXY",
                      "connectionType": "VPC_LINK",
                      "payloadFormatVersion": "1.0"
                    }
                  }
                }
              },
              "components": {}
        }'
EOF
kubectl apply -f apigw-api.yaml
```

- Create a stage:
```
echo "
apiVersion: apigatewayv2.services.k8s.aws/v1alpha1
kind: Stage
metadata:
  name: "apiv2"
spec:
  apiID: $(kubectl get apis.apigatewayv2.services.k8s.aws apitest-private-nlb -o=jsonpath='{.status.apiID}')
  stageName: api
  autoDeploy: true
" | kubectl apply -f -
```

- Get API endpoint:
```
kubectl get api apitest-private-nlb -o jsonpath="{.status.apiEndpoint}"
```

- Test the API:
```
curl $(kubectl get api apitest-private-nlb -o jsonpath="{.status.apiEndpoint}")/api/hello
```

# 3. Clean Up 
```
kubectl delete stages.apigatewayv2.services.k8s.aws apiv1 
kubectl delete apis.apigatewayv2.services.k8s.aws apitest-private-nlb
kubectl delete vpclinks.apigatewayv2.services.k8s.aws nlb-internal 
kubectl delete service hello-service
sleep 10
aws ec2 delete-security-group --group-id $AGW_VPCLINK_SG --region $AGW_AWS_REGION 
helm delete aws-load-balancer-controller --namespace kube-system
helm delete ack-apigatewayv2-controller --namespace kube-system
for role in $(aws iam list-roles --query "Roles[?contains(RoleName, \
  'eksctl-demo-cluster-addon-iamserviceaccount-')].RoleName" \
  --output text)
do (aws iam detach-role-policy --role-name $role --policy-arn $(aws iam list-attached-role-policies --role-name $role --query 'AttachedPolicies[0].PolicyArn' --output text))
done
for role in $(aws iam list-roles --query "Roles[?contains(RoleName, \
  'eksctl-demo-cluster-addon-iamserviceaccount')].RoleName" \
  --output text)
do (aws iam delete-role --role-name $role)
done
aws iam delete-policy --policy-arn $(echo $(aws iam list-policies --query 'Policies[?PolicyName==`ACKIAMPolicy`].Arn' --output text))
aws iam delete-policy --policy-arn $(echo $(aws iam list-policies --query 'Policies[?PolicyName==`AWSLoadBalancerControllerIAMPolicy-APIGWDEMO`].Arn' --output text))
eksctl delete cluster --name $AGW_EKS_CLUSTER_NAME --region $AGW_AWS_REGION
```