---
title: "Containerizing a web application with Docker"
date: 2023-02-24T23:11:13-04:00
type: "post"
image: "image/docker_linux.png"
tags: ["linux", "docker", "python", "flask"]
draft: false
---

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1114741647901732935/flask-docker_1.png)

Have you ever wondered if there was a way to code web applications using Python? Flask is just that, it is a web framework that lets you develop web applications using Python.

It also has many nice features like URL routing, template engine, and so on. I will be containerizing my simple Flask web application, you can find the source code of the application code here.

This is an extremely simple web application that uses the home.html inside the repository to display it to the end user each time they connect to the local public IP address using port 8080

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1114742642761281576/image-12.png)

As I said super simple. Now, what would be the process to containerize this web application? When it comes to containerizing applications with Docker we must first create our Dockerfile which will be used to build the Docker image and finally we can proceed with running the image inside our Docker host to have a container with our web application code up and running.

The most important thing we must know before creating our Dockerfile is to make a list of the dependencies that are required to run our application. In this case, it will be the following:

- Python
- Flask

With that being said we can proceed to build our Dockerfile to ensure we have these dependencies installed once an image has been built.

{{< highlight Dockerfile >}}
# Specify the OS we are using to run our web app from
FROM ubuntu

# Commands to install our dependencies
RUN apt-get update

# Install Python
RUN apt-get install python3 -y

# Install Python Pip
RUN apt-get install python3-pip -y

# Copy our Flask python file and the home.html
RUN mkdir /opt/templates
COPY app.py /opt/
COPY home.html /opt/templates/

ENTRYPOINT FLASK_APP=/opt/app.py flask run --host=0.0.0.0 --port=8080
{{< /highlight >}}

Now we have everything set up to build our Docker image and run our web application inside our Docker host within containers.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1114743174599032925/image-14.png)

As shown above we can now go ahead and build our image with the following command:
{{< highlight linux >}}
sudo docker build -t webapp1 .
{{< /highlight >}}

After our image has been built we can proceed with running our container.

{{< highlight linux >}}
sudo docker run --name webapp webapp1
{{< /highlight >}}

Now if we launch our site under port 8080 we will be able to see our web application up and running inside our container

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1114743592544636938/image-16.png)