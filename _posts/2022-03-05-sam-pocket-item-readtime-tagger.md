---
title: "Deploy a Python CRON Job to AWS Lambda with SAM"
excerpt: "Deploy a Python CRON Job to AWS Lambda with SAM"
header: 
  image: /assets/images/Deploy_CRON_Lambda_SAM.png
  teaser: /assets/images/Deploy_CRON_Lambda_SAM.png
categories:
  - Blog
tags:
  - aws
  - ec2 
  - aws cli
  - sam
  - python
  - lambda
  - serverless
  - docker
---
This post will demonstrate how to deploy a cron job to AWS Lambda with SAM (Serverless Application Model).

This tutorial assumes satisfaction of some prerequisites.

# Prerequisites:

- IDE (i.e. PyCharm)
- [AWS CLI installed and configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
- [AWS SAM CLI installed and configured](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)
- [Docker installed and configured](https://docs.docker.com/get-docker/)

# Step 1: Create a function/script

I'll assume that you have a script already, but if not, we will continue with the script found below. The code below is for a cron job which will retrieve unread items from my Pocket list from the last 7 days and tags them based on time to read (or read time).

{% gist /bea6efcff8a9fb3376196928f58cf8a8 %}

Locally, create a folder to contain your script, the directory structure will be as simple as this:

```markdown
.
└── app.py
```

We will assume this script runs fine locally, but if it's something we want to run at a time interval (a cron job), we should deploy this on a cloud server. Based on the title of this post, we're going to deploy to AWS Lambda. While AWS Lambda is considered "serverless," what that really means is that a server will be instantiated only at runtime. 

We will also be using AWS SAM. SAM (Serverless Application Model) is, as it sounds, a framework to build and deploy serverless applications on AWS.

# Step 2: Create a SAM Template

In the same directory, create a file called **template.yaml.** This will be the template that our SAM command uses to build and deploy.

Create a YAML file and input the following:

```yaml
Description: An AWS Serverless Specification template describing your function.
Resources:
  Function: 
    Type: 'AWS::Serverless::Function'
      Properties:
        FunctionName: FUNCTION_NAME_HERE
        Handler: app.lambda_handler
        Runtime: python3.9
```

So, in the above case, we have a Lambda function with a Python 3.9 runtime (you can use a different runtime if you want).

Now, our directory structure should look like this:

```markdown
.
├── app.py
└── template.yaml
```

# Step 3: Build the Application

Assuming you have followed the instructions on configuring SAM locally, let's run SAM build.

```markdown
sam build --use-container
```

You will likely get the following message after running the above.

```markdown
requirements.txt file not found. Continuing the build without dependencies.
```

The build will have "succeeded" but we should resolve the above message, so we don't have issues later on.

We need to have a **requirements.txt** if we have package dependencies (third party libraries/modules) in our script. Otherwise, Lambda only knows of built-in Python 3.9 packages (or those of your runtime).

Ideally, you initialized a virtual environment and installed your modules there so the **requirements.txt** won't have every module you've ever installed. Now our directory structure should be like this:

```markdown
.
├── app.py
├── requirements.txt
└── template.yaml
```

Now if you rerun `sam build --use-container`, you shouldn't see that message. 

# Step 4: Invoke the Application Locally

After building, try running `sam local invoke`. This command will simulate the Lambda runtime environment so you can test whether the function will work once deployed.

Upon running the local invoke, you will likely get the following error

```markdown
'lambda_handler' missing on module 'app'
```

That's because we are locally invoking **app.py** as if it is a Lambda function but do not have the lambda handler in the file.

To resolve this, we are just going to add the following code

```python
def lambda_handler(event, context):
   main()
```

You do not need to know what the event or context parameters are as we won't be using them in this case. Those are relevant if your Lambda function depends on the behavior of other AWS services, which this post will not be covering.

Build and invoke again using the previously shared commands and you should get some feedback in your IDE console.

In some cases, if your function takes longer than 3 seconds to return, then you will get an error like this

```markdown
Function 'FUNCTION_NAME_HERE' timed out after 3 seconds
```

This can also be quickly resolved by configuring the timeout value in the template.yaml. 

You can see I've added `Timeout: 30` in the yaml below.

```yaml
Description: An AWS Serverless Specification template describing your function.
Resources:
  Function: 
    Type: 'AWS::Serverless::Function'
      Properties:
        FunctionName: FUNCTION_NAME_HERE
        Handler: app.lambda_handler
        Runtime: python3.9
        Timeout: 30
```

Now you can again build and invoke your function. With my shared function, we will see the following:

```
START RequestId: *REQUEST_ID_HERE*
Version: $LATEST
null
END RequestId: *REQUEST_ID_HERE*
```

This is what will show up in the AWS Lambda console log, which by default will publish to a log group in Cloudwatch (that will not be covered in this post).

Now let's actually deploy this code to AWS. To do that, we'll continue using SAM.

# Step 5: Deploy the Application
Run

```yaml
sam deploy --guided
```

You'll be led through a series of prompts now as pasted below (with the default values in the square brackets). 

```
Stack Name [sam-app]: 
AWS Region [us-east-1]: 
        
#Shows you resources changes to be deployed and require a 'Y' to initiate deploy
Confirm changes before deploy [y/N]: 
       
#SAM needs permission to be able to create roles to connect to the resources in your template

Allow SAM CLI IAM role creation [Y/n]: 
Save arguments to configuration file [Y/n]: 
SAM configuration file [samconfig.toml]: 
SAM configuration environment [default]:
```

Respond in accordance with your preferences and the deployment should begin!

Hang tight. We're *almost* there. There's yet another setting we haven't configured in the template so you should see an error like this:

```
An error occurred (ValidationError) when calling the CreateChangeSet operation: Template format error: Unrecognized resource types: [AWS::Serverless::Function]
```

To resolve this, we will add a **Transform** value to our YAML, seen at the top of the file:

```yaml
Transform: 'AWS::Serverless-2016-10-31'
Description: An AWS Serverless Specification template describing your function.
Resources:
  Function:
    Type: 'AWS::Serverless::Function'
    Properties:
      FunctionName: FUNCTION_NAME_HERE
      Handler: app.lambda_handler
      Runtime: python3.9
      Timeout: 30
```

Per [AWS](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/transform-aws-serverless.html),
> The `AWS::Serverless` transform, which is a macro hosted by CloudFormation, takes an entire template written in the AWS Serverless Application Model (AWS SAM) syntax and transforms and expands it into a compliant CloudFormation template.

This time, the deployment should succeed.

Now, if you have the AWS CLI installed, you can invoke the deployed function (not a local invocation) from the command line with 

```yaml
aws lambda invoke --function-name FUNCTION_NAME_HERE output.txt
```

The **output.txt** flag is a required outfile command which pipes the log output of the function call to the file. Our function isn't returning anything, so it will just be a null value.

If you run the above command, and it succeeds, you will see

```yaml
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

in your terminal. 

If you cat the output.txt with `cat output.txt`, you will see `null`. 

That means it worked!

Now that we've invoked the function, let's create that cron schedule.

# Step 6: Configure a CRON Schedule
The syntax for a cron expression on AWS can be found [here](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html).

I've added a schedule below (in the Events section) of every Sunday at 2 PM.

```yaml
Description: An AWS Serverless Specification template describing your function.
Resources:
  Function: 
    Type: 'AWS::Serverless::Function'
    Properties:
      FunctionName: FUNCTION_NAME_HERE
      Handler: app.lambda_handler
      Runtime: python3.9
      Timeout: 30
      Events:
        Schedule:
          Type: Schedule
          Properties:
            Name: SCHEDULE_NAME_HERE
            Schedule: cron(0 14 ? * SUN *)
```

Your schedule (and name) can be to your preference.

Now that we've added the cron schedule, we can build and deploy again (you should recall how to do so from earlier in this post) -- **and that's it!**

# References:

- [Install and configure AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
- [What is AWS SAM?](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html)
- [Install and configure AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)
- [Install Docker](https://docs.docker.com/get-docker/)
- [Python Virtual Environments](https://realpython.com/python-virtual-environments-a-primer/)
- [What is the requirements.txt file?](https://www.idkrtm.com/what-is-the-python-requirements-txt/)
- [What is Pocket?](https://getpocket.com/en/)
- [Pocket Developer API](https://getpocket.com/developer/)

