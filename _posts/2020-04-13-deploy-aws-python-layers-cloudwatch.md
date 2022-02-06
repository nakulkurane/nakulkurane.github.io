---
title: "Deploying to AWS Lambda: Python + Layers + CloudWatch"
excerpt: "Deploy a Python function to AWS Lambda with Lambda Layers and CloudWatch"
header: 
  image: /assets/images/deploy-aws-lambda.png
  teaser: /assets/images/deploy-aws-lambda.png
categories:
  - Blog
tags:
  - aws
  - lambda
  - python
  - cloudwatch
  - layers
---
# Prologue

After
 integrating my Spotify account with Last FM, I wanted to start sending 
myself emails with my listening data and I eventually had a simple 
script that would retrieve data from Last FM and convert the data into 
tables (HTML).

In
 the process of testing this script, I was executing manually on my 
local machine in PyCharm. So, how can I program this to execute weekly? 
You might know about cron jobs, a utility to schedule script executions 
at specified times, and you would be right to assume that we can use 
CRON.

The
 limitation is that if we were to write the cron job on our local 
machine, in my case, Macbook, it really wouldn’t be valuable because the
 machine would have to be awake. So that’s where AWS Lambda comes in.

At
 a surface level, AWS Lambda is a managed service offered by Amazon Web 
Services which allows the user to upload code and not worry about the 
infrastructure of the servers.

This
 suggests why it is considered “serverless.” This doesn’t mean there are
 no servers at all, just that the user does not provision them or manage
 them. I figured Lambda was most appropriate for my objective.

So, before getting into the step-by-step, some prerequisites:

- AWS account to follow along (free tier)
- Basic understanding of Python/programming, cloud computing (Python libraries, Bash, SSH, SCP, AWS S3, AWS EC2)
- A Python script you want to deploy to Lambda

# Objective

I
 wanted to send an email to my Gmail account weekly on Sunday at 10 AM 
with a report of the artists, albums, and tracks I listened to the most 
in the past 7 days. My script was already connected to my Last FM and 
Gmail account when run locally and I simply needed this deployed to the 
cloud so it could run when scheduled.

We
 won’t delve into that script in this post, we will however go over some
 basics for deploying a script of your choice to the cloud (AWS Lambda 
in this case) like creating a layer for the external dependencies and 
creating the cron expression in CloudWatch.

# Step 1: Create a Lambda function

To do this, go to the Lambda service in your AWS console. Assuming you also want to deploy a Python script, select **Author from scratch,** fill in the *Function name*, and select a *Runtime (*mine was Python 3.6). In the Permissions section, it’s fine to select *Create a new role with basic Lambda permissions.* We won’t dig into that in this post. Go ahead and click *Create Function.*

Scroll down to the function code and you should see something like this:

```python
import json

def lambda_handler(event, context):
# TODO implement
   return {
     'statusCode': 200,
     'body': json.dumps('Hello from Lambda!')
 }
```


This is a basic function that will return what you see in the *return* block. If you create a *test event*
 (just name it, don’t bother changing the default JSON) and test this, 
this function should work successfully and display the following in the 
execution logs.

```json
{
   "statusCode": 200,
   "body": "\"Hello from Lambda!\""
}
```

Note that the *import json* does not cause any problems because **json** is a module built-in to Python 3.6.

In the console, you can replace the code (copy paste is fine) in the *lambda_function.py* with your own script to follow this tutorial constructively.

I'd recommend having a main method in your script as well and adding the following:

```python
if __name__ == "__main__":
   main()
   
def lambda_handler(event, context):
   main()
```

We
 won’t go in too deep on the purpose of the lambda handler or the 
parameters — if your script doesn’t take in parameters, the above should
 suffice for working in Lambda.

# Step 2: Creating and Uploading a Lambda Layer

If
 we want to import external modules into our function code, then AWS 
suggests we deploy a zip archive to Lambda Layers so we’ll do just that.

My Python script had the following import statements:


{% gist /f29d239a184359c53f24899a60b757da %}

How do we upload the above as a “layer?”

All
 these were pip installed to the local user library on my Mac and we 
don’t want to zip up every single module I’ve installed so we need to 
install just the essentials to a separate folder so we only have the 
packages we need in our zip. 

> Initially,
 I did this by simply creating a directory locally on my Mac and 
uploading that zip and eventually saw an error claiming that my Lambda 
function could not import numpy.
> 
> 
> I’m
>  pointing this out because it turns out what happened is Lambda will run
>  AWS on a Linux OS and since I had installed the packages on Mac OS, 
> this caused a problem when trying to import numpy.
> 

## Install Packages on EC2 Instance (Linux)

