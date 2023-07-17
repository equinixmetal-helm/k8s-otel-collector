# Development

This document describes the process for modifying and testing this Helm chart locally, as well as instructions on how to perform a canary deploy with a service's chart to test changes on this chart.

## Requirements

```sh
brew install helm
```

## Local testing

### Rendering the template

To check how Helm will attempt to render the template:

```sh
# from repository root
helm template . --output-dir test --debug
```

Note that Helm will overwrite the contents of your output directory with each run.

### Using a service chart with local k8s-otel-collector

(In this context, the service's chart is called the [*parent chart*](https://helm.sh/docs/chart_template_guide/subcharts_and_globals/)).
Mentioning here just because there's some Helm behavior that's specific to parent charts vs. subcharts.)

In k8s-otel-collector, update `Chart.yaml` to a test version:

```diff
  apiVersion: v2
  name: k8s-otel-collector
  description: OpenTelemetry Collector Helm Chart
  type: application
- version: 0.6.1
+ version: 0.7.0-test
```

In the service chart's `Chart.yaml`, update the k8s-otel-collector repository and version:

```diff
  dependencies:
    . . .
    - name: k8s-otel-collector
-     repository: helm.equinixmetal.com
+     repository: file://../k8s-otel-collector # replace with the path to your local repo
-     version: 0.6.1
+     version: 0.7.0-test
```

Run `helm dependency update` to have Helm pull your local version of the k8s-otel-collector (it might also update other dependencies):

```console
~/repos/k8s-my-service $ helm dependency update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 2 charts
Downloading common from repo https://charts.bitnami.com/bitnami
Deleting outdated charts
~/repos/k8s-my-service $ git status
On branch otel-test
Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   Chart.lock
        modified:   Chart.yaml
        deleted:    charts/common-2.4.0.tgz
        deleted:    charts/k8s-otel-collector-0.6.1.tgz

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        charts/common-2.6.0.tgz
        charts/k8s-otel-collector-0.7.0-test.tgz

no changes added to commit (use "git add" and/or "git commit -a")
~/repos/k8s-my-service $
```

### Render the service chart locally

You can test the compiled output with the same `helm template` command as above:

```console
~/repos/k8s-my-service $ helm template . --output-dir test --debug
```

The files for the k8s-otel-collector subchart will be rendered in the `charts/` subdirectory under your `test/` directory.

```
test
└── k8s-my-service
    ├── charts
    │   └── k8s-otel-collector
    │       └── templates
    │           ├── deployment.yaml
    │           ├── opentelemetry-collector-config.yaml
    │           ├── service-monitor.yaml
    │           └── service.yaml
    └── templates
        └── app
            ├── config.yaml
            ├── deployment.yaml
            ├── service.yaml
            └── etc.
```

**Be sure not to commit the rendered output directory to the service's chart repo!**
The `test/` directory is ignored by git in this repo but it may not be ignored in other repos.

### Note: `.Values.clusterInfo` won't render locally

This chart allows the application charts using it to create a `_otel-attributes.tpl` partial template to add custom attributes to spans and metrics with Kubernetes cluster metadata, which has proved useful in Metal's multi-cluster environment.
See the [README](./README.md/#create-templates_otel-attributestpl-partial-template-in-application-chart) for instructions on how to set up the cluster info OTel attributes within an application's Helm chart.

The `.Values.clusterInfo` values come from Atlas, so rendering the application chart's templates locally will not fill in those values correctly.
In order to test that the chart is working correctly, you'll need to canary a test branch in a Metal cluster.

## Canarying on a service

### Use local version of k8s-otel-collector

Follow the instructions in the above section updating the service chart's `Chart.yaml` to point to the k8s-otel-collector repository and version.

Make any other relevant changes in the service's chart, and then commit your changes to a new branch.
Push your branch to the service chart's upstream repo.

### Perform a branch deploy on the service

Before deploying any changes, check with the team whose service chart you're testing on.
If it's an edge service, the team may be able to suggest a site that's a safe place for canarying.

Log in to Argo prod and filter to the service's namespace and the desired cluster and select it to pull up the tree view.
At the top of the page, select **App Details**.

In the App Details view, **Target Revision** field should say `HEAD`.
Select the **Edit** button, and then update Target Revision to use your test branch.
It should auto-suggest the branch name once you start typing.
Save your changes and then select **Sync**.
Validate your changes in the ConfigMap and any other resources that come up, and check Honeycomb to be sure that data is still flowing.

To revert, switch back to `HEAD` and sync again.
