# Auto Deploymemnt of EKS Infrastructure on AWS with Terraform

## Project Content
This project contains the three modules
* **cluster-autoscaler**: Contains yaml file of cluster-autoscaler to scale out ans scale down the EKS cluster nodes. 
* **metrics-server**: Contains metrics server yaml files to deploy the metrics server. This server is needed to collect the metrics from pods.
* **tf-eks-demo**: Contains terraform scripts to deploy infrastructure in declarative format. 
* **HPA(Horizontal Pod Autoscaler)**: - To scale out and scale down the pods on nodes.

We create the following infrastructure on AWS with the instructions given below.
- EC2 Instances
- EKS Cluster
- Horizontal Pod AutoScaler
- Cluster AutoScaler
- Deploying Metrics-Server
- Deploying php-apache service 

There are 2 ways we can deploy EKS cluster on AWS

#### Eksctl tool: 
* It's a new CLI tool from AWS to create EKS clusters. It uses CloudFormation in background to create the clusters.

#### Terraform: 
* Its another popular IaC (Infrastructure as Code) tool to create infrastructure in declarative way. Using this tool 
  we can create servers, clusters or any other infra on on-premises, AWS, Azure, GCP, IBM Cloud and many more.
  It is much useful when your organization has hybrid/multi-cloud environment.

## Creating EKS cluster Using `eksctl`
- Prerequisite: ekstl cli tool needs to be installed on laptop. 

To create cluster
````
eksctl create cluster --region=us-east-1 --node-type=t2.medium
````
The above command will create EKS cluster with default parameters if you dont specify any.

We can create EKS cluster using eksctl with yaml file as well with following command.
````
eksctl create cluster -f eks_cluster.yaml
````
The content of eks_cluster.yaml can be 
````
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: basic-cluster
  region: eu-north-1

nodeGroups:
  - name: ng-1
    instanceType: m5.large
    desiredCapacity: 10
  - name: ng-2
    instanceType: m5.xlarge
    desiredCapacity: 2
````

## Creating EKS cluster with Terraform Scripts
 - Prerequisite: kubectl & aws-iam-authenticator cli tools need to be installed on laptop.
 
There are terraform scripts in "tf-eks-demo" folder. By running the following commands terraform creates
EC2 instance and EKS cluster for us in the desired region. All the required configs are defined in respective script, 
like IAM roles, policies, security groups, etc.
Before executing the below scripts the user must have created an IAM role and needs to be configured
on his laptop. Else the access_key & secret_key needs to be configured in terraform script.

````
terraform init   //to initialize terraform 
terraform plan   //to review the tf scripts and to make the plan by terraform
terraform apply  //final command to execute the provision of infra on cloud or on-premise
```` 
The deployment of infra will take at least 15 min.

To execute further commands the following CLI tools needs to be installed on laptop
* kubectl - To connect and co-ordinate with Kubernetes cluster.
* aws-iam-authenticator - To connect with AWS cloud with IAM roles in a secured way.

Once the cluster is up and running, we need to run the following commands to get configurations of EKS and 
to connect with AWS cloud

The below commands needs to be executed to copy the EKS configuration to kubectl cli tool to get conenct with Kubernetes
````
terraform output kubeconfig > ~/.kube/config 
aws eks --region us-east-1 update-kubeconfig --name terraform-eks-demo
````
The below command to get config details of authentication with AWS & deploy the config map deployment.
````
terraform output config-map-aws-auth > config-map-aws-auth.yaml  
kubectl apply -f config-map-aws-auth.yaml  
````

## Deploying Cluster AutoScaler
Before deploying Cluster Auto Scaler, we need to create AutoScaling Policy & attached to worker node group which is needed to 
access the worker nodes and auto scale whenever needed.

Add autoscaling Policy to the worker node. It can be done via Terraform script as well.

Goto
AWS Console ->
IAM -> Roles -> Find the worker node group and 
-> Add inline policy 
-> Go to JSON Tab and copy the below policy 
````
{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "autoscaling:DescribeAutoScalingGroups",
          "autoscaling:DescribeAutoScalingInstances",
          "autoscaling:DescribeTags",
          "autoscaling:SetDesiredCapacity",
          "autoscaling:TerminateInstanceInAutoScalingGroup"
        ],
        "Resource": "*"
      }
    ]
  }

 ````
Deploy Cluster Auto Scaler
````
 kubectl apply -f cluster-autoscaler/cluster_autoscaler.yaml
````
 ### Deploying Metrics-Server for Kubernetes 
Metrics Server needed to collect the metrics of cpu and memory utilization for HPA   
Run the following command from root folder
````
cd /metrics-server
kubectl apply -f .
````
To confirm metrics-server is running
````
kubectl get pods -n kube-system
````

### Deploying a php-Apache and service & Horizontal Pod AutoScaler(HPA)
To create a php-apache deployment
````
kubectl create deployment php-apache --image=k8s.gcr.io/hpa-example
````

To set CPU requests 
````
kubectl patch deployment php-apache -p='{"spec":{"template":{"spec":{"containers":[{"name":"hpa-example","resources":{"requests":{"cpu":"200m"}}}]}}}}'
````
To expose the deployment as a service
````
kubectl create service clusterip php-apache --tcp=80
````
To create an HPA
````
kubectl autoscale deployment php-apache --cpu-percent=20 --min=1 --max=20
````

To confirm that the HPA was created
````
kubectl get hpa
````

To create a pod to connect to the deployment that was created earlier
````
kubectl run --generator=run-pod/v1 -i --tty load-generator --image=busybox /bin/sh
````
To test a load on the pod in the namespace, run the following script
````
while true; do wget -q -O- http://php-apache; done
````
To see how the HPA scales the pod based on CPU utilization metrics
````
kubectl get hpa -w
````
To clean up the resources used for testing the HP
````
kubectl delete hpa,service,deployment php-apache
kubectl delete pod load-generator
````

Buy ingesting more load with the script, the php-apache will get scaled up( we gave --max=20) by Horizontal Pod Autoscaler.
So whenever it created more pods on worker nodes, it will increase the load on cluster, the Cluster Autoscaler jumps up
and will create new worker nodes to handle the load.

Once we stop the script of load ingestion, HPA scale down the pods and automatically replica set of that pods will come down. Then it will
decrease the load omn cluster and Cluster Autoscaler removes the worker nodes based on the load.
There are configurations set on cluster-autoscaler.yaml file to when to scale up and scale down based cpu, requests

````
 containers:
        - image: k8s.gcr.io/cluster-autoscaler:v1.2.2
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 300Mi
            requests:
              cpu: 100m
              memory: 300Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --nodes=2:5:terraform-eks-demo
            - --scale-down-delay-after-add=2m
            - --skip-nodes-with-system-pods=false
````
 
To undeploy infra on, first delete the manually cretaed security group policy and run the following command. This will ensure all the infra is deleted
````
cd tf-eks-demo
terraform destroy
````
