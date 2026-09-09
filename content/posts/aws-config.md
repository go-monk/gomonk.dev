+++
date = '2026-09-09T11:23:06+02:00'
title = 'AWS configuration sources'
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

```
$ cd ~/github.com/go-monk/aws-config/list-s3-buckets
$ GOOS=linux CGO_ENABLED=0 go build
$ docker run --rm -it --mount type=bind,source=".",target=/tmp,readonly busybox sh
# /tmp/list-s3-buckets 
list-s3-buckets: operation error S3: ListBuckets, resolve auth scheme: resolve endpoint: endpoint rule error, Invalid region: region was not a valid DNS name.
```

You get this error because region is a required configuration value and it's not set since there's no configuration in the container. The default configuration sources that the SDK's `config.LoadDefaultConfig` function searches are:

* environment variables, like `AWS_REGION`
* shared[^1] configuration and credentials files in `~/.aws/`

You can also optionally pass additional configuration to the function as you can see below where the [config.WithRegion](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/config#WithRegion) function is used.

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
# AWS_REGION=eu-central-1 /tmp/list-config-sources 
SOURCE  TYPE                 REGION
1       config.LoadOptions   us-east-1
2       config.EnvConfig     eu-central-1
3       config.SharedConfig  us-east-2
```

In the table above you can see the three types of configuration sources and the region configuration value. The us-east-1 region is hardcoded inside the program by virtue of the `config.WithRegion` option, eu-central-1 comes from the `AWS_REGION` environment variable and us-east-2 from the default profile in the `~/.aws/config` file:

```
# cat ~/.aws/config 
[default]
region=us-east-2
```

Play around with the programs, change them and change the `config` file and environment variables. It will help you to understand what's going on.

[^1]: They are called shared because they are used by all local AWS CLIs (like `aws s3 ls`) and SDKs (like the programs above).

