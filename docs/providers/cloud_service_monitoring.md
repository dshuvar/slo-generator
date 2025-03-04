# Cloud Service Monitoring

## Backend

Using the `cloud_service_monitoring` backend, you can use the
`Cloud Service Monitoring API` to manage your SLOs.

```yaml
backends:
  cloud_service_monitoring:
    project_id: "${WORKSPACE_PROJECT_ID}"
```

SLOs are created from standard metrics available in Cloud Monitoring and
the data is stored in `Cloud Service Monitoring API` (see
  [docs](https://cloud.google.com/monitoring/service-monitoring/using-api)).

The following methods are available to compute SLOs with the `cloud_service_monitoring`
backend:

* `basic` to create standard SLOs for Google App Engine, Google Kubernetes
Engine, and Cloud Endpoints.
* `good_bad_ratio` for metrics of type `DELTA` or `CUMULATIVE`.
* `distribution_cut` for metrics of type `DELTA` and unit `DISTRIBUTION`.


### Basic

The `basic` method is used to let the `Cloud Service Monitoring API`
automatically generate standardized SLOs for the following GCP services:
* **Google App Engine**
* **Google Kubernetes Engine** (with Istio)
* **Google Cloud Endpoints**

The SLO configuration uses Cloud Monitoring
[GCP metrics](https://cloud.google.com/monitoring/api/metrics_gcp) and only
requires minimal configuration compared to custom SLOs.

**Example config (App Engine availability):**

```yaml
backend: cloud_service_monitoring
method: basic
service_level_indicator:
  app_engine:
    project_id: ${GAE_PROJECT_ID}
    module_id: ${GAE_MODULE_ID}
  availability: {}
```
For details on filling the `app_engine` fields, see [AppEngine](https://cloud.google.com/monitoring/api/ref_v3/rest/v3/services#appengine)
spec.

**&rightarrow; [Full SLO config](../../samples/cloud_service_monitoring/slo_gae_app_availability_basic.yaml)**

**Example config (Cloud Endpoint latency):**

```yaml
backend: cloud_service_monitoring
method: basic
service_level_indicator:
  cloud_endpoints:
    service_name: ${ENDPOINT_URL}
  latency:
    threshold: 724 # ms
```
For details on filling the `cloud_endpoints` fields, see [CloudEndpoint](https://cloud.google.com/monitoring/api/ref_v3/rest/v3/services#cloudendpoints)
spec.

**Example config (Istio service latency):**
```yaml
backend: cloud_service_monitoring
method: basic
service_level_indicator:
  mesh_istio:
    mesh_uid: ${GKE_MESH_UID}
    service_namespace: ${GKE_SERVICE_NAMESPACE}
    service_name: ${GKE_SERVICE_NAME}
  latency:
    threshold: 500 # ms
```
For details on filling the `mesh_istio` fields, see [MeshIstio](https://cloud.google.com/monitoring/api/ref_v3/rest/v3/services#meshistio)
spec.

**&rightarrow; [Full SLO config](../../samples/cloud_service_monitoring/slo_gke_app_latency_basic.yaml)**

**Example config (Istio service latency) [DEPRECATED]:**
```yaml
backend: cloud_service_monitoring
method: basic
service_level_indicator:
  cluster_istio:
    project_id: ${GKE_PROJECT_ID}
    location: ${GKE_LOCATION}
    cluster_name: ${GKE_CLUSTER_NAME}
    service_namespace: ${GKE_SERVICE_NAMESPACE}
    service_name: ${GKE_SERVICE_NAME}
  latency:
    threshold: 500 # ms
```
For details on filling the `cluster_istio` fields, see [ClusterIstio](https://cloud.google.com/monitoring/api/ref_v3/rest/v3/services#clusteristio)
spec.

**&rightarrow; [Full SLO config](../../samples/cloud_service_monitoring/slo_gke_app_latency_basic_deprecated.yaml)**


### Good / bad ratio

The `good_bad_ratio` method is used to compute the ratio between two metrics:

- **Good events**, i.e events we consider as 'good' from the user perspective.
- **Bad or valid events**, i.e events we consider either as 'bad' from the user
perspective, or all events we consider as 'valid' for the computation of the
SLO.

This method is often used for availability SLOs, but can be used for other
purposes as well (see examples).

**Example config:**
```yaml
backend: cloud_service_monitoring
method: good_bad_ratio
service_level_indicator:
  filter_good:  >
    project="${GAE_PROJECT_ID}"
    metric.type="appengine.googleapis.com/http/server/response_count"
    resource.type="gae_app"
    metric.labels.response_code >= 200
    metric.labels.response_code < 500
  filter_valid: >
    project="${GAE_PROJECT_ID}"
    metric.type="appengine.googleapis.com/http/server/response_count"
```

You can also use the `filter_bad` field which identifies bad events instead of
the `filter_valid` field which identifies all valid events.

**&rightarrow; [Full SLO config](../../samples/cloud_service_monitoring/slo_gae_app_availability.yaml)**

## Distribution cut

The `distribution_cut` method is used for Cloud distribution-type metrics,
which are usually used for latency metrics.

A distribution metric records the **statistical distribution of the extracted
values** in **histogram buckets**. The extracted values are not recorded
individually, but their distribution across the configured buckets are recorded,
along with the `count`, `mean`, and `sum` of squared deviation of the values.

**Example config:**

```yaml
backend: cloud_service_monitoring
method: distribution_cut
service_level_indicator:
  filter_valid: >
    project=${GAE_PROJECT_ID}
    metric.type=appengine.googleapis.com/http/server/response_latencies
    metric.labels.response_code >= 200
    metric.labels.response_code < 500
  range_min: 0
  range_max: 724 # ms
```

The `range_min` and `range_max` are used to specify the latency range that we
consider 'good'.

**&rightarrow; [Full SLO config](../../samples/cloud_service_monitoring/slo_gae_app_latency.yaml)**


## Service Monitoring API considerations

### Tracking objects
Since `Cloud Service Monitoring API` persists `Service` and
`ServiceLevelObjective` objects, we need ways to keep our local SLO YAML
configuration synced with the remote objects.

**Auto-imported**

Some services are auto-imported by the `Service Monitoring API`: they correspond
to SLO configurations using the [`basic`](#basic) method.

The following conventions are used by the `Service Monitoring API` to give a
unique id to an auto-imported `Service`:

  * **App Engine:**

    ```
    gae:{project_id}_{module_id}
    ```
    &rightarrow; *Make sure that the `app_engine` block in your config has the
    correct fields corresponding to your App Engine service.*

  * **Cloud Endpoints:**

    ```
    ist:{project_id}-{service}
    ```
    &rightarrow; *Make sure that the `cloud_endpoints` block in your config has
    the correct fields corresponding to your Cloud Endpoint service.*

  * **Mesh Istio [NOT YET RELEASED]:**

    ```
    ist:{project_id}-{mesh_uid}-{service_namespace}-{service_name}
    ```
    &rightarrow; *Make sure that the `mesh_istio` block in your config has the
    correct fields corresponding to your Istio service.*

  * **Cluster Istio [DEPRECATED SOON]:**

    ```
    ist:{project_id}-{suffix}-{location}-{cluster_name}-{service_namespace}-{service_name}
    ```
    &rightarrow; *Make sure that the `cluster_istio` block in your config has
    the correct fields corresponding to your Istio service.*

You cannot import an existing `ServiceLevelObjective` object, since they use a
random id.

**Custom**

Custom services are the ones you create yourself using the
`Cloud Service Monitoring API` and the `slo-generator`.

The following conventions are used by the `slo-generator` to give a unique id
to a custom `Service` and `Service Level Objective` objects:

* `service_id = ${metadata.service_name}-${metadata.feature_name}`

* `slo_id = ${metadata.service_name}-${metadata.feature_name}-${metadata.slo_name}-${window}`

To keep track of those, **do not update any of the following fields** in your
configs:

  * `metadata.service_name`, `metadata.feature_name` and `metadata.slo_name` in the SLO config.

  * `window` in the Error Budget Policy.

If you need to make updates to any of those fields, first run the
`slo-generator` with the `-d` (delete) option (see
  [#deleting-objects](#deleting-objects)), then re-run normally.

To import an existing custom `Service` objects, find out your service id from
the API and fill the `service_id` in the `service_level_indicator` configuration.

### Deleting objects

To delete an SLO object in `Cloud Monitoring API` using the
`cloud_service_monitoring` class, run the `slo-generator` with the
`-d` (or `--delete`) flag:

```
slo-generator -f <SLO_CONFIG_PATH> -b <ERROR_BUDGET_POLICY> --delete
```

## Alerting

See the Cloud Service Monitoring [docs](https://cloud.google.com/monitoring/service-monitoring/alerting-on-budget-burn-rate)
for instructions on alerting.

### Examples

Complete SLO samples using `Cloud Service Monitoring` are available in [ samples/cloud_service_monitoring](../../samples/cloud_service_monitoring).
Check them out !
