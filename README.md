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

Create a `external-secret-honeycomb.yaml` template to get the application secret key from keymaker:

```yaml
---
apiVersion: keymaker.equinixmetal.com/v1
kind: ExternalSecretPull
metadata:
  name: "honeycomb-secret-key"
  labels:
    k8s-app: narwhal
  annotations:
    argocd.argoproj.io/sync-wave: '-1'
spec:
  backend: ssm
  mappings:
    - key: /prod/narwhal/narwhal-honeycomb-secret/v1
      name: api-key
```

In `values.yaml`, add the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable:

```yaml
appname:
  env:
    . . .
    OTEL_EXPORTER_OTLP_ENDPOINT: "http://opentelemetry-collector:55681"
```

Your application's OpenTelemetry configuration looks for that environment variable to determine where to send data.

## Architecture

By default, we're deploying one collector instance per application namespace.
