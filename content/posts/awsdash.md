+++
date = '2026-09-02T12:12:35+02:00'
title = 'Awsdash: a CLI for CloudWatch dashboards'
draft = true
+++

CloudWatch dashboards are useful once they exist, but building one by clicking through the console is tedious, and the JSON widget format underneath is not something you want to hand-edit. So I wrote [awsdash](https://github.com/go-monk/awsdash), a Go CLI (and library) that discovers your tagged AWS resources and creates or updates a CloudWatch dashboard for them.

Point it at a set of tags and it does the rest:

```sh
$ go install github.com/go-monk/awsdash@latest
$ awsdash -tags environment=production
```

Under the hood it discovers Amplify apps, API Gateway REST APIs, Lambda functions, and S3 buckets matching those tags - concurrently, via `errgroup` - and then hands them to a `dashboard.Put` call built out of small, composable widgets:

```go
if err := dashboard.Put(ctx, cfg, tags,
    widget.Text(dashboard.Header(tags), 24, 2),

    widget.Text("## Amplify Apps", 24, 1),
    widget.Metric(apps.Requests(cfg.Region), 8, 5),
    widget.Metric(apps.Errors4xx(cfg.Region), 8, 5),
    widget.Metric(apps.Errors5xx(cfg.Region), 8, 5),

    widget.Text("## Lambdas", 24, 1),
    widget.Metric(lambdas.Invocations(cfg.Region).LegendRight(), 12, 5),
    widget.Metric(lambdas.Errors(cfg.Region).LegendRight(), 12, 5),

    widget.Text("## S3 Buckets", 24, 1),
    widget.Metric(buckets.Size(cfg.Region).ShowUnits().LegendRight(), 12, 5),
    widget.Metric(buckets.Objects(cfg.Region).ShowUnits().LegendRight(), 12, 5),
); err != nil {
    log.Fatal(err)
}
```

Each resource family knows how to describe its own metrics - `apps.Requests(region)`, `lambdas.Errors(region)`, `buckets.Size(region).ShowUnits()` - so `main.go` reads like a table of contents instead of a pile of CloudWatch widget JSON, and the chainable widget options (`.ShowUnits()`, `.LegendRight()`) keep layout tweaks out of the AWS SDK's raw structs.

Resource discovery itself goes through the [Resource Groups Tagging API](https://docs.aws.amazon.com/resourcegroupstagging/latest/APIReference/overview.html) rather than one API call per service, so adding a new resource family later is a matter of filtering ARNs by type, not writing a new discovery path:

```go
func GetARNs(ctx context.Context, cfg aws.Config, resourceTypeFilters []string, tags map[string]string) ([]string, error)
```

Dashboards are named from a prefix plus a stable hash of the tag set (`dashboard.Name(tags)`), so running the same command twice against the same tags updates the same dashboard instead of creating duplicates - which is really the whole point: `awsdash -tags environment=production` is safe to run from CI on every deploy, and the dashboard just stays current.

It's a small tool doing one thing: turning tagged AWS resources into a dashboard, without touching the console.