The quickest solution I found to this was to spin up an EC2 instance running Linux, I just picked a **RHEL8 AMI** (Amazon Machine Image) and proceeded with default settings (including generating or selecting an existing key pair, *I’d recommend generating a key pair if at least for practice*).

So, after [logging in to the EC2 instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html#AccessingInstancesLinuxSSHClient), you should check if Python is already installed, run a

```
python --version
```

to check. If it is not your preferred version or not installed, run

```
sudo yum install python3
```

to install the latest available Python distribution for your AMI. For the RHEL8 AMI, it will be Python 3.6.

Run

```
python3 --version
```

to check it was installed.

With that out of the way, create a root directory for your libraries.

I created a directory *gmail_music.*

Lambda expects external dependencies in a [certain directory structure](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html#configuration-layers-path) so we will be mimicking that on our instance. So, after making the subdirectories, you should have the following structure:

```
/gmail_music/python/lib/python3.6/site-packages
```

Now it’s time to install your Python libraries.

So
 in the home directory on your EC2 instance, you’ll start installing 
your libraries with pip like so, below example is installing the *requests* module:

```
pip3 install requests -t /gmail_music/python/lib/python3.6/site-packages
```

You’ll
 do this for each module not built in to Python (you can install more 
than one module at once, just add it to the command above).

After installing what you need, you’ll zip it up and download to your local machine.

## Zip Up Modules Installed on EC2

Still on your EC2 instance, navigate to your home directory with

```
cd ~
```

Install the zip module with

```
sudo yum install zip
```

Navigate to the root directory where your modules are installed.

```
cd gmail_music
```

Now zip the directory like so (adjust name accordingly):

```
zip -r9 ../gmail_music_layer.zip .
```

## Download Zip to Local Machine with SCP

For brevity, [follow this quick reference](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html#AccessingInstancesLinuxSCP) provided by AWS to copy files/folders from your EC2 instance to your local machine.

## Upload Zip (Lambda Layer)

This is the simple part.

In the Lambda service in AWS console, select *Layers* (probably on the left hamburger menu).

Click *Create Layer* and name your Layer.

You should see the radio buttons for *Upload a .zip file* and *Upload a file from Amazon S3*.
 I’ve uploaded my Layer to S3 but several forums would tell you it’s not
 critical where the zip is, so pick whichever option and this is where 
you will select the zip downloaded in the previous step!

I’d suggest selecting a *compatible runtime* or few as well, to match the runtimes your function modules are compatible with (3.6 in my case).

Click *Create.*

## Link Lambda Layer with Lambda Function

This is also quick and simple.

Go to your Lambda function from the hamburger menu.

In the *Designer*, under your function name, you should see a box that says *Layers.* Click this and at the bottom of the page you will see an area with a button titled *Add a layer.* On the next screen, simply select the Layer you just uploaded/created and the *Version* (this will be 1 if this was your first time uploading the layer). Click *Add* and it’s connected!

# Step 3: Create a CloudWatch Trigger to Run Lambda Function
On AWS, to run functions on a schedule, as CRON does, we will use CloudWatch.

In the Lambda Function Designer, you will see a button labeled *Add Trigger*. Click this and select *CloudWatch Events/EventsBridge* from the dropdown as the trigger.

In the *Rule* dropdown, select *Create a new rule.*

Name and describe your rule as you wish.

In the *Rule type* section, make sure *Schedule expression* is selected. The input box is where you can enter your CRON expression.

> Note that the CRON expressions on AWS have six parameters instead of five. Read more about the parameters <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html">here</a> or <a href="https://docs.aws.amazon.com/lambda/latest/dg/services-cloudwatchevents-expressions.html">here</a>.
> 

Input your expression, click *Add* and you should see it as a trigger in your *Designer* and.…that’s that!

# References

[Getting Started with AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html)

[Building Lambda Functions with Python](https://docs.aws.amazon.com/lambda/latest/dg/lambda-python.html)

[External Dependencies for Lambda Layer](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html#configuration-layers-path)

[SSH and SCP on EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html#AccessingInstancesLinuxSSHClient)

[Schedule Expressions for Rules](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html)

[Schedule Expressions Using Rate or Cron](https://docs.aws.amazon.com/lambda/latest/dg/services-cloudwatchevents-expressions.html)

**Note:** For each step in this post, there is more than one way to do the respective activities, the ones here may not be considered *best practices*
 but offer a way to deploy and test ASAP, as I wanted to. The Last FM 
script referenced in this post is a work in progress, stay tuned for a 
post or few about it.

# Regardless, hope this helped, thanks for reading!

---

*This post can also be found on <a href="https://levelup.gitconnected.com/deploying-to-aws-lambda-python-layers-cloudwatch-31c4119d3a69">Medium</a>. Gimme some claps!*