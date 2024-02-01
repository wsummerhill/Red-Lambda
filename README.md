# Red-Lambda

Red Lambda is an AWS CloudFormation template that automatically deploys red team infrastructure in the cloud.
My blog post covers more details about the background of this project.

## Overview

A basic red team infrastructure is deployed using the `lamda.yml` AWS cloudformation template.
Infrastructure includes:
* VPC and Subnet
* EC2 with SSM Enabled to host a C2
* Lambda Function to act as redirector
* Lambda Function URL to expose redirector to internet

The lambda function python code is embedded in the cloudformation template.
However, I've copied it in the `lambda.py` file to review. 

<span style="color:blue">*Personally, I just manually create the Lambda function myself in AWS console using the code from `lambda.py` as its pretty quick to do.*</span>

### :warning: **Changes required of January 2024:** :warning: <br />
The AWS Lambda function can now be run with new versions of Python such as 3.11 or 3.12 with small modifications. Changes to the code and process have been detailed to allow the support of the Python `requests` library in newer versions of Lambda and Python since Python 3.7 is no longer supported and the new versions [don't provide support by AWS](https://aws.amazon.com/blogs/compute/upcoming-changes-to-the-python-sdk-in-aws-lambda/) for the Python `requests` library naturally, so we have to add it ourselves. Steps detailed in [below section](#lambda-redirector-changes-required).

### Tested C2 Frameworks
Frameworks tested while developing this tool include:
* **Cobalt Strike** setting data to be sent in message body
* **Mythic** using the *Athena* agent using the *http* profile
* **Sliver** using the *http* listener (encountered some throttling issues during high performance)
* **Covenant** using the http listener with SSL enabled (upload a random pfx cert)

## Prerequisites

1. AWS CLI \
   https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html
2. AWS CLI Session Manager Plugin \
   https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html

## Deploying/Deleting Infrastructure

From the command line, run the following command to start the creating infrastructure:
```
aws cloudformation deploy --stack-name Red --template-file red.yml --capabilities CAPABILITY_NAMED_IAM
```

From the command line, run the following command to *destroy* the infrastructure:
```
aws cloudformation delete-stack --stack-name Red
```

## Retrieving Resource Information

After deploying the infrastructure, use the following aws-cli commands to find necessary information.

List EC2 instances:
```
aws ec2 describe-instances --query 'Reservations[*].Instances[0].{Name:Tags[?Key==`Name`].Value|[0],Instance:InstanceId,IP:PublicIpAddress, State:State.Name}' --output table
```

## Accessing Infrastructure

**No need for internet facing SSH systems or jump boxes!** \
Simply use AWS Systems Manager (SSM) from the AWS CLI to interact with your infrastructure.

Access any of the EC2 instances by using the AWS CLI through the `aws ssm` command:
```
aws ssm start-session --target <instance id>
```

Use SSM to port forward to your local machine:
```
aws ssm start-session --target <instance id> --document-name AWS-StartPortForwardingSession --parameters "portNumber"=["80"],"localPortNumber"=["1234"]
```
*Note: This is helpful if your C2 has a web management interface or teamserver port that must to accessed locally.*

## :star: Lambda Redirector Changes Required :star:

**AWS Lambda no longer supports Python 3.7, and newer versions of Lambda don’t support the `requests` library in Python.**<br />
Follow these steps to fix this issue:
1. Create an AWS Lambda Layer (One time setup only): The Lambda Layer will allow us to import the Python `requests` library code ourselves which our redirector function will be using.
    - Use [this post](https://www.keyq.cloud/en/blog/creating-an-aws-lambda-layer-for-python-requests-module) to create a zip file of the Python `requests` library which we'll upload as a Lambda Layer in AWS.
        - *Note: Be sure to match your Python version used to create the `requests` library with the Python environment selected for your Lambda function*
    - Once you have your `requests` zip package from the above step, in AWS go to **Lambda → Layers → Create Layer → Upload the requests.zip package - Select Python 3.11** (or whatever Python version you’re using) **→ Create**
2. Modify your code function to use the updated code in `lambda.py` from this repo to support the `requests` library in the newer version of Python
3. Change Lambda runtime environment to Python 3.11 (or whatever version your `requests` library is using).
4. Now you're good to go! Test it out to confirm it works.
    - You can use `curl` to hit your endpoint and view logs on the C2 server to ensure the request are coming through without errors.

## Final Topology
![AWS Topology](/red-lambda-aws-topo.png)
