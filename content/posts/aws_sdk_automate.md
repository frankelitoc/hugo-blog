---
title: "Automating AWS EC2s with C++ SDK"
date: 2023-02-10T23:11:13-04:00
type: "post"
image: "image/docker_linux.png"
tags: ["devops", "aws", "c++"]
draft: false
---
![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1112522279671312445/6038586442907648_1.png)

The AWS SDK for C++ is a library that makes it easy to integrate your C++ application with AWS services like Amazon S3, Amazon EC2, and many others services.

For this project I will show how to create a C++ application that we will be using to create AWS EC2 instances from our newly created application using the C++ SDK provided by AWS without the use of logging into our AWS console to perform this action, all this will be done directly from our application hence automating the overall process.

If you would like to read more about the AWS C++ SDK you can do so [here](https://aws.amazon.com/sdk-for-cpp/)

Before we can start writing some code for our application, we must first install the AWS C++ SDK into our system to use the library within our project.

The SDK is open source and hosted on GitHub [here](https://github.com/aws/aws-sdk-cpp)


To set up the AWS SDK for C++, we can either build the SDK ourselves directly from the source or download the libraries using a package manager. For this project, we will be building the SDK ourselves with CMake.

The SDK source is separated into individual packages by service. Installing the whole SDK can take quite some time (hours) so for the purpose of this project we will be installing only the EC2 packages.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1112524693598113832/image-6_1.png)

Let's get started by first installing CMake

https://cmake.org/

CMake is an open-source, cross-platform family of tools designed to build, test, and package software. We will be using it to build the SDK and also to build our application as well after.

Create a folder where we will be storing the SDK source, after doing so let's clone the repository with Git:

{{< highlight linux >}}
git clone --recurse-submodules https://github.com/aws/aws-sdk-cpp
{{< /highlight >}}

Next, let's create an extra folder to store our generated build files

{{< highlight linux >}}
mkdir sdk_build
cd sdk_build
{{< /highlight >}}

Now that we are inside our sdk_build file we can generate the build files by running CMake.

{{< highlight linux >}}
cmake ../aws-sdk-cpp -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=/usr/local/ -DCMAKE_INSTALL_PREFIX=/usr/local/ -DBUILD_ONLY="ec2"
{{< /highlight >}}

Now that we have the required files to build our SDK we can proceed with

{{< highlight linux >}}
make
make install
{{< /highlight >}}

Now that we have our SDK installed we can proceed with writing our C++ code to start our EC2 instances

We can create a new Amazon EC2 instance by calling the EC2Client’s RunInstances function, providing it with a RunInstancesRequest containing the Amazon Machine Image (AMI) to use and an instance type.

Let's first include our SDK files:

{{< highlight c >}}
#include <aws/core/Aws.h>
#include <aws/ec2/EC2Client.h>
#include <aws/ec2/model/CreateTagsRequest.h>
#include <aws/ec2/model/RunInstancesRequest.h>
#include <aws/ec2/model/RunInstancesResponse.h>
#include <iostream>
{{< /highlight >}}

After this is done let's define our main function where we will initialize our AWS SDK with InitAPI function

{{< highlight c >}}
int main()
{
   methods.
    SDKOptions options;
    options.loggingOptions.logLevel = Utils::Logging::LogLevel::Debug;
    
    //The AWS SDK for C++ must be initialized by calling Aws::InitAPI.
    InitAPI(options); 
    {
       //Create instance code
    }

    //Before the application terminates, the SDK must be shut down. 
    ShutdownAPI(options);
    return 0;
}
{{< /highlight >}}

As you can see from the above we must initialize the SDK before we attempt to use it, things like setting up our access keys for example take place during the initialization.

Now to create our instances:

{{< highlight c >}}
Aws::EC2::EC2Client ec2;
    Aws::String ami_id = "ami-06878d265978313ca" # Ubuntu Server 22.04 AMI

    Aws::EC2::Model::RunInstancesRequest run_request;
    run_request.SetImageId(ami_id);
    run_request.SetInstanceType(Aws::EC2::Model::InstanceType::t1_micro);
    run_request.SetMinCount(1);
    run_request.SetMaxCount(1);

    auto run_outcome = ec2.RunInstances(run_request);
    if (!run_outcome.IsSuccess())
    {
        std::cout << "Failed to start ec2 instance " << instanceName <<
            " based on ami " << ami_id << ":" <<
            run_outcome.GetError().GetMessage() << std::endl;
        return;
    }
{{< /highlight >}}

That simple. We specify the AMI to the variable ami_id of type Aws::String which is a char array to hold our ubuntu AMI that will be used to launch our instances, as well as the instance type which we set to t1_micro for this example.

After we set our configuration we then run our RunInstances function passing out our configuration and deploying it to the real world.

Now if we build our C++ application and run it.

NOTE: For this example, I will be using my own C++ application that uses the same logic as I explained above to showcase the results.  

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1112525734548869120/image-7_1.png)

Let's launch 3 t2.micro instances from our new application.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1112525844074733658/image-10_1.png)

We head over to our AWS Console to check our EC2 dashboard for our newly created EC2 instance.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1112525958759587901/image-9_1.png)

***Conclusion:***
--------------
The AWS C++ SDK can be used for many more different things and services to automate the overall process of managing services in AWS. This example can be built up into so many different things (which is something I'm already working on as you can see from my application which I coded using IMGUI an open-source UI to integrate into C++ applications.) I'll leave that to the reader as a challenge to expand their applications as much as possible.