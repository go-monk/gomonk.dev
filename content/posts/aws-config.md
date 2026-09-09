+++
date = '2026-09-09T11:23:06+02:00'
title = 'AWS config'
+++

When talking to AWS from a Go program the easiest way is to use the [AWS SDK](https://github.com/aws/aws-sdk-go-v2). Say you want to list the S3 buckets. The program would look something like:

```go
package main

import (
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/s3"
)

func main() {
	// ...
	cfg, err := config.LoadDefaultConfig(ctx)
	// ...
	client := s3.NewFromConfig(cfg)
	out, err := client.ListBuckets(ctx, &s3.ListBucketsInput{})
	// ...
}
```

You can find the full program [here](https://github.com/go-monk/aws-config/tree/main/list-s3-buckets).

Now, let's build and run the program inside a container that has never been configured to connect to AWS:

```sh
$ cd ~/github.com/go-monk/aws-config/list-s3-buckets
$ GOOS=linux CGO_ENABLED=0 go build
$ docker run --rm -it --mount type=bind,source=".",target=/tmp,readonly busybox sh
/ # /tmp/list-s3-buckets 
list-s3-buckets: operation error S3: ListBuckets, resolve auth scheme: resolve endpoint: endpoint rule error, Invalid region: region was not a valid DNS name.
```

You get this error because region is required and it's not set. So how do we set it? One way is to use an environment variable:

```
/ # AWS_REGION="us-east-1" /tmp/list-s3-buckets
list-s3-buckets: operation error S3: ListBuckets, exceeded maximum number of attempts, 3, get identity: get credentials: failed to refresh cached credentials, no EC2 IMDS role found, operation error ec2imds: GetMetadata, exceeded maximum number of attempts, 3, request send failed, Get "http://169.254.169.254/latest/meta-data/iam/security-credentials/": dial tcp 169.254.169.254:80: connect: network is unreachable
```

Now we have a different error that basically means: "tried to find credentials but failed"[^1]. The default configuration sources that the `config.LoadDefaultConfig` function searches are:

* environment variables
* shared[^2] configuration and credentials files

If you have the environment variables[^3] and/or the shared configuration and credentials files in `~/.aws/` set up correctly the SDK's `config.LoadDefaultConfig` function will discover and use them. You can also optionally pass additional configuration to the function as shown below.

Let's look in more detail at all three configuration sources:

```go
	// ...
	cfg, err := config.LoadDefaultConfig(
		context.Background(), config.WithRegion("us-east-1"))
	// ...
	for i, source := range cfg.ConfigSources {
		var region string
		switch source := source.(type) {
		case config.LoadOptions:
			region = source.Region
		case config.EnvConfig:
			region = source.Region
		case config.SharedConfig:
			region = source.Region
		}

		fmt.Fprintf(writer, "%d\t%T\t%s\n", i+1, source, region)
	}
	// ...
```

You can find the full program [here](https://github.com/go-monk/aws-config/tree/main/list-config-sources). Let's run it:

```sh
/ # AWS_REGION=eu-central-1 /tmp/list-config-sources 
SOURCE  TYPE                 REGION
1       config.LoadOptions   us-east-1
2       config.EnvConfig     eu-central-1
3       config.SharedConfig  us-east-2
```

In the table above you can see the three types of configuration sources and the region configuration value. The us-east-1 region is hardcoded inside the program by virtue of the `config.WithRegion`, eu-central-1 comes from the `AWS_REGION` environment variable and us-east-2 from the default profile in the shared config file:

```sh
/ # cat ~/.aws/config 
[default]
region=us-east-2
```

Play around with the programs, change them and change the `config` file and environment variables. It will help you to understand what's going on.

[^1]: The IP address 169.254.169.254 is the AWS EC2 Instance Metadata Service (IMDS) address. It is a link-local address used from an EC2 instance to retrieve instance metadata and IAM role credentials. The AWS SDK could not find credentials from its normal sources, so it tried to retrieve IAM role credentials from IMDS and timed out. You can disable IMDS probing with `AWS_EC2_METADATA_DISABLED=true`.

[^2]: They are called shared because they are used by all local AWS CLIs (like `aws s3 ls`) and SDKs (like the programs above).

[^3]: In AWS SDK for Go v2, config.LoadDefaultConfig(...) reads a fairly broad set of AWS environment variables. The most important ones are: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN, AWS_REGION, AWS_PROFILE.
