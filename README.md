# Equinix Helm chart for the OpenTelemetry Collector

A minimal Helm chart for deploying and configuring the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/).
While an excellent [community Helm chart](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-collector) exists, this chart is set up to define consistent defaults with very limited configuration required of downstream users.

This chart does the following:

- creates a Kubernetes [deployment](templates/deployment.yaml) of pods that use the specified version of the
  [otel/opentelemetry-collector-contrib Docker image](https://hub.docker.com/r/otel/opentelemetry-collector-contrib) from Docker Hub
- mounts a volume for the [Collector configuration file](templates/opentelemetry-collector-config.yaml), which is generated from a template that pulls cluster metadata from Atlas
- creates a Kubernetes [service](templates/service.yaml) for the Collector, exposing relevant ports
- pulls the Honeycomb API key stored in keymaker

This configuration is specific to the strategy we're using at Metal:
**one Collector deployment per application namespace** (e.g. API, cacher, narhwal, boots, etc.).

## Maintainers

This chart is maintained by the [[Governor] Metal OpenTelemetry GitHub team](https://github.com/orgs/equinixmetal-helm/teams/governor-metal-opentelemetry).
If you would like to be a maintainer, request to join the Metal OpenTelemetry group via Governor.

## Deploy the OpenTelemetry Collector for your service

### Add k8s-otel-collector as a dependency to the app Helm chart

Add this Helm chart as a dependency:

```yaml
# Chart.yaml
dependencies:
  - name: k8s-otel-collector
    repository: https://helm.equinixmetal.com
    version: x.x.x  # replace with desired version
```

If desired, override the available values:

```yaml
# values.yaml
k8s-otel-collector:
  image:
    tag: "0.x.x" # override with your desired otel/opentelemetry-collector-contrib image version:
                 # https://hub.docker.com/r/otel/opentelemetry-collector-contrib/tags
  memory_limiter:
    limit_mib: "400"
    spike_limit_mib: "100"
    check_interval: "5s"

  servicemonitor:
    enabled: false
```

### Set the OTLP endpoint via environment variable

In addition to deploying the Collector itself to your application namespace,
you will need to update your app configuration to enable sending telemetry data to the Collector.

When you add the OpenTelemetry SDK to your application, it will look for the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable to determine where to send data.
In this case the OTLP endpoint is the static url of the Collector deployed to that application's namespace.

For apps sending OTLP over HTTP (legacy Ruby services), use the HTTP endpoint:

```shell
export OTEL_EXPORTER_OTLP_ENDPOINT="http://opentelemetry-collector:55681"
export OTEL_RUBY_EXPORTER_OTLP_SSL_VERIFY_NONE=true
```

For apps sending OTLP over gRPC (most services), use the gRPC endpoint:

```shell
export OTEL_EXPORTER_OTLP_ENDPOINT="opentelemetry-collector:4317"
export OTEL_EXPORTER_OTLP_INSECURE=true
```

Depending on the app's Helm chart configuration, the environment variable may need to be set in different ways.
Most k8s-site-{appname} charts will set environment variables in `values.yaml` like so:

```yaml
{appname}:
  env:
    . . .
    OTEL_EXPORTER_OTLP_ENDPOINT: "opentelemetry-collector:4317"
    OTEL_EXPORTER_OTLP_INSECURE: "true"
```

If you're not sure where to add the environment variable, ask Applied Resilience Engineering (`#sre`) or the Delivery team (`#em-delivery-eng`) for help.

### Add the ExternalSecretPull for the Honeycomb API key

Equinix Metal uses a global key for all Metal services for each environment.
Create an ExternalSecretPull to reference the global key.

```yaml
# external-secret-honeycomb-key.yaml
---
apiVersion: keymaker.equinixmetal.com/v1
kind: ExternalSecretPull
metadata:
  name: "honeycomb-key"
  labels:
    component: "opentelemetry-collector"
spec:
  backend: ssm
  mappings:
    - key: /{{ .Values.clusterInfo.environment }}/honeycomb-key/v1
      name: honeycomb-key
```

### Create `templates/_otel-attributes.tpl` partial template in application chart

No indentation needed, the subchart will fix that.
It's recommended to include the comment at the top so that future maintainers don't add too many custom attributes here.

```yaml
# templates/_otel-attributes.tpl
# Attributes included here will be added to every span and every metric that
# passes through the OpenTelemetry Collector in this service's namespace.
# Most custom instrumentation should be done within the service's code.
# This file is a good place to include k8s cluster info that's hard to access elsewhere.
{{- define "otel_attributes" }}
- key: k8s.cluster.endpoint
  value: {{ .Values.clusterInfo.apiEndpoint }}
  action: insert
- key: k8s.cluster.class
  value: {{ .Values.clusterInfo.class }}
  action: insert
- key: k8s.cluster.fqdn
  value: {{ .Values.clusterInfo.fqdn }}
  action: insert
- key: k8s.cluster.name
  value: {{ .Values.clusterInfo.name }}
  action: insert
- key: metal.facility
  value: {{ .Values.clusterInfo.facility }}
  action: insert
{{- end }}
```

Then update the chart's `values.yaml` so that k8s-otel-collector knows to use the template:

```yaml
k8s-otel-collector:
  include_otel_attributes: true
```

#### Ensure that the subchart can read clusterInfo via Atlas

Currently, clusterInfo is not available globally in Atlas.
In order to make the data available to subcharts, you will need to add it to the values:

```diff
    apps:
      - name: appname
        repoURL: "git@github.com:equinixmetal/k8s-central-appname.git"
        clusterValuesFile: true
+       values: |
+         k8s-otel-collector:
+           clusterInfo: {{ clusterInfo .Cluster .App | toYaml | nindent 10 }}
```

Additionally, your chart's GitHub checks might fail because of a bad pointer reference, since the data is not available locally.
To fix this, add an empty clusterInfo hashmap to your values.yaml file:

```yaml
clusterInfo: {}
```

Atlas will overwrite the empty hashmap with valid data when the chart is deployed.

### GitHub checks: make sure github-action-helm3 pulls down dependencies

If you're using [github-action-helm3](https://github.com/WyriHaximus/github-action-helm3) in your Helm linting check, be sure that you add the `--dependency-update` flag so it'll pull down remote dependencies.
Here's an example:

```diff
./github/workflows/helm-lint.yaml
  name: Yaml Lint
  on: [pull_request]
  jobs:
    lintAllTheThings:
      runs-on: ubuntu-latest
      steps:
      - uses: actions/checkout@v3
      - name: Prep helm
        uses: WyriHaximus/github-action-helm3@v3
        with:
-         exec: helm template --output-dir test-output/ .
+         exec: helm template --dependency-update --output-dir test-output/ .
      - name: yaml-lint
        uses: ibiqlik/action-yamllint@v3
        with:
          file_or_dir: test-output
          config_file: .yamllint.yml
```

### Sync in Argo

For initial deployment and any changes to the OTLP endpoint, the app's pods will need to be restarted in order to pick up the new/updated environment variables.
For some configurations, Argo will restart the pods automatically.
For others, you may need to manually restart the pods.
Reach out to Applied Resilience Engineering (`#sre`) or the Delivery team (`#em-delivery-eng`) if you need help with that.

### Add OpenTelemetry instrumentation to the application code

Go apps should use [equinix-labs/otel-init-go](https://github.com/equinix-labs/otel-init-go).
Follow the configuration instructions in the README.

For Ruby apps, follow the instructions in [Confluence](https://packet.atlassian.net/l/c/XBP11Ef4).

## Manage Honeycomb API keys

As of August 2022, Metal services share a global Honeycomb key for each environment.
Metal service teams no longer need to worry about managing Honeycomb keys for their services.
The Applied Resilience Engineering team manages the Honeycomb API keys.
Reach out in the `#sre` channel in Slack if you have questions.

### Rotate a Honeycomb key

(Note: this step requires that you [set up your local Kubernetes config according to the Delivery Docs](https://delivery-docs.metalkube.net/training_and_guides/kubectl/#import-kube-configs).)

The API key name in Honeycomb should use the format `metal-{appname}`.

You will need to create a yaml manifest file to update the ExternalSecretPush.
(For more information about using Keymaker, see [these instructions on the delivery docs site](https://delivery-docs.metalkube.net/core_services/keymaker/?h=keymaker#add-secret-to-secret-store).)

This file must NOT be committed to git so you can just create it in your home directory, for example:

```shell
vim ~/honeycomb-secret-push.yaml
```

Paste the following contents, being sure to use the correct value for the new Honeycomb API key:

```diff
    apiVersion: keymaker.equinixmetal.com/v1
    kind: ExternalSecretPush
    metadata:
      name: honeycomb-key
      annotations:
          clusterlevelsecret: "true"
    spec:
      backend: ssm
      environment: prod
      secrets:
        - key:   honeycomb-key
-         value: OLD_KEY
-         version: v1
+         value: NEW_KEY
+         version: v2
```

(Note that the above diff is for demonstration purposes only, since none of these files should be committed to version control.)

#### Perform the ExternalSecretPush

Save the file and run `kubectl apply` to tell Kubernetes to perform the ExternalSecretPush to create/update the key:

```shell
kubectl apply -f ~/honeycomb-secret-push.yaml
```

You can then use `kubectl get events` to confirm that it was saved successfully.
Here's the full output:

```shell
% kubectl apply -f honeycomb-key.yaml

externalsecretpush.keymaker.equinixmetal.com/honeycomb-key created
% kubectl get events
LAST SEEN   TYPE      REASON           OBJECT                             MESSAGE
80s         Normal    Backend          externalsecretpush/honeycomb-key   backend loaded: ssm
81s         Normal    Secret           externalsecretpush/honeycomb-key   secret saved to ssm: /prod/honeycomb-key/v1
```

When the secret is successfully added, delete the manifest:

```shell
rm ~/honeycomb-secret-push.yaml
```

The final key path will look like `/prod/honeycomb-secret/v2` (or whatever version you've updated it to).
This will automatically get picked up by the ExternalSecretPull generated by the template in this chart.

If you run into issues trying to push a secret, reach out to SRE (#sre) or the Delivery team (`#em-delivery-eng`) for help.

## Troubleshooting

[Follow the instructions in these docs](https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/troubleshooting.md)
to set the Collector's own logs to `DEBUG`.

## Releasing a new version

This repo uses [GitHub Actions](https://github.com/equinixmetal-helm/k8s-otel-collector/actions/workflows/release.yaml) to package the Helm chart and publish it at [`helm.equinixmetal.com`](https://github.com/equinixmetal-helm/charts/tree/gh-pages).

To create a new release, first update Chart.yaml:

```diff
-  version: 0.4.1
+  version: 0.5.0
```

Commit your change and get your PR merged.
Once it's merged create a tag with the same version and push it upstream.

Once you push the new tag, GitHub Actions will automatically create a [release](https://github.com/equinixmetal-helm/k8s-otel-collector/releases) with a changelog, package the Helm chart, and publish the package at equinixmetal-helm/charts.

Make sure you fetch/pull the latest tags before making a new one.
It's recommended to only push the tag you just created.

```sh
$ git fetch --all --tags
Fetching origin
$ git tag --list
v0.1.0
v0.2.0
v0.3.0
v0.4.0
v0.4.1
$ git tag 0.5.0  # create the new tag
$ git tag --list
v0.1.0
v0.2.0
v0.3.0
v0.4.0
v0.4.1
0.5.0 # it's better not to prefix with v
$ git push origin 0.5.0  # push the new tag upstream
```
