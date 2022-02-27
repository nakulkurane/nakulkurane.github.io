---
title: "EC2 Quick-Launch and SSH Login (via CL)"
excerpt: "Launch an EC2 Instance without leaving the command line"
header: 
  image: /assets/images/rocket-launch.jpg
  teaser: /assets/images/rocket-launch.jpg
categories:
  - Blog
tags:
  - aws
  - ec2 
  - aws cli
  - bash
  - ssh
  - linux
---
## In this post, we’ll be going over how to quickly launch a given EC2 instance and SSH log in to it without leaving the terminal.

# Prerequisites:

- General Linux knowledge
- [AWS CLI installed and configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
- [Pre-configured EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html)

*Note*: This tutorial is directed to Unix-like OSes (not Windows) so commands may be slightly different if following on Windows.

# Step 1: Confirm Instance Status (running vs stopped)

Get the instance id of the EC2 instance which you want to log in to. You can find this in the [AWS EC2 console](https://console.aws.amazon.com/ec2/).

Click **Instances** as shown in the image below.

![EC2 Home]({{ site.url }}{{ site.baseurl }}/assets/images/AWS_Console_EC2_Home.png){: .full}

Then you will see your instances like so:

![Instance List]({{ site.url }}{{ site.baseurl }}/assets/images/AWS_EC2_Console_Instances.png){: .full}

Take note of the Instance ID (in the column highlighted below) you want to use for this exercise.

![Instance ID]({{ site.url }}{{ site.baseurl }}/assets/images/aws_console_instance_id.png){: .full}

Let’s
 assume this instance is in a stopped state; since this instance is just
 for testing, no need to have it running when not in use.

You
 can see the instance status in the AWS console above but we’re going to
 use AWS CLI commands to get the info we’re looking for.

To see the instance status of a given instance id, input the following command:

```bash
aws ec2 describe-instance-status --instance-ids *insert_instance_id_here* --include-all-instances
```

We specify the instance id following the `--instance-ids` option and we also include the `--include-all-instances` option because the `describe-instance-status` command only returns *running* instances by default.

The above command would yield:

```bash
{
   "InstanceStatuses":[
      {
         "AvailabilityZone":"us-east-1e",
         "InstanceId":"*insert_instance_id_here*",
         "InstanceState":{
            "Code":80,
            "Name":"stopped"
         },
         "InstanceStatus":{
            "Status":"not-applicable"
         },
         "SystemStatus":{
            "Status":"not-applicable"
         }
      }
   ]
}
```

# Simplify Command (Optional):

Instead of copy/pasting and/or typing in the instance ID each time, we can save the value in an environment variable.

You can set an environment variable (by typing in your Terminal window) like so:

```bash
ec2_id = *insert_instance_id_here*
```

To confirm the value was set, type and enter

```bash
echo $ec2_id
```

and it should return the value in your command line.

Great, now we can use the following command:

```bash
aws ec2 describe-instance-status --instance-ids $ec2_id --include-all-instances
```

This will return the same JSON we saw earlier:

```bash
{
   "InstanceStatuses":[
      {
         "AvailabilityZone":"us-east-1e",
         "InstanceId":"$ec2_id",
         "InstanceState":{
            "Code":80,
            "Name":"stopped"
         },
         "InstanceStatus":{
            "Status":"not-applicable"
         },
         "SystemStatus":{
            "Status":"not-applicable"
         }
      }
   ]
}
```

We can also modify the `describe-instance-status` and get *just* the value of the instance state (stopped, running, etc).

To do so, we will run a *query* on the command output to filter what is returned in the terminal.

Our command will be:

```bash
aws ec2 describe-instance-status --instance-ids $ec2_id --include-all-instances --query 'InstanceStatuses[*].InstanceState.Name'
```

The modification, as you can see is `--query 'InstanceStatuses[*].InstanceState.Name'.`

This is a JMESpath query on the response with our new output being

```bash
[
    "stopped"
]
```

Even further, we can add `—output text` to our command:

```bash
aws ec2 describe-instance-status --instance-ids $ec2_id --include-all-instances --query 'InstanceStatuses[*].InstanceState.Name' --output text
```

and we simply get

```bash
stopped
```

as our response (because we entered “text” as the output flag — it could also be json).

# Step 2: Start Instance

Ok, so since we confirmed the instance is *stopped*
 (we’ll also run this command after we start and then stop the instance 
to confirm stoppage), let’s go ahead and start the instance.

Simply enough, you will run

```bash
aws ec2 start-instances --instance-ids $ec2_id
```

and receive a JSON response like

```bash
{
    "StartingInstances": [
        {
            "CurrentState": {
                "Code": 0,
                "Name": "pending"
            },
            "InstanceId": "$ec2_id",
            "PreviousState": {
                "Code": 80,
                "Name": "stopped"
            }
        }
    ]
}
```

As we had run earlier, we can run the modified `— describe-instance-status` command:

```bash
aws ec2 describe-instance-status --instance-ids $ec2_id --include-all-instances --query 'InstanceStatuses[*].InstanceState.Name' --output text
```

until we see that the instance is *running* (give it a minute or so to launch).

```bash
running
```

# Step 3: SSH into Instance Public DNS

Now that we’ve started the instance, let’s log in via SSH. This is something you would’ve configured on [initial setup](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/get-set-up-for-amazon-ec2.html) of the EC2 Instance.

We’ll be connecting to our instance using the following command format

```bash
ssh -i /path/my-key-pair.pem my-instance-user-name@my-instance-public-dns-name
```

Instead of copying the public dns name from the EC2 console, we’ll retrieve it via the AWS CLI.

I’ve already assembled a command like so:

```bash
aws ec2 describe-instances --instance-ids $ec2_id --query 'Reservations[*].Instances[*].NetworkInterfaces[*].Association.PublicDnsName' --output text
```

This will return one line of text in your terminal with the same *Public IPv4 DNS* you would see in the EC2 console.

To stay on the keyboard (and avoid using the mouse), we can step back and rerun the previous command with some modifications.

# Save Public DNS as env variable (Optional):

Yet
 another addition to the command from directly above, we can save the 
value as an environment variable and reference that when we SSH.

```bash
aws ec2 describe-instances --instance-ids $ec2_id --query 'Reservations[*].Instances[*].NetworkInterfaces[*].Association.PublicDnsName' --output text > dns.txt && dns=$(cat dns.txt)
```

In this command, we’ve added `> dns.txt && dns=$(cat dns.txt).`

The `> dns.txt` segment will simply output the contents of our command’s response (in this case, the public dns) into a text file called `dns.txt`.

Then, we’ve chained another command using `&&`.

The `dns=$(cat dns.txt)` portion is setting the environment variable `dns` to the result of the `cat dns.txt` command (which essentially prints the contents of the file, so it will be the public dns).

Now we can run

```bash
echo $dns
```

and we will see the Public IPv4 DNS of our Instance.

Setting an environment variable isn’t *necessary*, it just minimizes the copy/paste needed for the SSH command, which will now be:

```bash
ssh -i /path/my-key-pair.pem my-instance-user-name@$dns
```

The `/path/my-key-pair.pem` part of the command should be adjusted to the local path where your .pem file is stored, and `my-instance-user-name` will be the user name configured for your instance during initial setup (likely just ec2-user).

Now you will be logged in to your EC2 instance!

# Step 4: Stop Instance

Now,
 let’s say we’re done with what we needed the instance for, so we are 
ready to stop the instance. Similar to the *start instance* command, the 
stop instance command is:

```bash
aws ec2 stop-instances --instance-ids $ec2_id
```

This will return

```bash
{
    "StoppingInstances": [
        {
            "CurrentState": {
                "Code": 64,
                "Name": "stopping"
            },
            "InstanceId": "$ec2_id",
            "PreviousState": {
                "Code": 16,
                "Name": "running"
            }
        }
    ]
}
```

and you can run the *describe-instances* command

```bash
aws ec2 describe-instance-status --instance-ids $ec2_id --include-all-instances --query 'InstanceStatuses[*].InstanceState.Name' --output text
```

until you see that the instance has been stopped.

Done!

---

# References:

- [Setup to use AWS EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/get-set-up-for-amazon-ec2.html)
- [Get Started with AWS EC2 Linux instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html)
- [AWS CLI Installation and Configuration](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
- [Filtering AWS CLI Output](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-filter.html#cli-usage-filter-client-side)
- [Describe Instances](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instances.html)
- [Start Instances](https://docs.aws.amazon.com/cli/latest/reference/ec2/start-instances.html)
- [Stop Instances](https://docs.aws.amazon.com/cli/latest/reference/ec2/stop-instances.html)
- [SSH into Linux EC2 Instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)

*Header Image by [Bill Jelen](https://unsplash.com/@billjelen?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/quick-launch?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*

---

*This post can also be found on <a href="https://medium.com/@nakulkurane/ec2-quick-launch-and-ssh-login-via-cl-782b1be039a0" target="_blank">Medium</a>. Gimme some claps!*
