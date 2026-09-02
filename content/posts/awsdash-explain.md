+++
date = '2026-09-02T12:27:47+02:00'
title = 'Awsdash: explaining widgets with a local vision model'
draft = true
+++

[awsdash](https://github.com/go-monk/awsdash) builds CloudWatch dashboards from tagged AWS resources - I wrote about the basic version [here]({{< ref "awsdash" >}}). On the `kronk` branch it grew a second mode: instead of just putting widgets on a dashboard, `-explain` renders one or more of them as images and asks a local vision model to describe what's in the chart, right there in the terminal.

```sh
$ awsdash \
    -tags environment=production \
    -explain lambda.errors \
    -explain s3.size \
    -explain-since 24h \
    -explain-model Qwen3.6-35B-A3B-UD-Q4_K_M
```

It uses the [Kronk](https://github.com/ardanlabs/kronk) Go SDK directly, so there's no separate server or CLI to shell out to. Kronk installs missing runtime libraries and downloads the requested model on first use, and awsdash loads it once and reuses it for every selected widget instead of paying that cost per chart:

```go
type Explainer interface {
	Explain(context.Context, Request) (string, error)
	Close(context.Context) error
}
```

Explain mode is read-only and deliberately narrow. It doesn't create or update a dashboard, and it only discovers the resource families needed by the widgets you actually selected - asking for an Amplify widget doesn't go enumerate Lambda functions or S3 buckets it doesn't need:

```go
func resourcesForSelectors(selectors []string) resourceSelection {
	all := resourceSelection{amplify: true, apigateway: true, lambda: true, s3: true}
	if len(selectors) == 0 {
		return all
	}
	...
}
```

The more interesting constraint is in the prompt itself. A vision model looking at a spiky error graph will happily invent a cause if you let it, so the prompt sent to Kronk spells out evidence boundaries explicitly:

```go
for _, want := range []string{
    "Widget id: lambda.errors",
    "2026-07-14T06:00:00Z to 2026-07-14T12:00:00Z",
    "Do not invent values, events, or root causes",
    "Next checks",
}
```

That keeps the model describing what's visible in the widget and suggesting what to check next, rather than confidently fabricating an incident narrative from a line chart.

`-explain all` runs every stable widget through this in one pass; `-explain-since` controls the metric time range (longer windows like `24h` matter for S3 storage metrics, which CloudWatch only publishes daily); and `-explain-verbose` surfaces Kronk's own model and inference logs when something needs debugging. Run `awsdash -h` for the full list of widget IDs and whatever vision models are already installed locally.

It's the same tool doing one more thing: instead of only shipping metrics to a dashboard you still have to read, it can read a chart back to you.
