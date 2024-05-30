# DevOps-Engineer-Assignment
This is for the submission of a test assignment for an internship


 ## Creating the cluster
 1. SSH into the bootstrap-server
     e.g. `ssh -i kops-bootstrap.pem ec2-user@54.215.251.65`

 2.  Set the variables NAME(name of the cluster which we set in Amazon Route 53 Hosted Zones) & KOPS_STATE_STORE(kops saves the state of the 
     cluster in a S3 bucket)
    `export NAME=kube.xyz`
    `export KOPS_STATE_STORE=s3://<bucket name>`

3. Create configuration for cluster
   To check the available zones in the region 
   `aws ec2 describe-availability-zones --region <region name i.e us-west-1>`
   `kops create cluster --zones us-west-1a,us-west-1c --node-count=3 --node-size=t3.xlarge --networking=calico --topology=public ${NAME}`
   (other relevant options can be found here https://kops.sigs.k8s.io/cli/kops_create_cluster/)
   If we get warning related to SSH then we can create 
   `kops create secret --name ${NAME}  --state  sshpublickey admin -i ~/.ssh/id_rsa.pub`
   This will create configuration for creating cluster.

4. We can check the configuration of the cluster and add `Cluster Autoscaler` configuration  ---
   we can use `export EDITOR=nano` , to open the configuration file with nano , default is vi
   `kops edit cluster --name ${NAME}`  
   this will open the configuration file for cluster , we can edit this file for cluster configuration changes
   for autoscaler configuration we need to add the followinf under spec:
clusterAutoscaler:
  awsUseStaticInstanceList: false
  balanceSimilarNodeGroups: false
  cpuRequest: 100m
  enabled: true
  expander: least-waste
  image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.29.0
  memoryRequest: 300Mi
  newPodScaleUpDelay: 0s
  scaleDownDelayAfterAdd: 10m0s
  scaleDownUnneededTime: 10m0s
  scaleDownUnreadyTime: 20m0s
  scaleDownUtilizationThreshold: "0.5"
  skipNodesWithLocalStorage: true
  skipNodesWithSystemPods: true

5. Instance groups are the group of worker nodes , and control-plane nodes
   we can see the list of instance groups by using 
   `kops get ig --name ${NAME}`
   we can edit the instance group  (like number of nodes we need in cluster , max & min number of nodes , node size, subnet, etc ) 
   `kops edit ig <ig_name> --name ${NAME}`

6. Building the cluster using the above configuration.
   `kops update cluster ${NAME} --yes --admin=87600h`
   `kops export kubecfg --admin=87600h`
   `kops validate cluster ${NAME}`    
   (it will take around 10-15 mins for cluster to be ready)



## After the cluster is ready.
    The ingress-nginx controller will be created, which creates a network load balancer of ip address type - ipv4 , we have to change the 
    ip address type to dualstack.
    To know the Load balancer attached to the ingress controller--
    `kubectl get svc -n ingress-nginx`   
    This will list all the services in namesapce ingress-nginx. 
    The domain name of Load balancer will be attached to the EXTERNAL_IP of the service ingress-nginx-controller.
    On EC2 dashboard select Load balancers -> select the Load balancer -> go to Actions on top right corner -> select Edit IP addres type - 
    > select Dualstack -> Sav
    You can check the IPv4 and IPv6 network attached to Load balancer by using 
    `nslookup <domain of load balancer>`
## Docker and ECR setup.
    1. Firstly I installed Docker into my local system.
    2. Then I created a simple Java Application.
    3. Generating the .jar file of the application.
       `mvn clean install`
    4. Build the image using .jar file
       `docker build -t your-application .`


  Now let's setup ECR and then we will push this image to it.
  
    1. Create an ECR repository.
       Open the Amazon ECR console at https://console.aws.amazon.com/ecr/repositories.
       Choose Create repository.
       For Repository name, enter a unique name for your repository.
       Choose Create repository.

    2. Authenticate Docker to Your ECR Repository.
       `aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.us-west- 
        2.amazonaws.com`

    3. Tag Your Docker Image
       `docker tag your-application:latest <aws_account_id>.dkr.ecr.us-west-2.amazonaws.com/my-repository:latest`

    4. Push docker image.
       `docker push <aws_account_id>.dkr.ecr.us-west-2.amazonaws.com/my-repository:latest`



## Now I will pull the image on already running kubernetes cluster and run it along with the secret.yml, pvclaim.yml and ingress.yml.

    To apply or start service-
    kubectl apply -f service.yml

    Apply secrets-
    kubectl apply -f secrets.yml

    Apply persistent volume claim for the service-
    kubectl apply -f pvclaim.yml

    Now we apply ingress files-
    kubectl apply -f ingress.yaml

    To verify ingress service-
    kubectl get ingress myservice-ingress


After these steps the service is deployed.
