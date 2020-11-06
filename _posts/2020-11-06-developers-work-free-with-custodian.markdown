---
layout: post
title:  "Securing your cloud environment with Cloud Custodian"
date:   2020-11-06 07:00:00 -0300
categories: aws cloud custodian guardrails security
---
Check out this simple cross-account architecture to create guardrails across your organization and let your developers work without worrying about creating resources that could impact your organization.

Average reading time: 20 minutes

## What is Cloud Custodian?

[Cloud Custodian](https://cloudcustodian.io/) is an open-source, CNCF sandbox project, that helps you to keep a compliant cloud environment (AWS, GCP, or Azure) by monitoring your resources and checking for compliance, applying action on them to ensure your environment is safe.

You can use Cloud Custodian to avoid storage from being public, monitor your environment for exposed topics, delete unencrypted resources, and more!

You can check the [Cloud Custodian documentation](https://cloudcustodian.io/docs/index.html) for more information on the tool.

Also, you can check their Github repo [here](https://github.com/cloud-custodian/cloud-custodian)

## How are we going to secure our environment?

In this integration, we will use Cloud Custodian in an AWS Organization to monitor the creation of any S3 Buckets in any Developer Account that don't have access logs enabled. If any new buckets created fail to comply with this policy, then it is instantly deleted.

![Custodian flow](/assets/images/20201106-developers/custodian-flow.png)

Optionally, you can configure more actions, like posting to an SNS topic and send push notifications or sending an email to notify users about the deletion.

## Architecture

Here is the resulting architecture (mind that there might be hundreds or thousands of Developer Accounts):

![Architecture](/assets/images/20201106-developers/Cloud-Custodian-Architecture.svg)

[CloudCraft page with the architecture and docs for each component](https://app.cloudcraft.co/view/3bffe4cf-f17a-4907-973b-b0ae1bd98d16?key=96XwyeACP5v6v8Gx6VWfww)

You can find the Terraform template to deploy the Custodian Account infrastructure here https://github.com/gelouko/c7n-deployments/tree/master/centralized-arch

### Explanation

There will be 2 types of accounts:
  - **The Custodian Account**. This is the account where the core infrastructure will be deployed, like Lambdas and EventBridge Rules (automatically created by Cloud Custodian).
  - **The Developer Account(s)**. This is the account (or accounts) that will be monitored by Cloud Custodian alongside the Custodian Account. They need a CrossAccountRole that will perform all actions on the account's resources and an EventBridge Rule to forward the necessary events to a centralized event bus in the Custodian Account.

To be able to work with Amazon EventBridge events, we have to create a CloudTrail Trail to create rules that will catch AWS Management Events. In this architecture, we have created a single Organization Trail in the Master Account.

When we create guardrails based on EventBridge events in Cloud Custodian, it deploys a Lambda to be triggered with the specified Cloudtrail API event for each policy created, so keep in mind that the number of Lambdas deployed can be high considering the number of guardrails you create.

The basic flow of events for our example is the following:

1. A Developer creates an S3 Bucket in the Developer Account
2. The CloudTrail Trail will detect the API Call, send it to the Developer Account's event bus, and thus trigger the EventBridge Forwarder Rule in the Developer Account (make sure this rule is protected from deletion with an SCP or similar)
3. The Event Forwarder Rule will forward the event to the Custodian Account's default event bus. If it fails for some reason, it will retry, and after some retry attempts, it will send the event to an SQS queue that will serve as DLQ in the Custodian Account
4. An EventBridge Rule (created by Cloud Custodian when the policy is deployed) is triggered with the action information from the Developer Account
5. The Rule will trigger a Lambda Function (created by Cloud Custodian when the policy is deployed)
6. The function will check the event data for compliance and will assume the CrossAccountRole in the Developer Account if it needs to perform any actions
7. The CrossAccountRole will perform any actions defined in the Custodian policy in the Developer Account.
8. The Lambda function finishes.

It's important to note that currently, Cloud Custodian only works with the default event bus, so we have to change the Custodian Account's default event bus to make it accessible from within the current organization.

Now I'll show you how to deploy your first policy. If you want to follow up with hands-on, just deploy the terraform templates provided by this [Github repo](https://github.com/gelouko/c7n-deployments/tree/master/centralized-arch).

## Deploying your first policy

Now that we have our accounts ready, we need to deploy our first Cloud Custodian policy.

### Installing Cloud Custodian

Cloud Custodian is installed with Python/Pip. To separate your environment from other dependencies, it's a good idea to install Cloud Custodian using a virtual environment:

```bash
$ python3 -m pip install virtualenv   # installs virtualenv using Python 3
$ python3 -m venv custodian-env       # create a virtual environment for Cloud Custodian
$ source custodian-env/bin/activate   # activate the virtual environment
$ pip install c7n                     # installs Cloud Custodian in the virtual environment
$ custodian version                   # prints the installed version of Cloud Custodian
```

Before we continue creating our first policy, feel free to get more familiar with [Cloud Custodian's Documentation](https://cloudcustodian.io/docs/quickstart/index.html).

### Creating your first policy

In order to do that, create a new yaml file. We'll name it `policy.yml` (any name is allowed) and add the following code (change the values for your account's):

```yaml
policies:
  - name: s3-delete-buckets-without-access-logs
    resource: s3
    mode:
      type: cloudtrail
      role: CloudCustodianExecutionRole
      member-role: arn:aws:iam::{account_id}:role/CloudCustodianCrossAccountRole
      kms_key_arn: arn:aws:kms:us-east-1:{account_id}:key/7br86231-03bb-437d-9220-28a2d8272c15
      memory: 128
      timeout: 15
      execution-options:
        output_dir: s3://c7n-output-bucket
      events:
        - CreateBucket
    filters:
      - type: bucket-logging
        op: disabled
    actions:
      - type: delete
```

> If you have created the environment from the terraform repo, there's a working policy in the README file with a little less stuff in it.

Let's break down the configuration:

- **policies**: this is a list of policies, each of them will deploy a Lambda Function (the guardrails themselves).

- ***name***: The name of the policy. Mind that for `mode type: CloudTrail` policies, this name will be the suffix of the Lambda function and EventBridge Rule created.
- **resource**: This is the type of [resource](https://cloudcustodian.io/docs/aws/resources/index.html) that will have filters and actions applied.
- **mode**: information about the policy execution
  - **type**: This defines the mode type you will run your policy. For more info, check [this page](https://cloudcustodian.io/docs/aws/resources/aws-modes.html?highlight=modes)
  - **role**: The Lambda execution role that will be used for this policy. In the current architecture, it just needs BasicLambdaPermissions (for logging), S3 PutObject permissions for the output bucket, and sts:assumeRole permission on the CrossAccountRole for any accounts in your Organization
  - **member-role**: The cross-account role that will be deployed in the Developer Accounts.
  - **kms_key_arn**: A KMS Key that will be used to encrypt the Lambda environment variables
  - **memory**: The Lambda allocated RAM
  - **timeout**: The Lambda's timeout
  - **execution-options**: some options to configure the policy execution when the Lambda is triggered. By default, Custodian sends output to a random /tmp/ directory, so I've changed this to send outputs to an S3 Bucket in my account.
  - **events**: The EventBridge event information that will trigger this policy.

For our purpose, the only required attributes are `policies`, `name`, `resource`, `mode` and `mode.type`. For the other, Custodian may apply default values. There is a lot of information about each of these attributes [here](https://cloudcustodian.io/docs/aws/lambda.html).

Now, just run

```bash
$ custodian validate policy.yml
```

This will check for any errors in the policy template. There should be none. Then, run the policy using the following command:

```bash
$ custodian run -s output policy.yml
```

*Note*: you need AWS CLI installed and configured for the Custodian Account with a default region in your machine (or use the `--region` param). Your user/role should have access to deploy Lambda Functions, pass roles to Lambda and create EventBridge Rules (plus some of the permissions written in [this documentation](https://cloudcustodian.io/docs/deployment.html)).

If your command runs successfully, you will be able to check the created Lambda function and EventBridge Rule in the AWS Console.

![Lambda Function screenshot](/assets//images/20201106-developers/lambda-print.png)

Congratulations! You've deployed your first real-time guardrail policy!

You can test it out by creating an S3 Bucket in the Developer Account without access logs and see it be deleted within ~15 seconds (less if the Lambda is warm)

## Alerting your developers!

Okay, it's cool that your environment is more secure by enforcing access logs in the buckets, but it'd be quite weird for a Developer to create an S3 Bucket and it just vanishes out of nowhere, right?

Enters [Custodian Mailer](https://cloudcustodian.io/docs/tools/c7n-mailer.html)!

The Custodian Mailer (a.k.a c7n-mailer) is a plugin for Custodian that enables user notification by email!
Note that Cloud Custodian can notify SNS queues without c7n-mailer, but c7n-mailer helps with templating and other cumbersome email configuration.

As this post is already huge, I'm just leaving the documentation for enabling the notifications.

Feel free to follow [these steps](https://cloudcustodian.io/docs/tools/c7n-mailer.html) to implement them.

## Monitoring your environment

This example architecture will start monitoring and "fixing" your environment at the point where you've deployed your policy.

To monitor the health of your Guardrails Lambda Functions, you can check CloudWatch's default Lambda metrics for the Functions. With these metrics, you can check the Duration, Errors, ConcurrentExecutions, Invocations, and Throttles metrics:

![CloudWatch metrics screenshot](/assets/images/20201106-developers/metrics-screenshot.png)

![CloudWatch logs screenshot](/assets/images/20201106-developers/logs-screenshot.png)

Monitoring the whole environment at once is the drawback of this architecture, as it is not so straightforward. Each time the Lambda executes, it will override the data in the S3 Bucket, as it uses a common key to store the output.

Here are three ways that you can monitor your environment (depending on your needs, there might be better solutions):

1. **(Periodically, easier)** run your policies in pull mode, without any actions. That will fetch the resources that are uncompliant and log them to a defined output bucket defined in the `run` command, this time, without replacing the values. You will need [c7n-org](https://cloudcustodian.io/docs/tools/c7n-org.html) to run this through your whole organization.

2. **(Almost real-time or periodic)** Use CloudWatch Insights to parse the function logs and use it to create metrics for your environment + use these metrics to create an Organization Compliance Dashboard

3. **(Real-time, cumbersome)** Use s3 event notifications to trigger a function on the createObject event and create metrics based on the event's data.

If you find better ways to monitor the environment using this architecture, feel free to share, and I'll add them here with your credits!

## Cost

This architecture is quite straightforward, and the cost will be based mainly on your Lambda functions, cross-account events, and ingestion/storage of CloudWatch logs and S3 Buckets (if using one for outputs).

Check **Lambda functions' pricing** for your region [here](https://aws.amazon.com/lambda/pricing/).

Check **Amazon EventBridge's pricing** for your region [here](https://aws.amazon.com/eventbridge/pricing/).

Check **S3 storage's pricing** for your region [here](https://aws.amazon.com/s3/pricing/).

Check **CloudWatch logs' pricing** for your region [here](https://aws.amazon.com/cloudwatch/pricing/).

Keep in mind that some costs (e.g storage logs) will be accumulated if you don't do periodic cleanups. Also, you should have S3 lifecycle policies to ensure your costs keep at the lowest possible.

I've also not added the CloudTrail price as you probably already have a Trail for your Organization, and thus the CloudTrail cost of adding Custodian would be none. Just in case, you can check out the [CloudTrail pricing](https://aws.amazon.com/cloudtrail/pricing/) to see how much you'd pay for management events (if you don't already have it)

There should be also some small extras for the DLQ queue, data transfer, Lambda code storage, metrics, etc. so take this as a "rough" calculus.

Okay, so let's set some examples:

> You have an Organization with 100 accounts
>
> 10 engineers work on each account and they change buckets 5 times a month each.
>
> You have 10 real-time policies that check for different modify bucket events. Each event triggers 1 function.
>
> The total of events triggered for each account will be 10*5 = 50 events/month (1)
>
> The total of events sent to the Custodian Account is 100*50 = 5,000 events/month (2)
>
> Your lambdas have 128MB of memory and execute during an average of 10s. The total of Lambda executions in your account is 1*5000 = 5,000 executions/month (3)
>
> You are not sending logs to an S3 Bucket, but each execution will add 1.5KB of log data, so that's ~73.25MB of data ingestion/month. (4)
>
> Your final (average) cost for the us-east-1 region will be:
>
> (1) 50 inner-account events/month, per account: U$ 0
>
> (2) 5,000 cross-account events/month: U$ 0.005
>
> (3) 5,000 Lambda executions (128MB, 10s/execution): U$ 0.001 (requests) + U$ 0.142 (Duration) =~ U$0.143
>
> (4) 73.25MB of data ingestion into CloudWatch Logs: ~U$0.0358 (ingestion) + U$0.0012 (archival) =~ U$0.037
>
> That's a total of 0.005 + 0.143 + 0.037 = U$ 0,185/month

Now let's see the pricing for policy-intensive, enterprise platforms

> You have an Organization with 1000 accounts
>
> 20 engineers work on each account and they change monitored resources 20 times a month, each engineer.
>
> You have 200 real-time policies that check for different events, and each event triggers ~10 different functions.
>
> The total of events triggered for each account will be 20*20 = 400 events/month (1)
>
> The total of events sent to the Custodian Account is 1,000*400 = 400,000 events/month (2)
>
> Your lambdas have 128MB of memory and execute during an average of 10s. The total of Lambda executions in your account is 10*400,000 = 4,000,000 executions/month (3)
>
> You are not sending logs to an S3 Bucket, but each execution will add 1.5KB of log data, so that's ~5,73GB of data ingestion/month. (4)
>
> Your final (average) cost for the us-east-1 region will be:
>
> (1) 400 inner-account events/month, per account: U$ 0
>
> (2) 400,000 cross-account events/month: U$ 0.4
>
> (3) 4,000,000 Lambda executions (128MB, 10s/execution): U$ 0.80 (requests) + U$ 83.32 (Duration) =~ U$84.12
>
> (4) 5.73GB of data ingestion into CloudWatch Logs: ~U$2.865 (ingestion) + U$0,172 (archival) =~ U$3.04
>
> That's a total of 0.4 + 84.12 + 3.04 =~ U$87.56/month

I didn't take in consideration free tiers as I wanted to get a real price of it, considering you're already spending your free tier with other infrastructure. (i.e the first example would keep in the free tier for CloudWatch).

That's quite cheap considering the benefit of the product.

Another interesting thing to point is that the cost of this architecture would go mostly to the Custodian Account, and if you need to split these costs between accounts, you have to create some kind of automation to check the logs or outputs and gather cost information based on the number of events sent per account (that'd be kind of harsh).

For this architecture, I’d recommend going with an equal division of costs for each account, as the costs are not huge.

If you want to strictly split costs, there might be other deployment options.

## Limits

Now we're talking about it! How many times didn't you have problems scaling your architectures? Reached an AWS quota? (OMG Organizations requests/s!)

So, let's avoid surprises as much as we can! Let's check some of the main quotas for the us-east-1 region:

Checking some EventBridge's quotas:
> 2400 PutEvents/s per region! Considering the use of Cloud Custodian, this shouldn't be a problem, but be aware if you have other applications in the same region that are intensively putting events on EventBridge.

> The policy size of the event bus is 10240 characters. So, be careful when adding too many accounts, as you can easily reach this limit! Use the `DescribeEventBus` API to find out the current size of your policy.

> Event pattern is 2048 characters long. This can be a tough limit for the forwarder if you are creating fine-grained patterns, and thus might require more than one Forwarder.

> You can have 300 rules per event bus. So considering the current Cloud Custodian limitation of only using the default event bus, you would be able to have at most 300 real-time policies in your account. This might require that you "merge" some policies into 1, but it can be easily solved with some PRs on Cloud Custodian if that turns out to be a problem :)

> You can have 4500 rule invocations per second. This would only be a problem if you have other event-intensive applications or a REALLY big organization, as usual developers wouldn't be able to change resources in AWS so many times per second, even using IaaC tools.

Now, let's take a look at some Lambda quotas:
> 1,000 concurrent executions. That might be a problem if you are centralizing your Lambda functions. But the good news is that this number can be increased to hundreds of thousands, so if you realize you are reaching this limit, don't hesitate on requesting an increase to AWS! Keep in mind that AWS reserves 100 Lambda executions for pooling, so the number you see in the console might not be the one you have!

> ENIs per VPC: 250. (But it can be increased up to some more hundreds). This might be a problem if you are using VPCs for your Lambdas. If you are deploying your Lambdas inside a VPC within 2 subnets, then you will be able to add a max of ~125 Lambdas before requesting an increase (this is considering you have nothing more using ENIs in your Custodian Account!)

Finally, for CloudWatch:
> 150 PutMetricData transactions per second. That might be a problem considering the amount of events and policies you have, but this number can be increased by AWS.

> 5 PutLogEvents transactions per second per LogStream or 1500 tps / account / region. Both of them might be a problem if you have a lot of Developers working on products triggering the same policy at the same time, or just all Developers are intensively changing their accounts' infrastructure.

I don't think the other quotas should be so impactful, but feel free to take a look at your account's quotas. Your Organization might have specific requirements that would reach any other quotas.

## Wrap-up and two more adoption solutions

Finally, we are done!

You just learned how to deploy a Cloud Custodian architecture to enforce security guardrails in your Organization's accounts to enable developers to create AWS resources without worrying about creating vulnerabilities in their environment!

As some wise people say: "if you have one solution, you have no solutions". So here are two more architectures you could use to implement Cloud Custodian:

1. **Use c7n-org to periodically run the guardrails in the account**
  - You would have all policies in Pull Mode run at once using c7n-org. This might take some time considering the number of vCPUs your machine has, but it's effective for all resources in the accounts.
  - It is also really easy to monitor these resources as there will be a single log and storage output.
  - Pricing will be based on the time you take to run Custodian, if you are using an EC2 for it. If you run Custodian from your own machine, even this cost would be none.
  - BUT this wouldn't guarantee compliance in real-time. You might have some time windows that an unadvised developer could create a public bucket and leak some data before you realize the bucket existed.
  - It's quite a simple and straightforward solution, as it just requires Cross Account roles in the Developer accounts.


2. **Use decentralized lambdas**
  - Instead of creating the centralized Lambdas, you invoke Cloud Custodian Lambdas for each account.
  - This way it is easier to separate costs for each account, but it adds A LOT of "trash" in the Developer Account, as it might add hundreds of Lambda functions and EventBridge Rules in each of them, thus increasing the risk of reaching the EventBridge Rules quotas and some other AWS hard limits.
  - You would need a cross-account role to deploy the Lambdas and also a role in each account to be used by the Lambda functions.
  - Monitoring can be easy if you centralize the lambdas' logging, but you still have the problem with the S3 output that would be overriden (so you would depend on the logs for monitoring).
  - You'd also need some SCPs to avoid deletion of the custodian Lambdas and Event Rules.
  - The only thing that is easier in this architecture is cost-splitting. So the only case where I'd advise this architecture is if you need strictly cost-splitting between accounts while ensuring real-time compliance (but I'd try to avoid this architecture)

Thank you for your time!

This is my first post, and I'd really appreciate feedback for the future posts :) (check my [contact](/contact) email)

I hope you liked it!

**Keep creating, bros _O/**
