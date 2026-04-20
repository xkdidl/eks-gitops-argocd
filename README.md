# eks-gitops-argocd

This repository walks through implementing GitOps with GitHub Actions and ArgoCD to deploy applications via helm to EKS

## High Level Architecture
![Alt text](images/eks-gitops-argocd.png?raw=true "GitOps on EKS with GitHub Actions and ArgoCD")

## Build EKS Cluster
For this demo, we will create the EKS cluster using EKSCTL. Before that, some housekeeping
```
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[0].[RegionName]' --output text)
export AZ1=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[0].ZoneName' --region $AWS_REGION --output text)
export AZ2=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[1].ZoneName' --region $AWS_REGION --output text)
export AZ3=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[2].ZoneName' --region $AWS_REGION --output text)
export K8S_VERSION=1.25
```

If you do not have eksctl installed, install it.
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

Create a CMK for the EKS cluster to use when encrypting your Kubernetes secrets:
```
aws kms create-alias --alias-name alias/eksworkshop --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
```

Let’s retrieve the ARN of the CMK to input into the create cluster command.
```
export KEY_ARN=$(aws kms describe-key --key-id alias/eksworkshop --query KeyMetadata.Arn --output text)
```

We set the KEY_ARN environment variable to make it easier to refer to the KMS key later.

Now, let’s save the KEY_ARN environment variable into the bash_profile
```
echo "export KEY_ARN=${KEY_ARN}" | tee -a ~/.bash_profile
```


Now let's create the cluster
```
eksctl create cluster -f config/cluster.yaml
```

Test the cluster
```
kubectl get nodes
# if we see our 3 nodes, we know we have authenticated correctly
```

Update kubeconfig file to interact with the EKS cluster
```
aws eks update-kubeconfig --name eks-gitops --region ${AWS_REGION}
```

## Create an ECR Repository
... In the same region, and call it  `eks-gitops-argocd`. This is because the files/ configurations in this demo use that name.

## Setup GitHub Actions

Use IAM roles for GitHub Actions to connect to Amazon ECR. Follow [this blog](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/) to set it up. 

Use the contents below for `.github/main.yml`
```
# This is a basic workflow to help you get started with Actions

name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
env:
  AWS_REGION : "<AWS_REGION>" 
# Permission can be added at job level or workflow level    
permissions:
      id-token: write   # This is required for requesting the JWT
      contents: write    # This is required for actions/checkout
jobs:
  build:
    name: Building and Pushing the Image
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v2

    - name: configure aws credentials
      uses: aws-actions/configure-aws-credentials@v1.7.0
      with:
        role-to-assume: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<IAM_ROLE_NAME>
        role-session-name: GitHub_to_AWS_via_FederatedOIDC
        aws-region: ${{ env.AWS_REGION }}
    # Hello from AWS: WhoAmI
    - name: Sts GetCallerIdentity
      run: |
        aws sts get-caller-identity

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build, tag, and push image to Amazon ECR
      id: build-image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: eks-gitops-argocd

      run: |
        # Build a docker container and push it to ECR
        git_hash=$(git rev-parse --short "$GITHUB_SHA")
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_REF##*/}-$git_hash docker/.
        echo "Pushing image to ECR..."
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_REF##*/}-$git_hash
        echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_REF##*/}-$git_hash"
        
    - name: Update Version
      run: |
          git_hash=$(git rev-parse --short "$GITHUB_SHA")
          version=$(cat ./charts/helm-example/values.yaml | grep version: | awk '{print $2}')
          sed -i "s/$version/${GITHUB_REF##*/}-$git_hash/" ./charts/helm-example/values.yaml
          
    - name: Commit and push changes
      uses: devops-infra/action-commit-push@v0.3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        commit_message: Version updated

```

## Install ArgoCD on EKS

Install helm
```
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version --short

```

Install ArgoCD
```
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --set server.service.type=LoadBalancer
```

Access the UI which is sitting behing an ELB. To get the password:
```
kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Connect to the Repo -> Add/Create an application
```
sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64

sudo chmod +x /usr/local/bin/argocd

export ARGOCD_SERVER=`kubectl get svc argocd-server -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname'`

export ARGO_PWD=`kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

argocd login $ARGOCD_SERVER --username admin --password $ARGO_PWD --insecure
```

Add ECR helm repo
```
argocd repo add XXXXXXXXXX.dkr.ecr.us-west-2.amazonaws.com --type helm --name eks-gitops-argocd --enable-oci --username AWS --password $(aws ecr get-login-password --region us-west-2)
```