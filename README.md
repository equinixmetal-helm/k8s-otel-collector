# Equinix Helm chart for the OpenTelemetry Collector

This chart does the following:

- creates a Kubernetes [deployment](templates/deployment.yaml) of pods that use the specified version of the
  [otel/opentelemetry-collector-contrib Docker image](https://hub.docker.com/r/otel/opentelemetry-collector-contrib) from Docker Hub
- mounts a volume for the [Collector configuration file](templates/opentelemetry-collector-config.yaml), which is generated from a template that pulls cluster metadata from Atlas
- creates a Kubernetes [service](templates/service.yaml) for the Collector, exposing relevant ports
- pulls the Honeycomb API key stored in keymaker

This configuration is specific to the deployment strategy we're using at Metal:
**one Collector "app" per application namespace** (e.g. cacher, narhwal, boots, etc.).

## Deploy the OpenTelemetry Collector for your service

### Add the Collector chart to your app's namespace in Atlas

In the packethost/delivery-infrastructure repository, locate your service under your team's directory in [`atlas/apps.d`](https://github.com/packethost/delivery-infrastructure/tree/main/atlas/apps.d).
Add `otel-collector` and the repoURL (git URL) as an additional list item under the `apps` section of your application's configuration:

```yaml
    - name: otel-collector
      repoURL: "git@github.com:packethost/k8s-otel-collector.git"
```

[Here's what the configuration looks like for Bouncer.](https://github.com/packethost/delivery-infrastructure/blob/f14004104df373fd63856600a2ebd14b80042652/atlas/apps.d/nautilus/bouncer.yaml#L19-L20)

Once your PR is merged, you will need to ask the Delivery team (`#em-delivery-eng`) to sync cluster-apps.
That will pick up the reference in Atlas to this Helm chart.
The new `otel-collector` app will appear in your service's namespace as out-of-sync, and at that point you can sync the resources in Argo.

## Configure your service to use the Collector

Once you've deployed the Collector itself to your application namespace, you can update your app configuration to enable sending telemetry data to the Collector.

### Add OpenTelemetry instrumentation to the application code

Go apps should use [equinix-labs/otel-init-go](https://github.com/equinix-labs/otel-init-go).
Follow the configuration instructions in the README.

For Ruby apps, follow the instructions in [Confluence](https://packet.atlassian.net/l/c/XBP11Ef4).

### Set the OTLP endpoint in the app Helm chart

Your application's OpenTelemetry SDK configuration looks for the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable to determine where to send data.
In this case the OTLP endpoint is the static url of the collector deployed to that application's namespace.

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

If you're not sure where to add the environment variable, ask SRE (`#sre`) or the Delivery team (`#em-delivery-eng`) for help.

### Sync in Argo

For initial deployment and any changes to the OTLP endpoint, the app's pods will need to be restarted in order to pick up the new/updated environment variables.
For some configurations, Argo will restart the pods automatically.
For others, you may need to manually restart the pods.
Reach out to SRE (`#sre`) or the Delivery team (`#em-delivery-eng`) if you need help with that.

## Managing Honeycomb API keys

As of August 2022, Metal services share a global Honeycomb key for each environment.
Service teams no longer need to worry about managing Honeycomb keys for their services.

### Rotating a Honeycomb key

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
