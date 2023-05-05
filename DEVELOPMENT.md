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

### Packaging the chart locally

Use `helm package` with optional flags to create a `.tgz` of this chart, which can be copied into the `charts/` directory of a service chart using k8s-otel-collector as a subchart:

```console
# from repository root
$ helm package .
Successfully packaged chart and saved it to: /Users/spees/repos/k8s-otel-collector/k8s-otel-collector-0.5.0.tgz
```

### Rendering a service chart with k8s-otel-collector

Once you've packaged k8s-otel-collector, copy it into the local chart repo for your desired service chart:

```console
~/repos/k8s-otel-collector $ cp k8s-otel-collector-0.5.0.tgz ../k8s-site-narwhal/charts
```

From within the other service's chart you can test the compiled output with the same `helm template` command as above:

```console
~/repos/k8s-site-narwhal $ helm template . --output-dir test --debug
```

The files for k8s-otel-collector (considered a subchart in this case) will end up in the `charts/` subdirectory under your `test/` directory.

**Be sure not to commit the rendered output directory to the service's chart repo!**
Both the packaged `.tgz` files and the `test/` directory are ignored by git in this repo but they may not be ignored in other repos.
You'll need to commit the `.tgz` package for canary testing (see next section) but be sure to remove it before merging to main in the service chart's repo.

### Note: `.Values.clusterInfo` won't render locally

This chart allows the application charts using it to create a `_otel-attributes.tpl` partial template to add custom attributes to spans and metrics with Kubernetes cluster metadata, which has proved useful in Metal's multi-cluster environment.
See the [README](./README.md/#create-templates_otel-attributestpl-partial-template-in-application-chart) for instructions on how to set up the cluster info OTel attributes within an application's Helm chart.

The `.Values.clusterInfo` values come from [Atlas](https://github.com/equinixmetal/delivery-infrastructure), so rendering the application chart's templates locally will not fill in those values correctly.
In order to test that the chart is working correctly, you'll need to canary a test branch in a Metal cluster.

## Canarying on a service

### Commit to a test branch

Package k8s-otel-collector and copy package into service chart repo `charts/` directory:

```console
~/repos/k8s-otel-collector $ helm package .
Successfully packaged chart and saved it to: /Users/spees/repos/k8s-otel-collector/k8s-otel-collector-0.5.1-test.tgz
~/repos/k8s-otel-collector $ cp k8s-otel-collector-0.5.1-test.tgz ../k8s-site-narwhal/charts
```

Update the version in `Chart.yaml`:

```diff
  dependencies:
    . . .
    - name: k8s-otel-collector
      repository: https://helm.equinixmetal.com
-     version: 0.5.0
+     version: 0.5.1-test
```

Make any other relevant changes in the service's chart (in this context, called the [*parent chart*](https://helm.sh/docs/chart_template_guide/subcharts_and_globals/)), and then commit your changes to a new branch.
Push your branch to the service chart's upstream repo.

### Canary deploy your test branch

Before deploying any changes, check with the team whose service chart you're testing on.
If it's an edge service, the team may be able to suggest a site that's a safe place for canarying.

Log in to [Argo prod](https://argo.delivery.metalkube.net/) (requires being on the Metal VPN, plus Okta SSO).
Filter to the service's namespace and the desired cluster and select it to pull up the tree view.
At the top of the page, select **App Details**.

In the App Details view, **Target Revision** field should say `HEAD`.
Select the **Edit** button, and then update Target Revision to use your test branch.
It should auto-suggest the branch name once you start typing.
Save your changes and then select **Sync**.
Validate your changes in the ConfigMap and any other resources that come up, and check Honeycomb to be sure that data is still flowing.

To revert, switch back to `HEAD` and sync again.