---
title: "Mask API Keys in AWS Lambda with Secrets Manager"
excerpt: "Mask API Keys in AWS Lambda with Secrets Manager"

header:
  image: /assets/images/Mask_API_Keys_AWS.jpg
  teaser: /assets/images/Mask_API_Keys_AWS.jpg
categories:
  - Blog
tags:
  - aws
---
This post will demonstrate how to store API key information in AWS Secrets Manager instead of as plain text in your code.

This tutorial can be followed in isolation but is also a follow-up to [this post](/blog/sam-pocket-item-readtime-tagger) (Deploying a CRON Job to AWS Lambda with SAM). 

Hence, a prerequisite is that you are familiar with AWS Lambda and have deployed code to Lambda before, particularly an app which interacts with an API.

# Step 1: Create a Secret in AWS Secrets Manager

The code from [this blog](/blog/sam-pocket-item-readtime-tagger) and [this gist](https://gist.github.com/nakulkurane/bea6efcff8a9fb3376196928f58cf8a8) was ultimately transformed into [this serverless app](https://github.com/nakulkurane/sam-pocket-item-readtime-tagger/blob/main/app.py) (which uses AWS Secrets Manager instead of environment variables).

In my case, my app used [Pocket's Developer API](https://getpocket.com/developer/), and in the gist from [Part I](/blog/sam-pocket-item-readtime-tagger), the API keys are referenced like so:

```python
consumer_key=os.environ['POCKET_KEY'],
access_token=os.environ['POCKET_TOKEN']
```

Locally, this was coming from the `.bash_profile` file on Mac. 

On AWS, in the Lambda console for a specific function, you would input the keys and values in the section shown below:

![AWS Lambda Environment Variables](/assets/images/AWS_Lambda_Environment_Variables_UI.png){: .full}

Documentation can be found [here](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html).

So, this code:

```python
consumer_key=os.environ['POCKET_KEY'],
access_token=os.environ['POCKET_TOKEN']
```

would still work! (because `os.environ` is now referencing the environment variables of the Lambda function).

In an effort to learn more about other AWS services, and even pursue better engineering practices, I found [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html).

Essentially,

> Secrets Manager enables you to replace hardcoded credentials in your code, including passwords, with an API call to Secrets Manager to retrieve the secret programmatically. This helps ensure the secret can't be compromised by someone examining your code, because the secret no longer exists in the code. 

With that in mind, let's make that Secret!

Navigate to the [Secrets Manager interface](https://us-east-1.console.aws.amazon.com/secretsmanager/home?region=us-east-1#!/listSecrets/):

![AWS Secrets Manager Secrets Home](/assets/images/AWS_Secrets_Manager_Secrets_Home.png){: .full}

and click **Store a new secret** in the top right, which will take you to the following screen:

![AWS Secrets Manager 
Store New Secret](/assets/images/AWS_Store_New_Secret.png){: .full}

By default, the **Key/value** option will be selected at the bottom, instead select **Plaintext**. This text box is where the "Secret" will be inputted. 

Still following the script I shared, it would be the *actual* values of either of the following:

```python
consumer_key=os.environ['POCKET_KEY'],
access_token=os.environ['POCKET_TOKEN']
```

Once you've done that, you will name the Secret. Do this for each "Secret."

![AWS Secrets Manager 
Store New Secret 2](/assets/images/AWS_Store_New_Secret_2.png){: .full}

This name is up to you, I named my key and token, POCKET_KEY and POCKET_TOKEN respectively, and that is what will be referenced in the "Get Secret" function.

Skip the _optional_ sections and continue clicking **Next** until you reach the last page where you will see the following at the bottom:

If not already selected, select the Python3 language and copy the _get_secret_ function/definition into your code. This is sample/boilerplate code, and you will likely need to modify it a bit to practically get the Secret and use it.

![AWS Secrets Manager 
Get Secret Boilerplate](/assets/images/AWS_Get_Secret_Boilerplate.png){: .full}

Ultimately, my modified _Get Secret_ function turned into this:

{% gist /1dbd4a7b93868f362640f437cf8f0dba %}

which can also be found in the actual [app](https://github.com/nakulkurane/sam-pocket-item-readtime-tagger/blob/main/app.py).

As opposed to hard-coding the _Secret Name_ in the function, I made it a parameter if more than one Secret needs to be retrieved, which was the case for me.

Whether you use the sample code provided by AWS or modify it like I did, you should be able to test the function locally (either running a script locally or locally invoking a deployed function). That's a wrap!

# References:

- [What is Pocket?](https://getpocket.com/en/)
- [Pocket Developer API](https://getpocket.com/developer/)
- [Configure environment variables for AWS Lambda function](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html)
- [What is AWS Secrets Manager?](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)

