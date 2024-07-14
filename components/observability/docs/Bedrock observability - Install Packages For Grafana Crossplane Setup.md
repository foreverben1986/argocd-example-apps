This guild is intended to demonstrate how the observability components work and correlate with Grafana Crossplane, focusing on the install packages and implementation details.

---

## Deployment Structure

For Bedrock-infra and observability generical Deployment Structure, reference [Bedrock observability - Install Packages Setup](Bedrock%20observability%20-%20Install%20Packages%20Setup.md).

---

## Crossplane Install Packages configuration

Crossplane configuration, just like Kubevela, is only required in Control Plane cluster, as the it's where the resources will be controlled at. 
Most of the Grafana Crossplane Resources does not utilize namespaces, making the control of such resources only doable by resource name.
Due to this fact, almost all OAM definitions utilizes the `"\(contex.appName)-..."` in order to keep track of the resource owner.

Grafana Crossplane components as per this moment include:
- 1 Crossplane chart
- 1 Provider for Grafana provider package.
- All CRD from https://github.com/grafana/crossplane-provider-grafana/tree/main/package folder on the desired release.
- 1 Provider Configuration to authenticate Crossplane in the desired Grafana Instance. Requires 1 `kind: ProviderConfig` per Grafana instance.

To connect to the Grafana instance, we need the application secrets. 
For that we are leveraging GCP Secret Manager and `SecretProviderClass.secrets-store.csi.x-k8s.io/v1` to get the secrets from GCP to Kubernetes. 
This secret is referenced in the `ProviderConfig`.

This secret is formatted as per Grafana Terraform Provider [Authorization](https://registry.terraform.io/providers/grafana/grafana/latest/docs#authentication) is JSON format. 
This is because Crossplane utilizes [Upbound](https://marketplace.upbound.io/providers/grafana/provider-grafana/v0.6.0/resources/grafana.crossplane.io/ProviderConfig/v1beta1) as a translation layer from Terraform, so all Crossplane resources will be translated to Terraform.

The final JSON should look like this:
```
{"auth": "Service_Account_API_Token", "oncall_access_token": "Non_Service_Account_OnCall_Token", "url": "Grafana_Instance_URL"}
```

As mentioned, this secret should be saved in GCP Secret Manager and retrieved using `CSI` Driver in Kubernetes.

### Live Control Plane Cluster

Not Applicable, as Grafana only requires the resource creation in one Cluster. 
As the CRDs are required due to Kubevela configuration, the Live Control Plane `ProviderConfig` shouldn't exist, to avoid duplication of Grafana Resources and UID conflicts.

### Workload Cluster

Not Applicable, as Grafana only requires the resource creation in one Cluster.

---

## CD Pipeline for Grafana Crossplane Resources

### Assumptions

The intent for observability team is to avoid the need of adoption from the teams, as this will require an extra step for every team to onboard to Observability stack.
Instead, we focus on offering an `opt-out` solution for the engineers, if needed.

On this regard, the Grafana IAC resource creation tries to leverage as much information as it can from the existing CD pipeline.
This means that, instead of plainly declaring the Crossplane Resources, Kubevela Components and Workflow-steps are used to create the resources.

As Kubevela Application has a 1Mb size limit, instead of injecting the dashboard JSON on the application, `ConfigMaps` will be used to input Dashboard Information from Kubernetes to Crossplane.
Although not often expected, if the `ConfigMap` is not synced before the Kubevela Workflow-step, the dashboard will not be created.
As this Workflow-step is the very last one, it's not expected to experience this issue.
If it happens, a Kubevela retrigger will be required. The pipeline will not fail due to this issue.

### Implementation

Following the described assumption, by default, every service that utilizes CD pipeline will automatically create a Grafana Folder with the Application Name.
The Kubevela Pipeline is only responsible for the resource creation, so even if the Sync or creation fails, the pipeline will not be compromised.

Together with the folder, if the app has a `dashboard/` folder in the repository root with dashboard JSONs, each JSON will be deployed in Grafana as a Dashboard.
To manage this, each JSON is converted to a `ConfigMap` file with the same file name. It's very important that the `$` are replaced by `\$` to avoid Kubernetes interpretation.
The `ConfigMap` will be added to `bedrock-application-<tenant>` repository and  synced to the cluster by Argo CD.
Those `ConfigMap` will be tracked be Kubevela through the `op.#List` definition based on the `matchingLabel: {"grafana": "dashboard"}` field, present in the `ConfigMap`.
For each `ConfigMap`, the `data.dashboard` content will be retrieved.
Before inserting the content in Crossplane resource,  the `\$` are replaced by `$` in order to allow Grafana Interpretation.
The Dashboard will then be created in Grafana on the folder with the same Application Name.

#### Manual Declaration

If more specific components need to be created, the Grafana Kubevela Components can be declared manually in the `bedrock-application.yaml`.

