---
title: "Installing Docker on my Raspberry Pi (Ubuntu)"
date: 2023-01-06T23:11:13-04:00
type: "post"
image: "image/docker_linux.png"
tags: ["linux", "docker"]
draft: false
---

![Alt Text](/image/docker_linux.png)

I recently got my hands on a Raspberry Pi 4 with 4GBs of RAM for a decently reasonable price. I took the bullet on it to make it my small home server for the next few months since I'm experimenting with Linux more and more every day.

**Raspberry Pi**
--------------
A raspberry Pi is a minicomputer the amount of stuff this little thing can achieve is impressive. I have been having my eyes on these micro machines for a while so I was super excited to get a hold of one and make it my home server.

**What do I plan to do with my Raspberry Pi?**
--------------
As of right now talking about what software and Linux distribution I'm planning to run on this machine are for the main purpose of learning. I will be installing softwares like:

- Ubuntu 22.04 as the base OS
- Docker - To run containers on the go I will go over how I installed it on my environment later on
- Portainer - As the container management platform for Docker which I will also cover as well
- NextCloud - Self-hosted cloud data system
- PiHole - Self-hosted DNS server
and many more things....

**What is Docker?**
--------------
Docker is an open source platform that enables developers to build, deploy, run, update and manage containers. Think about it like virtual machines VirtualBox will be your Docker in this example and the virtual machines will be the containers.

**What are containers?**
--------------
Think of containers as actual containers in real life. Containers are used to hold things right? Containers in the cloud concept hold all the necessary components, such as files, libraries, and environment variables needed to run a desired application without worrying about the platform.

**Why use Docker?**
--------------
The main reason why I decided to use Docker is because of how fast it is. The ability to fire up a container with a few commands in just seconds is unbelievable. I come from downloading windows ISOs and manually going through the process of installing them using VirtualBox. The wait time for downloading and also of installing the OS does not compare with Docker. I'm truly amazed by it.

![Alt Text](/image/containers_vs_vms.png)

**Why is Docker so fast? How does it manage to fire up machines so quickly?**
--------------
The answer to this question is simple. Let's go back to the VirtualBox and virtual machines example, when you are installing an OS with virtual machines the installation takes care of the operating system including things like the Kernel correct? With Docker, we don't have to worry about the OS or the Kernel base it is all being taken care of already. Docker virtualizes the operating system of the computer on which it is installed and running. All you have to worry about is configuration files and what exactly you want to be running on said containers. That is why Docker is so fast, this enables me to fire up instances fast and efficiently.

Now to the action of installing Docker:

First, as good practice let's update and upgrade our existing packages by running the following commands. I'm using Ubuntu 22.04 for my environment.

{{< highlight linux >}}

sudo apt update
sudo apt upgrade

{{< /highlight >}}
Now that we have our Raspberry Pi updated we can install docker by running the following:

{{< highlight linux >}}
sudo apt install docker.io
{{< /highlight >}}
That's all. Docker is now installed, to verify that the installation went through successfully we can check the docker service status which is active in our case

![Alt Text](/image/sdfsdf3.png)


Let's deploy our first container here using Docker by running:

{{< highlight linux >}}
sudo docker run hello-world
{{< /highlight >}}
This command starts a container with the hello-world image which is pulled from the docker hub. This image outputs hello world into our bash to make sure that our Docker is launching containers successfully by communicating with the Docker daemon.

![Alt Text](/image/image-2.png)


As you can see we get the Hello from Docker! text and we now have our first container running using Docker, all done in about 2-3 seconds.

Now that we have Docker running and we can deploy containers successfully we can proceed with installing Portainer to manage our containers more easily. I usually force myself to use the bash as much as possible to run, remove and overall manage Docker since you learn more by doing it that way but is also nice to have an interface to help you as well, Portainer will take care of that for us.

To get Portainer running we will be using Docker as well since we will be deploying it into a container. We can deploy Portainer into a container by using the following commands:

{{< highlight linux >}}
sudo docker run -d \
--name="portainer" \
--restart on-failure \
-p 9000:9000 \
-p 8000:8000 \
-v /var/run/docker.sock:/var/run/docker.sock \
-v portainer_data:/data \
portainer/portainer-ce:latest
{{< /highlight >}}

To further explain the logic behind this Docker command it uses the latest image from portainer-ce:latest to run in the container. We are also specifying the ports that we would like this container to be accessed from which will be 9000 (Docker's default port).

After running this command our container will start and Portainer will be installed successfully. We can verify that our new Portainer container is up and running by calling the following:

{{< highlight linux >}}
sudo docker ps
{{< /highlight >}}

This command displays the currently active and running containers in our machine.

![Alt Text](/image/image-3.png)

Now if we open up our machine's IP under port 9000 we will see our Portainer instance up and running :)

![Alt Text](/image/image-4.png)