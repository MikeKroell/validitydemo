# Validity Interview Demo   

This project is to show I have some skills.

### The Exercise
Write a set of Terraform or configuration files or Cloudformation templates to set up a load balanced web service based on a docker container running on Fargate. This should include all dependencies such as but not limited to security groups, IAM roles, Load Balancer and ECS service as appropriate. Pick or build a docker image that will bring up a basic web service or UI.

### Rules:
Use any resource you have available to you - but remember that we’re going to ask you questions about your code, so be prepared to defend any design/technical choices you make.
Please provide a github repository and basic set up instructions.

## Getting Started

In the repository you will find all the CloudFormation files that will build:

* VPC with three AZs, with both public and private subnets.
* An ECR Repo for the demo service.
* ECS Fargate Cluster with ALB, listerners, target groups, and security groups.
* ECS Task & Service for the demo service with assoicated IAM Roles.

### Prerequisites

* git client
* Docker
* AWS CLI
* AWS SSL Certificate & ARN in the account and region you are deploying to.
* DNS recorded assoicated with the certificate

### Design & Considerations

The VPC is created with 3 AZs to ensure HA during deployments and failures. It has 3 public and 3 private subnets, as well as associated NAT gateways and IGWs. The ALB is stationed in the public AZs with the only public facing ingress. The services will be protected behind the NAT Gateway and setup one per private AZ. 

Using Exports – as one CloudFormation is created, you create exports for higher level templates to use. This allows resources that are dynamically created to be available to be consumed by the higher level templates.

## Setting up the Environment & Deploying the Service

This guide will not cover local installs or setup of any tools. You should be fimiliar with the tools and requirements within the prereqs.

To get started, start with pulling down the docker container. In this example we will be using a NGINX demo container.

```
docker pull nginxdemos/hello
```

Now that we have a copy of the image, we will now moving on to building the basic infrastructure. Start with the "infrastructure-vpc.yaml" and "infrastructure-vpc-parameters-dev.json" files. Ensure the json file parameters are updated to match your environment. The only one that is required is to choose a value for "EnvironmentName". Once complete there will exports available for cross-stacking.

```
aws cloudformation create-stack --stack-name infrastructure-vpc --template-body file://infrastructure-vpc.yaml --capabilities CAPABILITY_NAMED_IAM --parameters file://infrastructure-vpc-parameters-dev.json
```

At this point you should have a VPC and it's exports. Now we will launch the ECS cluster and basic components. Start with the "ecs-primary.yaml" and "demo-ecs-primary-parameters-dev.json". Edit as needed. This is where you will need the Certificate ARN. Once compelete there will exports available for cross-stacking.

```
aws cloudformation create-stack --stack-name demo-ecs-primary --template-body file://ecs-primary.yaml --capabilities CAPABILITY_NAMED_IAM --parameters file://demo-ecs-primary-parameters-dev.json
```

Now you should have a cluster, load balancer with listeners, target group, and security group. The security group should be limiting access to the load balancer. Now we will create the repo. Start with the "ecr.yaml" and "demo-ecr.json". Edit as needed.

*At this step you should be gathering the A record for the load balancer and updating the DNS record. Route53 would be perfered here, but this was not integrated for this demo.

```
aws cloudformation create-stack --stack-name demo-ecr --template-body file://ecr.yaml --parameters file://demo-ecr-parameters.json
```

With the repo created, we can now upload the Docker image. The following are sample commands. Your command will be slightly different.

```
docker tag nginxdemos/hello 687697781762.dkr.ecr.us-east-1.amazonaws.com/demo:latest
docker push 687697781762.dkr.ecr.us-east-1.amazonaws.com/demo:latest
```

Now we should have an image in the repo tagged "latest". Now we will deploy the container as a service using Fargate. Start with the "demo-service.yaml" and "demo-service-parameters-dev.json" files. Edit as needed. You will need the reposity URL for the image you just uploaded. Similiar to "687697781762.dkr.ecr.us-east-1.amazonaws.com/demo:latest"

```
aws cloudformation create-stack --stack-name demo-service --template-body file://demo-service.yaml --capabilities CAPABILITY_NAMED_IAM --parameters file://demo-service-parameters-dev.json
```
This should result in 3 running services attached to the target group created in the previous steps. The security group for the services should only be allowing connections from the load balancer. Now you should be able to browse to the DNS recorded created for this demo. You will be redirected from HTTP to HTTPS, and there should be no SSL errors. Refreshing the page will result in displaying the IP of the container, one for each AZ.

My example is available at: http://validity.mkroell.com

## Ending Statements

Thank you for your consideration and time in reviewing this application.

