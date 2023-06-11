---
title: "GitLab CI/CD Pipeline: Kubernetes (EKS), and Terraform for deployment"
date: 2023-06-04T00:16:02-04:00
type: "post"
tags: ["linux", "cicd", "devops"]
draft: false
---

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1114769740976635934/xo8c2f0mcoyyx7gcr3ox.webp)

In today's world of fast-paced software development, efficient deployment of applications has become more critical than ever. Containerization has revolutionized how we build, ship, and run applications, making it easier to deploy them consistently across different environments.

In this blog post, we will explore containerizing a simple Node.js weather application GITHUB that displays weather data from any given location, and then with this done proceed to automate its deployment using a GitLab CI/CD pipeline. I will demonstrate the entire process of containerizing the application, pushing it to a container registry, which will be AWS ECR, and deploying it to EKS ready to be used.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1115081880891564042/image_2.png)

Not only that but I will also automate our infrastructure resources provisioning with Terraform as well. A full CI/CD pipeline, any time a new change in our code has been made and pushed into the Gitlab repository, will trigger our resources to get provisioned and deploy our application.

First, we need to dockerize the application by creating its Dockerfile.

{{< highlight Dockerfile >}}
FROM node:latest

WORKDIR /usr/src/app

COPY source/package*.json ./
RUN npm install
COPY source/. .
EXPOSE 5000
CMD ["npm", "start"]
{{< /highlight >}}

This Dockerfile is based on the latest Node.js image since our application requires Node.js. Then, the Dockerfile copies all the files and directories from the source directory to the current working directory in the Docker image using the COPY instruction.

Finally, the CMD instruction specifies the command that should be executed when a container based on this image is run. In this case, it starts the Node.js application using the npm start command.

Now that we have successfully containerized the application we can proceed with writing our infrastructure that will be used to deploy the application. For this particular case, I will be using Terraform to write our EKS cluster with its required configurations.


{{< highlight terraform >}}
resource "aws_default_subnet" "default_az1" {
  availability_zone = "us-east-1a"

  tags = {
    Name = "Default subnet for us-east-1a"
  }
}


resource "aws_default_subnet" "default_az2" {
  availability_zone = "us-east-1b"

  tags = {
    Name = "Default subnet for us-east-1b"
  }
}


resource "aws_eks_cluster" "eks_cluster_tf" {
  name     = "eks_cluster"
  role_arn = aws_iam_role.eks-deployment.arn

  vpc_config {
    subnet_ids = [aws_default_subnet.default_az1.id, aws_default_subnet.default_az2.id]
  }

  # Ensure that IAM Role permissions are created before and deleted after EKS Cluster handling.
  # Otherwise, EKS will not be able to properly delete EKS managed EC2 infrastructure such as Security Groups.
  depends_on = [
    aws_iam_role_policy_attachment.AmazonEKSClusterPolicy,
    aws_iam_role_policy_attachment.AmazonEKSVPCResourceController,
  ]
}


resource "aws_eks_node_group" "example" {
  cluster_name    = aws_eks_cluster.eks_cluster_tf.name
  node_group_name = "main-node-group"
  node_role_arn   = aws_iam_role.ec2-worknode.arn
  subnet_ids      = [aws_default_subnet.default_az1.id, aws_default_subnet.default_az2.id]

  scaling_config {
    desired_size = 1
    max_size     = 2
    min_size     = 1
  }

  update_config {
    max_unavailable = 1
  }

  # Ensure that IAM Role permissions are created before and deleted after EKS Node Group handling.
  # Otherwise, EKS will not be able to properly delete EC2 Instances and Elastic Network Interfaces.
  depends_on = [
    aws_iam_role_policy_attachment.example-AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.example-AmazonEKS_CNI_Policy,
    aws_iam_role_policy_attachment.example-AmazonEC2ContainerRegistryReadOnly,
  ]
}

output "endpoint" {
  value = aws_eks_cluster.eks_cluster_tf.endpoint
}

output "kubeconfig-certificate-authority-data" {
  value = aws_eks_cluster.eks_cluster_tf.certificate_authority[0].data
}

