# Equinix Helm chart for the OpenTelemetry Collector

Based on [Honeycomb's Helm chart](https://github.com/honeycombio/helm-charts/tree/main/charts/opentelemetry-collector),
but significantly pared down.

## Using this chart

Generate a tarball from this Helm chart and place it in the `charts/` directory of the target application's k8-site repository.

```sh
cd k8s-site-appname
mkdir charts
helm package ~/path/to/helm-equinix-opentelemetry-collector --destination ./charts
```

In `values.yaml`, add the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable:

```yaml
appname:
  env:
    . . .
    OTEL_EXPORTER_OTLP_ENDPOINT: "http://opentelemetry-collector:55681"
```

Your application's OpenTelemetry configuration looks for that environment variable to determine where to send data.

Note: If the value for the OTLP endpoint is updated from a previous version running in your cluster,
you will need to restart your pods so that they pick up the new endpoint.

## Troubleshooting

[Follow the instructions in these docs](https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/troubleshooting.md)
to set the Collector's own logs to `DEBUG`.

## Architecture

By default, we're deploying one collector instance per application namespace.
