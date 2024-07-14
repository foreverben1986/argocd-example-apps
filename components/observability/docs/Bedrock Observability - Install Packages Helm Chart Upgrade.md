This guild is intended to demonstrate the necessary steps and cautions on a Helm chart version update.

---

## Kustomize

As the kustomize Helm Charts are being kept statically on `./chats` folder, to update the Helm Chart you will need to clone the original chart repository and manually copy all files from the original directory to `./charts` folder.

Despite Helm don't support `CRDs` upgrade, with Kustomize the CRDs are indeed upgraded. You can test this manually by checking the result after applying in Kubernetes and also by running `kustomize build <Folder_path>`, that will produce the YAML file for the CRDs among the output, differently to what is mentioned in [Helm documentation](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/).

---

## Testing

At the current moment, no local environment for Observability is built, so all tests are being conducted in `Control Plane Preview` cluster. 
This is because very little attention is being devoted to Observability Stack in that environment, as All bedrock teammates have kubernetes access. 

Due to the same reason, all Control Plane logs are deactivated, to avoid unnecessary costs. Before making any change, make sure to enable the logs so you have a solid baseline to compare if anything goes wrong. 
To enable the logs, comment out the following lines:

```
# Filter everything to disable control Plane logs. Remove the line if you want any log to be sent.  
        - type: filter  
          expr: '"everything" == "everything"'
```

It is also recommended that all log levels are enabled to have a bigger log sample to verify if something don't work properly.
To enable all the logs levels, remove the filter from the processors list:

```
  service:  
    extensions: [ basicauth/metrics, basicauth/logs, health_check, memory_ballast, zpages ]  
    pipelines:  
      metrics:  
        receivers: [otlp, prometheus]  
        processors: [memory_limiter, batch, k8sattributes, resourcedetection]  
        exporters: [prometheusremotewrite]  
      logs:  
        receivers: [filelog]  
#        processors: [batch, transform/logs, filter/logs, resource/logs, k8sattributes]  
        processors: [batch, transform/logs, resource/logs, k8sattributes]  
        exporters: [loki]  
    telemetry:  
      metrics:  
        level: detailed  
        address: ":8888"  
      logs:  
        encoding: json
```

---

## Selecting the Correct Chart Version.

Usually, the chart version and the service version are not represented equally. For example, to implement OTEL collectors with version `0.88`, the OTEL collector chart version is `0.73.0` or `0.73.1`. This can be checked in the chart release notes or in the chart files. 

If possible, avoid `breaking changes` versions, so the configuration can change as minimally as possible. 
Not all breaking changes are applicable to your current configuration and even non-breaking changes could highly impact some use cases. For that reason, it's very important to review carefully the `upgrading.md` file, if available, and even take a look on the code changes to verify your service don't have any significant impact. 

Be mindful of [OTEL monitoring dashboard](https://grafana.com/grafana/dashboards/15983-opentelemetry-collector/), as even non-breaking changes can affect it's functionality.
Make sure to verify if there is no newer version for the dashboard when updating.


