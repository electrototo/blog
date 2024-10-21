+++
title = 'AWS Multi-Region Deployments'
description = 'Multi-region deployments in AWS using CDK, CodePipeline, GitHub webhooks, and CloudFormation'
date = 2024-09-30T22:31:58-07:00
draft = true
tags = ["cdk", "aws", "lambda", "codepipeline"]
categories = []
[params]
    [params.author]
        email = 'cristobal@liendo.net'
        name = 'Cristobal Liendo'
+++

As a full-stack developer my day-to-day routine consist mostly on developing features, and making sure they are able to be launched successfully to a global audience.

In 2024, with more than half of the global population [^1] having access to the internet, I truly believe that being able to provide a good browsing experience is a **must**
for any newly created web product, specially if you are using any big cloud computing provider, as they usually have infrastructure available in different geographical regions.

## Overview

In this blog post I will be discussing how to perform and automate multi-region deployments in AWS using CDK and CodePipeline. 


### Application
The high-level overview of the architecture used in this blog post looks like the following:

{{< figure src="images/global-endpoint.png" width="75%" caption="Multi-Region deployment using GeoLocation DNS records" >}}

The application will be deployed to three different regions: `us-east-2`, `us-west-2`, and `sa-east-1`, and will be backed by lambda.

The code is pretty simple, as its output just contains the region name where the lambda is being invoked from:

