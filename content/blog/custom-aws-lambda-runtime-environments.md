+++
title = 'Custom AWS Lambda Runtime Environments'
description = 'How to create a custom environment on AWS Lambda using simple scripting techniques.'
date = 2024-09-14T09:51:32-07:00
draft = false
categories = ["AWS"]
tags = ["Lambda"]
[params]
    [params.author]
        email = 'cristobal@liendo.net'
        name = 'Cristobal Liendo'
+++

_Foreword: This blog post is based on the [AWS Documentation](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-walkthrough.html) for custom Lambda environments._

On the last couple of months I have been working extensively with Lambda on different project which have involved reducing code bundle sizes so we are able to deploy code under the 250 MB limit imposed by Lambda, doing integration with other AWS services such as API Gateway, creation of a custom [lambda authorizer](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html) which performs custom authentication and authorization, and [A/B testing using a lambda@edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-examples.html#lambda-examples-a-b-testing) deployed on a CloudFront distribution.

Because of this, I have spent a lot of time reading the documentation of AWS and found out a couple of things that have grabbed my attention, principally [OS-only runtimes](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-provided.html) environments.

In this blog post I want to explain how to create a simple bash script that echoes back the event received by lambda, which can be later modified to run [any executable]({{< ref "serverless-prometheus-blackbox-exporter-1.md" >}}) you can think of.


## About AWS Lambda
AWS Lambda is a _serverless_ [^1] runtime environment that AWS provides its customers to run your application code in the cloud, without the need to worry about provisioning a server or giving major thoughts about rightsizing the hardware specifications so your code is able to run optimally and able to scale as much as you want.

As a managed environment, AWS has a list of [supported runtimes](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html) that your code can use. For example, at the time of writing of this article, AWS Lambda has support for Node.js 18 but with a deprecation date of July 31, 2025. This means that you can create applications that make use of Node.js 18 until the runtime is deprecated. After a runtime is deprecated, as a developer, you will need to worry about doing the appropriate upgrades to the next supported version of the runtime, otherwise you will not be able to create **nor** update your code, which can be scary if you have a product used by many customers!

## Custom environments
Custom environments allow you to run any piece of code that is not under the supported runtimes of AWS Lambda. This comes handy when you want to run compiled programs such as Rust, Go, or C++ on Lambda, or when you want to create a wrapper on an existing binary; which I will be talking about in my next blog post [Serverless Prometheus BlackBox Exporter]({{< ref "serverless-prometheus-blackbox-exporter-1.md" >}}).

When creating a custom runtime environment, everything boils down to some API calls:
1. Poll for an event from the [Lambda runtime API](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-api.html).
2. Propagate some information to the downstream handler such as the X-Ray trace header and Context for the lambda. For purposes of this tutorial, we won't be needing them.
3. **Invoking the function handler**, and handling the response and / or errors.

## Implementation
I am assuming you already have an AWS account, and have some familiarity using the AWS console and the AWS CLI.

### Create the code our lambda will be running
Our code is really simple, we will retrieving the event from the Lambda runtime API, echoing it to the console, and return it as the lambda response. The file needs to be called `bootstrap`.

```bash
#!/bin/bash

while true
do
    HEADERS="$(mktemp)"

    # 1. Retrieve the information of the invocation, including headers and the payload of the event.
    #    The headers will be stored in the file created when running the mktemp command.
    EVENT_DATA=$(curl -sS -LD "$HEADERS" "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/next")

    # 2. Echo it to the terminal, which will go to CloudWatch
    echo "${EVENT_DATA}"

    # 2.1 Just out of curiosity, print the contents of the temporary file
    cat $HEADERS

    # 3. From the file, get the request ID, so we are able to successfully comunicate the response
    REQUEST_ID=$(grep -Fi Lambda-Runtime-Aws-Request-Id "$HEADERS" | tr -d '[:space:]' | cut -d: -f2)

    # 4. Echo the event back as response
    curl "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/$REQUEST_ID/response" -d "$EVENT_DATA"
done
```

### Prepare the code for lambda

Change the permissions of the file, so it can be executed:
```bash
chmod 755 bootstrap
```

And create the bundle that will be used by lambda:
```bash
zip code.zip bootstrap
```

### Create the role to be used by the lambda
A role with basic execution permission is required for the lambda, so we create the trust relation policy that the IAM Role will be using:

_role-trust-policy.json_
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

The IAM Role is created:
```bash
aws iam create-role --role-name custom-lambda-runtime-role --assume-role-policy file://role-trust-policy.json
```

And finally, a basic execution policy is attached to the role:
```bash
aws iam attach-role-policy --role-name custom-lambda-runtime-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

### Create, deploy and test the lambda with the custom runtime
After creating the role, and bundling the code into a ZIP file, the lambda can be created using the following command:

```bash
aws lambda create-function --function-name custom-lambda-runtime \
    --role arn:aws:iam::{REPLACE_THIS_WITH_YOUR_AWS_ACCOUNT}:role/custom-lambda-runtime-role \
    --zip-file fileb://code.zip --handler lambda.handler --runtime provided.al2023
```

Just be sure to replace the **{REPLACE_THIS_WITH_YOUR_AWS_ACCOUNT}** with your aws account ID.

And finally, invoke your newly created lambda!

```bash
# For reference, the payload is expected to be binary encoded.
# The --cli-binary-format will handle doing the transformation.
aws lambda invoke --function-name custom-lambda-runtime \
    --payload '{"hello": "world!"}' \
    --cli-binary-format raw-in-base64-out out.txt
```

You should be able to see the following output on your terminal:
```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

and if you open the output of the file out.txt file, you will see that the lambda echoed back the input payload:
```json
{"hello": "world!"}
```

Also, if you go to the AWS console and open CloudWatch logs, you will see something like this:

---
| @message |
| --- |
| START RequestId: 2228e217-73c9-4edc-8c22-3deec9e02c46 Version: $LATEST |
| {"hello": "world!"} |
| HTTP/1.1 200 OK |
| Content-Type: application/json |
| Lambda-Runtime-Aws-Request-Id: 2228e217-73c9-4edc-8c22-3deec9e02c46 |
| Lambda-Runtime-Deadline-Ms: 1726530287087 |
| Lambda-Runtime-Invoked-Function-Arn: arn:aws:lambda:us-east-1:836716626655:function:custom-lambda-runtime |
| Lambda-Runtime-Trace-Id: Root=1-66e8c2ec-1cff8e0f58ea22f160197322;Parent=256264007522c53a;Sampled=0;Lineage=1:7101cf12:0 |
| Date: Mon, 16 Sep 2024 23:44:44 GMT |
| Content-Length: 19 |
| % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current |
| Dload  Upload   Total   Spent    Left  Speed |
| 0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0 100    35  100    16  100    19  19184  22781 --:--:-- --:--:-- --:--:-- 35000 |
| {"status":"OK"} |
| END RequestId: 2228e217-73c9-4edc-8c22-3deec9e02c46 |
| REPORT RequestId: 2228e217-73c9-4edc-8c22-3deec9e02c46 Duration: 239.44 ms Billed Duration: 240 ms Memory Size: 128 MB Max Memory Used: 25 MB |
---

which are all the echo statements we included in the lambda!

## Cleanup
To keep your AWS tidy, I recommend cleaning up the AWS resources on this tutorial:

### Delete the Lambda
```bash
aws lambda delete-function --function-name custom-lambda-runtime
```

### Delete the IAM Role
```bash
aws iam detach-role-policy --role-name custom-lambda-runtime-role \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role --role-name custom-lambda-runtime-role
```

## Closing thoughts
I find it quite interesting all the possibilities that open up with a custom lambda runtime environment. Theorically you could port any binary you want
by creating a wrapper around it (even plain bash could work), and deploy it to a custom runtime environment in lambda.

While writing this blog post, and testing out multiple executions, I was expecting the execution to be fast, under the 2 digits milliseconds range, but
the last log statement showed that the billed duration was around 240 ms:

```txt
REPORT RequestId: 2228e217-73c9-4edc-8c22-3deec9e02c46 Duration: 239.44 ms Billed Duration: 240 ms Memory Size: 128 MB Max Memory Used: 25 MB |
```

You could argue that it was due to cold start; but that's not the case. I manually invoked the lambda multiple times and the average time was around 240ms.
I am wondering if this is related to the inner-workings of cURL, which we are using to retrieve the event, and communicate back the response.

[^1]: I mean, everything has to have a server, so nothing is really _serverless_.