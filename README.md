## Deploy Containerized Application To AWS App Runner

This lab is provided as part of **[AWS Innovate Modern Applications Edition](https://aws.amazon.com/events/aws-innovate/modern-apps/)**, click [here]() to explore the full list of hands-on labs.

ℹ️ You will run this lab in your own AWS account. Please follow directions at the end of the lab to remove resources to avoid future costs.

### About this lab

[AWS App Runner](https://aws.amazon.com/apprunner/) is a fully managed service that makes it easy for developers to quickly deploy containerized web applications and APIs, at scale and with no prior infrastructure experience required. Start with your source code or a container image. App Runner automatically builds and deploys the web application and load balances traffic with encryption. App Runner also scales up or down automatically to meet your traffic needs. With App Runner, rather than thinking about servers or scaling, you have more time to focus on your applications.

[aws-app-runner](./images/how_it_works.png)

In this lab, we will build a container image which we will push to [Amazon Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/), a fully managed container registry. The container image we have built will then be deployed to AWS App Runner.

## Setup

### Step 1 - Create Cloud9 environment via AWS CloudFormation

1. Log in your AWS Account
1. Click [this link](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=AWS-App-Runner&templateURL=https://aws-innovate-modern-applications.s3.amazonaws.com/aws-app-runner/cloud9.yaml) and open a new browser tab
1. Click *Next* again to the stack review page, tick **I acknowledge that AWS CloudFormation might create IAM resources** box and click *Create stack*.
  
  ![Acknowledge Stack Capabilities](./images/acknowledge-stack-capabilities.png)

4. Wait for a few minutes for stack creation to complete.
5. Select the stack and note down the outputs (*Cloud9EnvironmentId* & *InstanceProfile*) on *outputs* tab for next step.

  ![Cloud9 Stack Output](./setup/images/stack-cloud9-output.png)

### Step 2 - Assign instance role to Cloud9 instance

1. Launch [AWS EC2 Console](https://console.aws.amazon.com/ec2/v2/home?#Instances).
2. Use stack output value of *Cloud9EnvironmentId* as filter to find the Cloud9 instance.

  ![Locate Cloud9 Instance](./setup/images/locate-cloud9-instance.png)

3. Right click the instance, *Security* -> *Modify IAM Role*.
4. Choose the profile name matches to the *InstanceProfile* value from the stack output, and click *Apply*.

  ![Set Instance Role](./setup/images/set-instance-role.png)

### Step 3 - Disable Cloud9 Managed Credentials

1. Launch [AWS Cloud9 Console](https://console.aws.amazon.com/cloud9/)
1. Locate the Cloud9 environment created for this lab and click "Open IDE". The environment title should start with *AppRunnerCloud9*.
1. At top menu of Cloud9 IDE, click *AWS Cloud9* and choose *Preferences*.
1. At left menu *AWS SETTINGS*, click *Credentials*.
1. Disable AWS managed temporary credentials:

  ![Disable Cloud 9 Managed Credentials](./setup/images/disable-cloud9-credentials.png)

### Step 4 - Prepare lab environment on Cloud9 IDE

Run commands below on Cloud9 Terminal to clone this lab repository:

```
git clone https://github.com/phonghuule/aws-app-runner.git
```
## Lab

### Create ECR Repository
Our first step will be to create a repository in ECR, this is where we later will store our container image once it’s been built.

Use the following command:
```
aws ecr create-repository --repository-name hello-app-runner
```
We are interested in using the repositoryUri later, so that we can tag and push images to the repository, let’s go ahead and store that as an environment variable for easy access later on.

```
export ECR_REPOSITORY_URI=$(aws ecr describe-repositories --repository-names apprunnerworkshop-app --query 'repositories[?repositoryName==`hello-app-runner`].repositoryUri' --output text)
echo $ECR_REPOSITORY_URI
1234567891012.dkr.ecr.eu-west-1.amazonaws.com/apprunnerworkshop-app
```

### Step 1

Create a Kubernetes service account named alb-ingress-controller in the kube-system namespace, a cluster role, and a cluster role binding for the ALB Ingress Controller to use with the following command.

```
sudo chmod +x /home/ec2-user/bin/kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/rbac-role.yaml
```
### Step 2

ALB Ingress Controller needs to know the EKS cluster name. Run the command below to download, update the deployment manifest and deploy the ALB Ingress Controller .

```
curl -sS "https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/alb-ingress-controller.yaml" \
     | sed "s/# - --cluster-name=devCluster/- --cluster-name=eks-alb-2048game/g" \
     | kubectl apply -f -
```

### Step 3

Confirm that the ALB Ingress Controller is running with the following command. 

```
kubectl get pods -n kube-system
```

Expected output:

```
NAME                                      READY   STATUS    RESTARTS   AGE
alb-ingress-controller-55b5bbcb5b-bc8q9   1/1     Running   0          56s
```

### Step 4

Create a namespace for 2048 game.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-namespace.yaml
```

Verify the namespace has been created:

```
kubectl get ns
```

Expected output:

```
NAME              STATUS   AGE
2048-game         Active   42h
default           Active   42h
kube-node-lease   Active   42h
kube-public       Active   42h
kube-system       Active   42h
```

### Step 5

Create a deployment to run 2048 game application pods.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-deployment.yaml
```

Verify the deployment has been created:

```
kubectl get deployment -n 2048-game
```

Expected output:

```
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
2048-deployment   5/5     5            5           42h
```

### Step 6

Create a service to abstract 2048 game application pods.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-service.yaml
```

Verify the service has been created:

```
kubectl get service -n 2048-game
```

Expected output:

```
NAME           TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service-2048   NodePort   10.100.49.101   <none>        80:32667/TCP   42h
```

### Step 7

Deploy ALB Ingress resource to expose 2048 Game via AWS Application Load Balancer.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-ingress.yaml
```

After a few minutes, verify that the Ingress resource was created with the following command. 

```
kubectl get ingress/2048-ingress -n 2048-game
```

Expected output:

```
NAME           HOSTS   ADDRESS                                                                     PORTS   AGE
2048-ingress   *       7dfe79d3-2048game-2048ingr-6fa0-35333457.ap-southeast-2.elb.amazonaws.com   80      41h
```

To debug, run the following command to view the Ingress controller log:

```
kubectl logs -n kube-system   deployment.apps/alb-ingress-controller
```

### Step 8

Navigate to [AWS Console Load Balancer page](https://console.aws.amazon.com/ec2/v2/home#LoadBalancers:sort=loadBalancerName) to see the Load Balancer created by ALB Ingress Controller according to the Ingress resource. Wait for Load Balancer to be **active** before heading to next step.

![Wait for ALB to Active](./setup/images/wait-for-alb-active.png)

### Step 9

Open a browser and navigate to the ALB endpoint (shown in ADDRESS field from the previous command `kubectl get ingress/2048-ingress -n 2048-game` output or from AWS Load Balancer Console) to see the 2048 game application.

![Game 2048](./setup/images/game-2048.png)

## Clean up

### Step 1

Run *cleanup.sh* from Cloud 9 Terminal to delete EKS cluster and its resources. Cleanup script will:

- Delete all the resources installed in previous steps.
- Delete the EKS cluster created via bootstrap script.

```
./cleanup.sh
```

### Step 2

Double check the EKS Cluster stack created by eksctl was deleted:

- Launch [AWS CloudFormation Console](https://console.aws.amazon.com/cloudformation/home)
- Check if the stack **eksctl-eks-alb-2048game-cluster** still exists.
- If exists, click this stack, in the stack details pane, choose *Delete*.
- Select *Delete* stack when prompted.

### Step 3

Delete the Cloud 9 CloudFormation stack named **EKS-ALB-2048-Game** from AWS Console:

- Launch [AWS CloudFormation Console](https://console.aws.amazon.com/cloudformation/home)
- Select stack **EKS-ALB-2048-Game**.
- In the stack details pane, choose *Delete*.
- Select *Delete* stack when prompted.

## Reference

- [Kubernetes Ingress with AWS ALB Ingress Controller](https://aws.amazon.com/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/)
- [Github repository for AWS ALB Ingress Controller](https://github.com/kubernetes-sigs/aws-alb-ingress-controller)