_[lib/lambda_handler.ts](https://github.com/electrototo/aws-multi-region-deployments/blob/master/lib/lambda_handler.ts)_
```typescript
import { Context, APIGatewayProxyResultV2, APIGatewayProxyEventV2 } from 'aws-lambda';

export const handler = async (event: APIGatewayProxyEventV2, context: Context): Promise<APIGatewayProxyResultV2> => {
    console.log(`Event: ${JSON.stringify(event, null, 2)}`);
    console.log(`Context: ${JSON.stringify(context, null, 2)}`);

    return {
        statusCode: 200,
        body: JSON.stringify({
            message: `Hello from ${process.env.AWS_REGION}!`
        })
    }
}
```

To reach the lambda, and be able to invoke it, I placed an API Gateway in front of it. The advantage of using an API Gateway, and not using a [function URL](https://docs.aws.amazon.com/lambda/latest/dg/urls-invocation.html), is that I can customize the domain name used for my application, instead of relying on the auto-generated one that is provided by lambda.

_[lib/serverless-api-stack.ts](https://github.com/electrototo/aws-multi-region-deployments/blob/master/lib/serverless-api-stack.ts)_

```typescript
import * as cdk from 'aws-cdk-lib';
import { LambdaRestApi } from 'aws-cdk-lib/aws-apigateway';
import { Certificate, CertificateValidation } from 'aws-cdk-lib/aws-certificatemanager';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import { Construct } from 'constructs';

export class ServerlessAPIStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const lambdaFunction = new NodejsFunction(this, 'LambdaFunction', {
      entry: 'lib/lambda_handler.ts',
      handler: 'handler'
    });

    const apiGateway = new LambdaRestApi(this, 'apigw', {
      handler: lambdaFunction
    });

    const globalizedDomainName = 'global.c.liendo.net';
    const acmCertificate = new Certificate(this, `${this.region}-RegionalizedACM`, {
      domainName: globalizedDomainName,
      validation: CertificateValidation.fromDns()
    });

    apiGateway.addDomainName(`${this.region}-DomainName`, {
      domainName: globalizedDomainName,
      certificate: acmCertificate
    });
  }
}
```

Finally, I will be using [GeoLocation routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html) on Route53 to deliver the most appropriate DNS record to the user, based on its region that it is accessing the domain name created for this blog post: [global.c.liendo.net](https://global.c.liendo.net):


{{< figure src="images/route53-geolocation-records.png" width="100%" caption="Route53 GeoLocation records" >}}

Even though I manually created them through the AWS console, you can automize their deployment by using the [CDK constructs for Route53](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_route53.ARecord.html).


### Pipeline
The pipeline will be connected to the [GitHub repo](https://github.com/electrototo/aws-multi-region-deployments) that I created for this blog post, and the idea is to automate the infrastructure deployment everytime I do changes to my main branch (master).

{{< figure src="images/pipeline.png" height="75%" caption="CodePipeline deploying to multipl e regions" >}}

As the code flows through the pipeline, it will through different phases:
1. **Building**: The CDK code will be synthesized to CloudFormation templates, and the application code will be bundled for its deployment.
2. **Packaging**: Using the assets built on the "Build" step, CodePipeline will deploy it to the corresponding buckets; either in the same AWS account and region, on different regions, or different AWS account and different regions.
3. **Deployment**: Finally, CodePipeline will deploy the code using CloudFormation deployments on the corresponding AWS accounts and regions.

The [CDK construct](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.pipelines-readme.html) I will be using for the pipeline has the following notion: _build once, deploy in multiple regions_. 

_[lib/pipeline.ts](https://github.com/electrototo/aws-multi-region-deployments/blob/6afc57ee18faae87cc553743b578f34bf788627c/lib/pipeline.ts)_


The following code block contains the definition of the pipeline, along with the application stacks (`RegionalizedApp`), here we are defining that the `RegionalizedApp` shuld be deployed in three different regions; `sa-east-1`, `us-east-2`, and `us-west-2`.

```typescript
import { Stack, StackProps } from "aws-cdk-lib";
import { StringParameter } from "aws-cdk-lib/aws-ssm";
import { CodePipeline, CodePipelineSource, ShellStep } from "aws-cdk-lib/pipelines";
import { Construct } from "constructs";
import { RegionalizedApp } from "./regionalized-app";

export class Pipeline extends Stack {
    constructor(scope: Construct, id: string, props?: StackProps) {
        super(scope, id, props);

        const pipeline = new CodePipeline(this, 'Pipeline', {
            useChangeSets: false,
            synth: new ShellStep('Synth', {
                input: CodePipelineSource.connection(
                    'electrototo/aws-multi-region-deployments',
                    'master',
                    {
                        connectionArn: `arn:aws:codeconnections:${this.region}:${this.account}:connection/dc9f06dd-ca6e-4bd1-a748-aa579d16e450`
                    }
                ),
                commands: [
                    'npm ci',
                    'npm run build',
                    'npx cdk synth'
                ]
            })
        });

        const southAmericaWave = pipeline.addWave('SouthAmerica');
        southAmericaWave.addStage(
            new RegionalizedApp(this, 'SaoPaolo', {
                env: {
                    account: this.account,
                    region: 'sa-east-1'
                }
            })
        )

        const northAmericaWave = pipeline.addWave('NorthAmerica');

        northAmericaWave.addStage(
            new RegionalizedApp(this, 'Oregon', {
                env: {
                    account: this.account,
                    region: 'us-west-2'
                }
            })
        );

        northAmericaWave.addStage(
            new RegionalizedApp(this, 'Ohio', {
                env: {
                    account: this.account,
                    region: 'us-east-2'
                }
            })
        );
    }
}
```

Finally, to be able to use the pipeline, we should wrap all the stacks into what CodePipeline calls a _stage_. The stage that we are deploying only contains one Stack `ServerlessAPIStack`, which is the one that contains the Lambda code along with API Gatway that will be receving the requests from the customers:

_[lib/regionalized-app.ts](https://github.com/electrototo/aws-multi-region-deployments/blob/6afc57ee18faae87cc553743b578f34bf788627c/lib/regionalized-app.ts)_
```typescript
import { Stage, StageProps } from "aws-cdk-lib";
import { Construct } from "constructs";
import { ServerlessAPIStack } from "./serverless-api-stack";

export class RegionalizedApp extends Stage {
    constructor(scope: Construct, id: string, props?: StageProps) {
        super(scope, id, props);

        new ServerlessAPIStack(this, 'ServerlessAPIStack');
    }
}
```

The pipeline needs to be manually deployed once, and on subsequent executions after receiving a notification from GitHub, it will auto mutate! How amazing is it?

The synthesized pipeline should look like the following picture:

{{< figure src="images/pipeline_console_1.png" height="700px" >}}
{{< figure src="images/pipeline_deploy_2.png" height="630px" >}}
{{< figure src="images/pipeline_deploy_3.png" height="550px" caption="Pipeline in AWS Console" >}}


## Testing

After everything was setup, with the appropriate permissions between AWS and my GitHub repository, and deployed to all regions, I am able to call my region aware endpoint:

```sh
> curl -s https://global.c.liendo.net

{
  "message": "Hello from us-west-2!"
}
```

I am invoking the lambda that is placed in us-west-2, because currently I am living on Washington State.

If I open a CloudShell terminal in AWS, in the South America region (sa-east-1), I will obtain a different response:

```sh
[cloudshell-user ~]$ curl -s https://global.c.liendo.net
{
  "message": "Hello from sa-east-1!"
}
```

Finally, if I access the lambda from anywhere else in the world that is not United States, or any country in South America, I will obtain the response from the default region.

```sh
[cloudshell-user ~]$ curl -s https://global.c.liendo.net
{
  "message": "Hello from us-east-2!"
}
```

And everything is powered by DNS. If I resolve the `global.c.liendo.net` domain name in the three different regions, I will obtain three different A entries:

_us-west-2_
```
; <<>> DiG 9.18.28 <<>> global.c.liendo.net
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5560
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; ANSWER SECTION:
global.c.liendo.net.    60      IN      A       52.37.129.38
global.c.liendo.net.    60      IN      A       35.83.144.35
global.c.liendo.net.    60      IN      A       52.89.136.230
```

_sa-east-1_
```
; <<>> DiG 9.18.28 <<>> global.c.liendo.net
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45082
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; ANSWER SECTION:
global.c.liendo.net.    60      IN      A       52.67.99.123
global.c.liendo.net.    60      IN      A       54.232.156.89
```

_ap-northeast-1_
```
; <<>> DiG 9.18.28 <<>> global.c.liendo.net
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 37007
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; ANSWER SECTION:
global.c.liendo.net.    60      IN      A       3.20.134.37
global.c.liendo.net.    60      IN      A       13.59.146.244
global.c.liendo.net.    60      IN      A       18.116.119.69
```

[^1]: https://www.itu.int/en/ITU-D/Statistics/Pages/stat/default.aspx