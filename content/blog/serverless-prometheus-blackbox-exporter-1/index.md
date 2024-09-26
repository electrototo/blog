+++
title = 'Serverless Prometheus Blackbox Exporter pt 1'
description = 'Porting prometheus blackbox exporter to an AWS lambda using custom lambda environments, and a little bit of source code modification.'
date = 2024-09-25T15:02:32-07:00
draft = false
tags = ["Lambda"]
categories = ["Observability"]
[params]
    [params.author]
        email = 'cristobal@liendo.net'
        name = 'Cristobal Liendo'
+++

Recently I discovered [Prometheus](https://prometheus.io/) which is a monitoring system that collects metrics from different data sources by polling
on certain cadence. Afterwards you can access and query the metrics that were obtained through different interfaces such as [Grafana](https://grafana.com/) and
a web UI.

Being a monitoring tool that is capable to obtain metrics such as latency and availability of services, I started to wonder if I can deploy it in AWS Lambda, mainly for two
reasons:
1. Lambda is relatively cheap.
2. Lambda is avaialble in many, many AWS regions around the world. 

Point number 2 interests me, as I can theoretically monitor a service (for example this blog) from different geographical places and have a basic understanding regarding how is it performing.

## Multi-Target Exporter Pattern
I want to focus on the [Multi-Target Exporter Pattern](https://prometheus.io/docs/guides/multi-target-exporter/) that is common within Prometheus. As high level,
this exporter pattern is used to obtain metrics related to latency and reachability outside from the network that prometheus is running on.

A sample request to a local installation of the [blackbox_exporter](https://github.com/prometheus/blackbox_exporter), which implements the multi-target exporter pattern
looks like the following:

### Request
The request is reaching the blackbox exporter running at the port 9115 of my laptop, and it is requesting to probe [cristobal.liendo.net](https://cristobal.liendo.net) with the
module `http_2xx`.

```bash
curl 'localhost:9115/probe?target=cristobal.liendo.net&module=http_2xx'
```

I will not dive into the configuration of blackbox exporter, but basically modules indicate _how_ should the probe work: https configuration, timeouts, assertions, among others. Depending
on the configuration of the module, the response containing the metrics will vary.


### Response
```
# HELP probe_dns_lookup_time_seconds Returns the time taken for probe dns lookup in seconds
# TYPE probe_dns_lookup_time_seconds gauge
probe_dns_lookup_time_seconds 0.118188376
# HELP probe_duration_seconds Returns how long the probe took to complete in seconds
# TYPE probe_duration_seconds gauge
probe_duration_seconds 0.270411918
# HELP probe_failed_due_to_regex Indicates if probe failed due to regex
# TYPE probe_failed_due_to_regex gauge
probe_failed_due_to_regex 0
# HELP probe_http_content_length Length of http content response
# TYPE probe_http_content_length gauge
probe_http_content_length 2078
# HELP probe_http_duration_seconds Duration of http request by phase, summed over all redirects
# TYPE probe_http_duration_seconds gauge
probe_http_duration_seconds{phase="connect"} 0.005934780000000001
probe_http_duration_seconds{phase="processing"} 0.091509201
probe_http_duration_seconds{phase="resolve"} 0.120317337
probe_http_duration_seconds{phase="tls"} 0.050705509
probe_http_duration_seconds{phase="transfer"} 0.000334677
# HELP probe_http_last_modified_timestamp_seconds Returns the Last-Modified HTTP response header in unixtime
# TYPE probe_http_last_modified_timestamp_seconds gauge
probe_http_last_modified_timestamp_seconds 1.726642887e+09
... omitted for brevity ...
```

Voilà! I have latency metrics for DNS lookup time, total duration to reach the website, and more granular duration metrics.

Keep in mind that these values are perceived from the place where the blackbox exporter server was running, in this case: my laptop.

## Porting blackbox exporter to AWS Lambda
As high-level we need to do two things to port blackbox exporter to Lambda, which basically will merge into a single file:
1. Use a custom AWS Lambda Runtime Environment [^1] in which the blackbox binary will be running.
2. Create a _wrapper_ between lambda and the blackbox process (aka `cURL`'ing everyting you can).

All the code that will be used on this section can be found in my GitHub repository: [aws-lambda-blackbox-exporter](https://github.com/electrototo/aws-lambda-blackbox-exporter).

### Lambda CDK Definition
We will start with a simple lambda that is defined on CDK. For that we have to ensure a couple of things:
1. The runtime must be Amazon Linux 2, and not a managed environment such as node, or python.
2. The architecture of the lambda must be x86_64, as the binary we will be downloading will be compiled for that architecture.
3. We need to find a way (by using CDK's `Code.fromCustomCommad()` method) to bootstrap the environment at synth with the appropriate binaries and lambda entrypoint.

_[lib/aws-lambda-blackbox-exporter-stack.ts](https://github.com/electrototo/aws-lambda-blackbox-exporter/blob/master/lib/aws-lambda-blackbox-exporter-stack.ts)_
```ts
import * as cdk from 'aws-cdk-lib';
import { CfnOutput } from 'aws-cdk-lib';
import * as aws_lambda from 'aws-cdk-lib/aws-lambda';

import { Construct } from 'constructs';

export class AwsLambdaBlackboxExporterStack extends cdk.Stack {
    constructor(scope: Construct, id: string, props?: cdk.StackProps) {
        super(scope, id, props);

        // We need to ensure that the binary is compatible with x86_64 architectures, as the lambda is provisioned for
        // x86_64
        const blackboxDownloadURL = 'https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz';

        // These are the commands that will be used to create the bundle, they will be executed at synth time
        const code = aws_lambda.Code.fromCustomCommand('bundle/', ['./bundle.sh'], {
            commandOptions: {
                'env': {
                    'BLACKBOX_DOWNLOAD_URL': blackboxDownloadURL
                }
            }
        });

        // This is the lambda where the blackbox exporter will be running on
        const lambdaFunction = new aws_lambda.Function(this, 'BlackboxExporterLambda', {
            code: code,
            runtime: aws_lambda.Runtime.PROVIDED_AL2,
            architecture: aws_lambda.Architecture.X86_64,
            handler: 'doesnt-really-matter',
            timeout: cdk.Duration.seconds(10)
        });

        // Create the function URL, so it can be invoked using an HTTP request
        const functionURL = lambdaFunction.addFunctionUrl({
            authType: aws_lambda.FunctionUrlAuthType.NONE
        });

        new CfnOutput(this, 'LambdaEndpoint', {
            value: functionURL.url,
            description: 'Endpoint to invoke the lambda from'
        });
    }
}
```

### Custom Lambda Code Bundling
_[bundle.sh](https://github.com/electrototo/aws-lambda-blackbox-exporter/blob/master/bundle.sh)_
```sh
#!/bin/bash

# If the bundle directory exists, then just copy the bootsrap file
# this is to avoid downloading multiple times the blackbox exporter
if test -d bundle; then
    cp lambda/bootstrap bundle/
    exit 0
fi

# Otherwise, create the dir and download the assets
mkdir bundle/

# Download the exporter, and copy the required files
wget "$BLACKBOX_DOWNLOAD_URL" -O bundle/exporter.tar.gz
cp lambda/bootstrap bundle/

# Unzip the binary and delete original zip
cd bundle/
tar -xzvf exporter.tar.gz --strip-components 1
rm exporter.tar.gz
```

This bash script is executed when the `cdk synth` command is executed. It does a couple of things:
1. Downloads the blacbox exporter binary which is given through the `BLACKBOX_DOWNLOAD_URL` environment variable on the CDK file,
2. Uncompress its contents to the `bundle/` directory,
3. Copies the `bootstrap` file from the `lambda/` directory to the `bundle/` directory, and
4. Does some post cleanup.

### Blackbox exporter wrapper
_[lambda/bootstrap](https://github.com/electrototo/aws-lambda-blackbox-exporter/blob/master/lambda/bootstrap)_
```sh
#!/bin/bash

# Run the blackbox exporter
./blackbox_exporter &

# Wait for 100ms for the exporter to initialize
sleep 0.1

while true
do
    HEADERS="$(mktemp)"

    # 1. Retrieve the information of the invocation, including headers and the payload of the event.
    #    The headers will be stored in the file created when running the mktemp command.
    EVENT_DATA=$(curl -sS -LD "$HEADERS" "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/next")

    # 2. Parse the event, retrieving the url to be probed
    RAW_PATH=$(echo "$EVENT_DATA" | grep -o '"rawPath":"[^"]*' | grep -o '[^"]*$')
    QUERY_PARAMS=$(echo "$EVENT_DATA" | grep -o '"rawQueryString":"[^"]*' | grep -o '[^"]*$')

    # 3. Retrieve the response from blackbox exporter
    RESPONSE=$(curl "localhost:9115$RAW_PATH?$QUERY_PARAMS")

    # 4. From the file, get the request ID, so we are able to successfully comunicate the response
    REQUEST_ID=$(grep -Fi Lambda-Runtime-Aws-Request-Id "$HEADERS" | tr -d '[:space:]' | cut -d: -f2)

    # 5. Send the response from the probing back to the caller
    curl "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/$REQUEST_ID/response" -d "$RESPONSE"
done
```

This bootstrap file is the one that is being executed by lambda everytime there's a new request made through the endpoint. As a high level, for any kind of environment
running on AWS Lambda, being either managed ones or custom ones, the following needs to happen:
1. The event is requested from the runtime API.
2. The underlying code is executed, and the response is obtained.
3. Using the runtime API provided by lambda, the response is sent back to the caller through the response endpoint.

In this implementation the blackbox exporter is forked, and the main process sleeps for 100ms to wait for the blackbox exporter to start. After that the main loop is executed
obtaining the event from the runtime api, redirecting it to the blackbox exporter, and returning back whatever was the output of the cURL to the blakbox exporter.

### Deployment
After the [aws-lambda-blackbox-exporter](https://github.com/electrototo/aws-lambda-blackbox-exporter) github CDK repo is built, and deployed by executing the following commands:
```sh
npm run build
cdk synth
cdk deploy
```

We obtain the endpoint that we can invoke the blackbox exporter as a stack output:
```sh
✨  Synthesis time: 4.91s

AwsLambdaBlackboxExporterStack: deploying... [1/1]

 ✅  AwsLambdaBlackboxExporterStack (no changes)

✨  Deployment time: 0.33s

Outputs:
AwsLambdaBlackboxExporterStack.LambdaEndpoint = https://abcdefghijlkmnopqrstuvwxyz.lambda-url.us-west-2.on.aws/
Stack ARN:
arn:aws:cloudformation:us-west-2:1122334455:stack/AwsLambdaBlackboxExporterStack/aaaa-bbbb-ccccc-dddd

✨  Total time: 5.24s
```

## Results
Using the endpoint that was obtained after the deployment, we can make a simple HTTP request using cURL to request the latency metrics for any endpoint; for example my blog:
```
$ curl 'https://abcdefghijlkmnopqrstuvwxyz.lambda-url.us-west-2.on.aws/probe?target=cristobal.liendo.net&module=http_2xx'

# HELP probe_dns_lookup_time_seconds Returns the time taken for probe dns lookup in seconds
# TYPE probe_dns_lookup_time_seconds gauge
probe_dns_lookup_time_seconds 0.037133299
# HELP probe_duration_seconds Returns how long the probe took to complete in seconds
# TYPE probe_duration_seconds gauge
probe_duration_seconds 0.14913135
# HELP probe_failed_due_to_regex Indicates if probe failed due to regex
# TYPE probe_failed_due_to_regex gauge
probe_failed_due_to_regex 0
# HELP probe_http_content_length Length of http content response
# TYPE probe_http_content_length gauge
probe_http_content_length -1
# HELP probe_http_duration_seconds Duration of http request by phase, summed over all redirects
# TYPE probe_http_duration_seconds gauge
probe_http_duration_seconds{phase="connect"} 0.026434002999999998
probe_http_duration_seconds{phase="processing"} 0.07593583100000001
probe_http_duration_seconds{phase="resolve"} 0.039267746
probe_http_duration_seconds{phase="tls"} 0
probe_http_duration_seconds{phase="transfer"} 0.006892414
# HELP probe_http_redirects The number of redirects
# TYPE probe_http_redirects gauge
probe_http_redirects 1
# HELP probe_http_ssl Indicates if SSL was used for the final redirect
# TYPE probe_http_ssl gauge
probe_http_ssl 0
# HELP probe_http_status_code Response HTTP status code
# TYPE probe_http_status_code gauge
probe_http_status_code 200
# HELP probe_http_uncompressed_body_length Length of uncompressed response body
# TYPE probe_http_uncompressed_body_length gauge
probe_http_uncompressed_body_length 22531
# HELP probe_http_version Returns the version of HTTP of the probe response
# TYPE probe_http_version gauge
probe_http_version 1.1
# HELP probe_ip_addr_hash Specifies the hash of IP address. It's useful to detect if the IP address changes.
# TYPE probe_ip_addr_hash gauge
probe_ip_addr_hash 3.831431737e+09
# HELP probe_ip_protocol Specifies whether probe ip protocol is IP4 or IP6
# TYPE probe_ip_protocol gauge
probe_ip_protocol 4
# HELP probe_success Displays whether or not the probe was a success
# TYPE probe_success gauge
probe_success 1
```

### Prometheus and Grafana integration
After hooking the blackbox expoter with prometheus and grafana, and scraping the results from the blackbox exporter for a while, I can obtain metrics about the latency of my website, as if users located in the North West of United States were accessing my blog[^2]. 

{{< figure src="images/grafana.png" width="100%" caption="Visualization on Grafana" >}}

## Closing thoughts
The initial results look very promising. Even though the blackbox exporter is currently deployed in a single AWS region (us-west-2), we can deploy it in any available AWS region that
your AWS account has access to; in fact, I will be talking about this in another blog post.

### Optimizations
One thing that would be interesting to explore is the [initialization]({{< ref "#blackbox-exporter-wrapper" >}} "aa") of the blackbox exporter process. Right now we are waiting for 100ms, assuming
that the process will initialize in time. We could pipe the output of the process somewhere, and wait until we read from the output that the server started successfully.

### Pricing
Doing some estimates about how much I would be paying, if I constantly probe the blackbox exporter lambda, I ended up with the following results using the [lambda pricing](https://aws.amazon.com/lambda/pricing/) for us-west-2:
* Scraping 1 time every 15 seconds, equals to 4 invocations per minute * 60 minutes * 24 hours * 31 days = 178,560 invocations per month.
* Assuming an average execution duration of 300ms * 178,560 invocations = 53,568 seconds.
* The lambda has 128MB of memory configured = 0.125GB
* GB-second per month = 0.125GB * 53,568 seconds = 6,696 GB-seconds

Price just because we are invoking the lambda: ($0.20 / 1,000,000 invocations) * 178,560 invocations per month = **$0.035712 USD**

Price due to memory configuration: 6,696 GB-seconds * $0.0000166667 for every GB-second = **$0.111555 USD**

Grand total: **$0.1472 per month**

Even if I were to host this on a Raspberry PI, I would need to make a request every 15 seconds for 15 years to justify moving away from lambda! Isn't that amazing?


[^1]: If you want to learn more about custom AWS Lambda Runtime Environments, feel free to read my [previous blog post]({{< ref "custom-aws-lambda-runtime-environments" >}})!
[^2]: This is not 100% accurate, as I am hosting both the blackbox exporter and my blog within AWS. The request from the blackbox to my blog would not need to leave the AWS network,
which would make it more performant. This could be easily tested if we deploy something similar to Azure, GCP, or a raspberry pi!