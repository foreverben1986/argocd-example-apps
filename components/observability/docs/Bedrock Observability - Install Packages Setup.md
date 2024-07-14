This guild is intended to demonstrate how the observability components work and correlate with Observability addons, focusing on the install packages. 

---

## Deployment Structure

The Bedrock-infra repository is being watched by Argo CD, with config files in [components/argocd](../../argocd) folder. 
According to the current configuration, Argo CD will be watching the main branch `components/**/overlays/preview/**` folders. 

Argo CD creates an Application within the Control Plane to track each component and addon. 
One known limitation is that the folder name much have the same name as the namespace the objects should be applied to. 

---

## Kustomize

To organize and facilitate the deploying, we are using `Kustomize` to control which files should be included and how they should behave. 

The kustomize file can be dived into 3 different sections: Resources, Helm Charts and Patches.

[Kustomize documentation.](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#kustomization)

### Resources

It will `kubectl apply` any file that is mentioned there. If the file is mentioned in the Helm chart, the mention in `resources` will result in an error.

### Helm Charts

Must have a `helmGlobals:  chartHome: <PATH>` pointing to the charts folder and the `helmCharts:` section where each chart that needs to be created will be declared.

> [!Attention]
> 
> The helm chart could be referenced as a link to GitHub repository, but Bedrock Team decided to keep an statical file of the chart in `helmGlobals: chartHome` so the CD pipeline doesn't have to download the charts at every deployment.
> 

In the `helmCharts:` section, a config file can be referenced, according to the specific chart documentation, to better encapsulate all configurations needed to a specific file.

### Patches

Patches are aimed to change dynamic configurations, such as environment properties, or to change configurations that are not editable through the chart native configuration.


### Testing Kustomize

Run `kustomize build <PATH_TO_FOLDER> --enable-helm ` to check all components that will be created.

### Kustomize Example

```
apiVersion: kustomize.config.k8s.io/v1beta1  
kind: Kustomization  
  
resources:  
  - ../../base  
  - iam-service-account.yaml  
  - otel-csi-secret.yaml  
  - secret-permissions.yaml  
  - workload-idenity-binding.yaml  
  - cluster-info-map.yaml  
  
helmGlobals:  
  chartHome: ../../base/charts  
  
helmCharts:  
  - name: opentelemetry-collector  
    namespace: observability  
    releaseName: bdrck-cluster-monitor-ctrlpln  
    valuesFile: collectors/otel-cluster-monitor.yaml  
    additionalValuesFiles:  
     - secrets/secretVolumes.yaml  
  - name: opentelemetry-collector  
    namespace: observability  
    releaseName: bdrck-gw-ctrlpln  
    valuesFile: collectors/otel-coll-deployment.yaml  
    additionalValuesFiles:  
     - secrets/secretVolumes.yaml  
  - name: opentelemetry-collector  
    namespace: observability  
    releaseName: bdrck-ctrlpln  
    valuesFile: collectors/otel-coll-daemonset.yaml  
    additionalValuesFiles:  
     - secrets/secretVolumes.yaml  
  - name: prometheus-node-exporter  
    namespace: observability  
    releaseName: nodeexporter  
    valuesFile: node-exporter/node-exporter-config.yaml  
  - name: kubernetes-event-exporter  
    namespace: observability  
    releaseName: eventexporter  
    valuesFile: event-exporter/event-exporter.yaml  
  
  
  
patches:  
- target:  
    kind: ServiceAccount  
    namespace: observability  
  patch: |-  
    - op: add  
      path: /metadata/annotations/iam.gke.io~1gcp-service-account  
      value: observability-sa@prj-bdrck-preview-ctrlpln.iam.gserviceaccount.com
```

---

## Kubevela

Kubevela applications can be utilized in the components install packages, being very useful for pre-configured components or traits that are already available in Kubevela. 
One example is the replication component that can be used to replicate a secret from control plane cluster to all `us-central1` workload clusters.

```
apiVersion: core.oam.dev/v1beta1  
kind: Application  
metadata:  
  name: observability-cred-replication  
  namespace: observability  
spec:  
  components:  
    - name: grafana-cloud  
      type: ref-objects  
      properties:  
        objects:  
          - resource: secret  
            name: grafana-cloud  
  policies:  
    - name: topology-us-central-1  
      type: topology  
      properties:  
        clusterLabelSelector:  
          region: us-central1
```


---

## Observability Install Packages configuration

Observability components as per this moment include:
- 3 OTEL collectors charts- each require 1 configuration file.
- 1 OTEL operator chart- no configuration file needed.
- 1 Kube-state-metrics chart - require 1 configuration file.
- 1 Kubernetes-event exporter chart - require 1 configuration file.
- 1 Instrumentation - Kubernetes resource applied directly.
- 1 Headless Gateway - Kubernetes resource applied directly.
- 1 Prometheus-node-exporter - require 1 configuration file.

As per the current configuration, Grafana Cloud client is used as the Final data receiver: Tempo (time series Database), Loki (Logs Database), and Mimir (Prometheus Metrics Database).
To connect to those services, we need the applications secrets. For that we are leveraging GCP Secret Manager and `SecretProviderClass.secrets-store.csi.x-k8s.io/v1` to get the secrets from GCP to Kubernetes. See [otel-csi-secret.yaml](../overlays/preview/otel-csi-secret.yaml)

In order to allow the secret access in GCP in need to provide a few components:
- An GCP IAM Service Account. See [iam-service-account.yaml](../overlays/preview/iam-service-account.yaml).
- An GCP IAM Policy Member for each secret to allow the IAM SA to access the secret. See [secret-permissions.yaml](../overlays/preview/secret-permissions.yaml).
- An GCP IAM Policy Member for each Kubernetes Service Account to the IAM SA that can access the required secret. See [workload-idenity-binding.yaml](../overlays/preview/workload-idenity-binding.yaml).
	- The Kubernetes Service Account is usually created by default in the chart creation.
- An annotation in every Kubernetes Service Account pointing to the expected GCP project. See [Kustomize Patch Example](Bedrock%20observability%20-%20Install%20Packages%20Setup.md#kustomize-example).

### Workload Cluster

In order to allow the secrets to be available in the workload clusters, a Kubevela Replication Component is created in Control Plane to replicate the kubernetes secret object created by the CSI driver in Workload clusters. This is the file workloadClusterSecretReplication.yaml which needs to be updated in `components/observability/base/instrumentation` folder.

---

## Additional Configuration

In Observability charts, we require the Cluster Name as a label in all metrics. For that, a manual ConfigMap is declared, mounted and mentioned in all Charts as a label.