{{< /highlight >}}

This is a Terraform configuration file that specifies the creation of an Amazon Elastic Kubernetes Service (EKS) cluster with a node group. The file includes the following resources:

aws_eks_cluster resource: Defines the EKS cluster and specifies the name, IAM role for the EKS service, and the VPC configuration that includes the subnet IDs of the two default subnets.

aws_eks_node_group resource: Defines the EKS node group and specifies the cluster name, node group name, IAM role for the EC2 instances, and the subnet IDs of the two default subnets. This resource also specifies the scaling and update configurations for the node group and has a depends_on block that ensures that the IAM Role policies are created before and deleted after the EKS node group and its associated infrastructure.

Now that we have all our pieces is time to put them together with some CI/CD. I will be using Gitlab for the purpose of this post.

Our pipeline will consist of two stages:

1. Build the application (Docker)
2. Deploy our infrastructure and application (Terraform, EKS)


{{< highlight yaml >}}
image:
  name: registry.gitlab.com/gitlab-org/gitlab-build-images:terraform
  entrypoint:
    - '/usr/bin/env'
    - 'PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'

stages:          # List of stages for jobs, and their order of execution
  - build
  - deploy

variables:
  DOCKER_REGISTRY: 320964430417.dkr.ecr.us-east-1.amazonaws.com
  AWS_DEFAULT_REGION: us-east-1
  APP_NAME: weatherapp_nodejs
  TF_ROOT: infrastructure  # The relative path to the root directory of the Terraform project

cache:
  paths:
    - .terraform

buildapp-job:
  image: docker:latest
  stage: build
  services:
    - docker:dind
  before_script:
    - apk add --no-cache curl jq python3 py3-pip
    - pip install awscli
    - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $DOCKER_REGISTRY
    - aws --version
    - docker info
    - docker --version
  script:
    - echo "Building the image..."
    - docker build --pull -t $DOCKER_REGISTRY/$APP_NAME:$CI_PIPELINE_IID -t $DOCKER_REGISTRY/$APP_NAME:latest .
    - docker push -a $DOCKER_REGISTRY/$APP_NAME
    - echo "Compile complete."
  rules:
    - if: $CI_COMMIT_BRANCH
      exists:
        - Dockerfile

deploy-infrastructure:
  stage: deploy
  before_script:
  - cd "${TF_ROOT}"
  - terraform --version
  - terraform init -backend-config="tfstate.config"
  script:
    - terraform validate
    - terraform plan -out="planfile"
    - terraform apply "planfile"
  artifacts:
    paths:
      - planfile

deploy-eks:
  image:
    name: matshareyourscript/aws-helm-kubectl
  stage: deploy
  before_script:
    - export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
    - export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
    - export AWS_DEFAULT_REGION=us-east-1
  script:
    - cd eks
    - aws eks update-kubeconfig --name eks_cluster --region us-east-1
    - kubectl apply -f aws-auth.yaml
    - kubectl apply -f deployment.yaml
    - kubectl apply -f service.yaml
  dependencies:
    - deploy-infrastructure

{{< /highlight >}}

The pipeline defines two stages: "build" and "deploy". In the "build" stage, a Docker image is built from a Dockerfile and pushed to an ECR registry. In the "deploy" stage, the Terraform script is used to deploy the infrastructure, followed by deploying the Kubernetes resources to the EKS cluster.

The pipeline configuration file also includes several environment variables, such as the Docker registry URL, the AWS region, and the Terraform root directory. It uses the Docker-in-Docker (dind) service to build and push the Docker image. Additionally, the "awscli" and "terraform" tools are installed and used in the pipeline.

The "deploy-eks" job uses an external Docker image that has the necessary tools installed to interact with the EKS cluster. It also depends on the successful completion of the "deploy-infrastructure" job, indicating that the infrastructure is in place before deploying the Kubernetes resources.

Now that we have tied up all our pieces with some CI/CD let's test our pipeline by pushing all our changes into our repository.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1115091927247880202/image-1.png)

We are good! Our application has been fully deployed into our EKS cluster and is now available to use by our customers.