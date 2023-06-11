---
title: "Serverless backend architecture - AWS Cloud Resume"
date: 2023-03-06T23:11:13-04:00
type: "post"
image: "image/docker_linux.png"
tags: ["devops", "aws", "python"]
draft: false
---
![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1107442336536985600/WnS8Blk.png)

When you think about serverless architectures in the cloud what exactly comes to mind? If you have not researched more into this topic you would probably think there is a way to run resources without servers because it's called a server-LESS architecture.

Well not necessarily, when I mention a serverless architecture I mean we are building an architecture that requires no server management, we don't see the server, or manage it. We have our specific resource running without the worry of server management, in other words, it just works.

I will be showing you how I used AWS services like DynamoDB, Lambda Function, and API Gateway to build a serverless backend architecture for my AWS cloud resume.

Our architecture diagram will be the following:

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1107442336536985600/WnS8Blk.png)

As you can see is fairly simple, let's now explain it in further detail. Our resume is currently hosted in an S3 Bucket which is the frontend of our project, now I decided to add a way to show the number of visitors my page has, I had to make sure that this will have little to no cost and also to have a way to store the number of visitors somewhere.

The diagram above shows how we use an API Gateway trigger to a Lambda function that runs a python code that retrieves the number of visitors which is an integer stored inside a DynamoDB table after this is done the function then proceeds to increase the number since technically every time this API call is made it would be a new visitor (we will need some Javascript code inside our frontend which I will discuss later on). After all of this process is done the Lambda function then sends the updated number back to our site which is then displayed to our visitors.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1107442532918493184/image-6.png)

Inside our DynamoDB table we must have a key and value pair for our Lambda function to retrieve the number from. In this example, I used the partition key mainid, assigned it a value of 1, and added a new attribute of an integer that holds the number of our site visitors.

Now that we have our DynamoDB table ready now is time for the main part of the puzzle, the Lambda function. As I mentioned above we will be using a Python Lambda function since our code will be written in Python.

After having our function created before continuing with writing our code we have to make sure that the IAM role created for our Lambda function has the correct permissions policies to retrieve/update items inside our DynamoDB table.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1107442668935585802/image-7.png)

For this case, I added the AmazonDynamoDBFull policy into the role which technically is not the smartest idea. For security purposes, it would be better to create a policy that only grants usage to the VisitorCount table since in this case this role now has access to all the tables inside our AWS account, for this situation I will keep this way since this is my personal account.

Now let's proceed to write our Python code to manipulate the data inside our DynamoDB table, we will be using the Boto3 library which is the standard AWS SDK for Python.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1107442768378343505/image-8.png)

{{< highlight python >}}
import json
import boto3

#Setup our boto3 DynamoDB resource 
dynamoDB = boto3.resource('dynamodb')
table = dynamoDB.Table('TestTable')

def lambda_handler(event, context):
    
    #Let's first obtain the latest count from the table
    response = table.get_item(Key={'mainid': 1})
    
    #Once we obtain the current count we declare a new variable and add + 1 into it
    Newcount = response['Item']['views'] + 1
    
    #Put the latest value into the table
    response = table.put_item(Item={'mainid':1, 'views':Newcount})
    
    #Return 200
    return {
    'statusCode': 200,
    'body': Newcount
}
{{< /highlight >}}

Now that we have our Python code set up and ready we can now add our API Gateway trigger in front of it.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1107442975270785144/image-9.png)

And that will do it, now every time we access our API Gateway URL provided we will see our visitor count integer displayed.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1107442996854673568/image-10.png)

We can now use Javascript inside our site to invoke the API call every time a new visitor has joined our site and display it to our users.

![Alt Text](https://cdn.discordapp.com/attachments/953834757257580554/1107443015498342471/image-11.png)