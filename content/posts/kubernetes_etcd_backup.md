---
title: "Kubernetes ETCD Backup and Restore"
date: 2023-07-05T23:11:13-04:00
type: "post"
image: "image/docker_linux.png"
tags: ["devops", "kubernetes", "docker"]
draft: false
---
![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1105697155408220262/1620155432-blog-library-product-terraform-aws-logomarks-dark.jpg)

Why waste your time clicking and setting up infrastructure through a web console if you can write code to provision those resources for you?

Today I will share with you my latest project of deploying an EC2 instance using Terraform, but not only we will be provisioning this instance, also running a bash script as soon as it's deployed to install Nginx to have a small site for us to show our friends like this.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1105697364745924669/image-5.png)

***What is Terraform?***
--------------
Terraform is an infrastructure as code tool which lets you manage cloud and on-prem resources. It's primarily used by DevOps teams to automate various infrastructure tasks like the one that I will be showing you today. Instead of going manually on AWS and clicking to deploy this project I will be using Terraform to automate that process and deploying our resources needed in a matter of seconds.

You can read more about Terraform and Infrastructure as code here

I would go into more detail about these topics but this will be outside of the main scope of this post, with that being said let's get started.

***Setting up the project environment***
--------------
Let's start by creating our main files here. Our project will consist of the following files, I will go into further detail on each.

- main.tf - > We will declare our AWS resources here (EC2, Security Group, VPC)
- outputs.tf -> Will hold our output variables which will be shown after deployment (EC2 DNS)
- providers.tf - > This file will hold our AWS provider
- variables.tf -> We will define all our variables here for example the AWS region we would like to deploy our resources to.

***AWS Resources***
--------------
On our main.tf let's start by declaring the resources needed for our EC2 instance. In order for our friends to have access to see our cool site we must allow public access to it. We could create our own VPC for the purpose of our site but the default VPC that AWS creates for us should be enough here alongside a new AWS security group, which will be provisioned by Terraform as well to allow port 80 (HTTP).

{{< highlight terraform >}}
# < Declare the region we are going to deploy our AWS resources >
provider "aws"{
    region = var.aws_region
}

# Default VPC for our EC2
resource "aws_default_vpc" "default" {

}

# <Our security group resource to configure public access to our EC2 >
resource "aws_security_group" "world_wide_access" {
  name = "world_wide_access"
  description = "default VPC security group to provide public access to our EC2 instance"
  vpc_id = aws_default_vpc.default.id

  # TCP access
  ingress {
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP access from anywhere
  ingress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
   from_port   = 0
   to_port     = 0
   protocol    = "-1"
   cidr_blocks = ["0.0.0.0/0"]
 }
}
{{< /highlight >}}
As you can see we create a aws_default_vpc resource which will retrieve the required information to be used for our new security group called world_wide_access. We then go on to pass the arguments to allow TCP 22 (SSH) and port 80 (HTTP)

Next, we can proceed to declare our main EC2.

{{< highlight terraform >}}
# <Our main EC2 instance which will host our NGINX configuartion >
resource "aws_instance" "ec2_instance" {
    ami = "ami-06878d265978313ca" # Ubuntu Server 22.04 Free Tier
    instance_type = "t2.micro" # t2.micro instance type on this case gives us enough hardware to play around here
    key_name = "${var.ssh_key_name}" # Decided to generate our SSH key pair manually here instead of creating it through Terraform. In a real world scenario we should be careful with keys and keep them private.
    
    #Script to install nginx
    user_data = <<-EOF
  #!/bin/bash
  echo "*** Installing nginx"
  sudo apt update -y
  sudo apt-get install nginx -y
  sudo systemctl start nginx
  sudo systemctl enable nginx
  echo '<center><h1> Hi! I was fully deployed using code instead of boring clicks :> </h1></center>' > /var/www/html/index.html
  echo "*** Completed Installing nginx"
  EOF

    vpc_security_group_ids = [
     aws_security_group.world_wide_access.id
   ]
}
{{< /highlight >}}

We pass on our AMI which is a Ubuntu Server 22.04, instance type which t2.micro should give us enough hardware for this case, and our SSH key name which I decided to generate manually here instead of creating it through Terraform code. The reason behind that is that in a real-world scenario, we should be careful with keys and keep them private.

The main takeaway from the EC2 resource is that we use the argument user_data to run our Nginx install script after the instance has been provisioned. We could have used one of Terraform's provisoners here for example the remote_exec provisoner which invokes a script on a remote resource after it is created. You can read more about it here.

Terraform's documentation states that provisoners should be used as last resort and the user_data argument already gives us the ability to perform such script without the use of actual provisoners so we will go that route for this case.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1105698150443925564/image-7_1.png)

***Variables and Outputs***
--------------
Our variables.tf file will consist of the variables that we used in our resources which were the AWS region we will be deploying our instance to and our SSH key name.

{{< highlight terraform >}}
variable "aws_region" {
    type = string
    description = "AWS Region to deploy our resources"
    default = "us-east-1"
}

variable "ssh_key_name" {
    type = string
    description = "Our key pair key name created from our AWS console for security purposes"
    default = "master-key"
}
{{< /highlight >}}

As for our outputs.tf we only are going to have one output for this case and that will be the public_dns of our EC2 instance. We can click it right off the console once the resources have been provisioned.

***Terraform Providers***
--------------
As for our last but not least provider.tf file we will declare the providers that we will be using in our environment which of course will be the AWS provider.

{{< highlight terraform >}}
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "4.0.0"
    }
  }
}
{{< /highlight >}}

That will take care of our main resources and we are now ready to provision our EC2 instance.

***Launching Our Resources 🚀***
--------------
We can now launch our resources with only 3 commands. The first command will be to initialize our backend to install our providers.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1105699041330864269/image-8_1.png)

Next on we have our plan command which should be a dry run to see how our resources are going to be provisioned in the cloud.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1105699175959642132/image-9.png)

For last we have our apply command which will provision our resources in the cloud and launch the EC2 instance with our Nginx script.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1105699210067718265/image-10_1.png)

We can now see our output which we declared in our outputs.tf file and that is:

aws_instance_public_dns = "ec2-52-90-199-149.compute-1.amazonaws.com"

If we open up our browser here and copy-paste our instance public DNS we should see our new site up and running.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1105699442960646215/image-11_1.png)

***Conclusion***
--------------
Terraform is truly unique and I'm loving it, I come from a developer background so just the fact that we can use actual code to provision resources is truly amazing. I used this EC2 instance just to deploy a simple Nginx web server but this can be evolved into so many different things, for example, we could use Terraform to test a new production site, we could have a git script that pulls your site's repository from GitHub and automatically deploys it into your newly created EC2 instance to test and troubleshoot before launching it into the real world.

***Source Code***
--------------
https://github.com/frankelitoc/Terraform-EC2