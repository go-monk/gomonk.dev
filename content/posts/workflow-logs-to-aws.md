+++
date = '2026-09-02T12:05:33+02:00'
title = 'Push GitHub Actions logs to CloudWatch'
draft = true
+++

GitHub keeps workflow run logs for 90 days by default, then deletes them. If you need to keep them longer - for audits, incident postmortems, or just peace of mind - you have to ship them somewhere else yourself. So I wrote [workflow-logs-to-aws](https://github.com/go-monk/workflow-logs-to-aws), a small Go CLI and GitHub Action that pushes job logs from a workflow run into AWS CloudWatch Logs.

It's a plain Go program under the hood: no framework, just the [go-github](https://github.com/google/go-github) and [AWS SDK v2](https://github.com/aws/aws-sdk-go-v2) clients wired together.

```go
runAttempts, err := ghc.WorkflowRunAttempts(ctx, runID)
...
for _, runAttempt := range runAttempts {
    jobs, err := ghc.WorkflowJobs(ctx, runID, runAttempt)
    ...
    for _, job := range jobs {
        logs, err := ghc.DownloadJobLogs(ctx, *job.ID)
        ...
        err = cw.UploadJobLog(ctx, logs, *job.WorkflowName, runID, runAttempt, *job.ID, *job.Name, logGroup, retentionDays, replace)
    }
}
```

For each completed job it downloads the log, ensures a CloudWatch log group and stream exist, and writes the log lines as CloudWatch events, parsed from GitHub's own timestamped log format. Run it as a standalone binary:

```sh
$ go install github.com/go-monk/workflow-logs-to-aws@latest
$ workflow-logs-to-aws -repository owner/repo 123456789
```

or wire it into CI as an action, triggered after another workflow finishes:

```yaml
on:
  workflow_run:
    workflows: [Some workflow]
    types: [completed]

permissions:
  id-token: write
  actions: read

jobs:
  push-logs:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubWorkflowLogsWriter
          aws-region: eu-central-1
      - uses: go-monk/workflow-logs-to-aws@v0
        with:
          retention-days: 30
```

Credentials come from whatever AWS action ran before it (no secrets baked in), and the action itself runs from a prebuilt image on GHCR, so there's no build step on every CI run.

I also added an `-emf` flag that, alongside the raw logs, emits [CloudWatch EMF](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format.html) events per job - job count, failures, and duration - so you get workflow metrics and dashboards for free, without a separate metrics pipeline.

It's a small tool doing one thing: taking logs that would otherwise vanish after 90 days and putting them somewhere durable and queryable, using infrastructure (CloudWatch, OIDC roles) you probably already have.
