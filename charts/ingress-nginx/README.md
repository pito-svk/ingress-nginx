# ingress-nginx

[ingress-nginx](https://github.com/kubernetes/ingress-nginx) Ingress controller for Kubernetes using NGINX as a reverse proxy and load balancer

To use, add the `kubernetes.io/ingress.class: nginx` annotation to your Ingress resources.

This chart bootstraps an ingress-nginx deployment on a [Kubernetes](http://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.

## Prerequisites

- Kubernetes v1.16+

## Get Repo Info

```console
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

## Install Chart

**Important:** only helm3 is supported

```console
helm install [RELEASE_NAME] ingress-nginx/ingress-nginx
```

The command deploys ingress-nginx on the Kubernetes cluster in the default configuration.

_See [configuration](#configuration) below._

_See [helm install](https://helm.sh/docs/helm/helm_install/) for command documentation._

## Uninstall Chart

```console
helm uninstall [RELEASE_NAME]
```

This removes all the Kubernetes components associated with the chart and deletes the release.

_See [helm uninstall](https://helm.sh/docs/helm/helm_uninstall/) for command documentation._

## Upgrading Chart

```console
helm upgrade [RELEASE_NAME] [CHART] --install
```

_See [helm upgrade](https://helm.sh/docs/helm/helm_upgrade/) for command documentation._

### Upgrading With Zero Downtime in Production

By default the ingress-nginx controller has service interruptions whenever it's pods are restarted or redeployed. In order to fix that, see the excellent blog post by Lindsay Landry from Codecademy: [Kubernetes: Nginx and Zero Downtime in Production](https://medium.com/codecademy-engineering/kubernetes-nginx-and-zero-downtime-in-production-2c910c6a5ed8).

### Migrating from stable/nginx-ingress

There are two main ways to migrate a release from `stable/nginx-ingress` to `ingress-nginx/ingress-nginx` chart:

1. For Nginx Ingress controllers used for non-critical services, the easiest method is to [uninstall](#uninstall-chart) the old release and [install](#install-chart) the new one
1. For critical services in production that require zero-downtime, you will want to:
    1. [Install](#install-chart) a second Ingress controller
    1. Redirect your DNS traffic from the old controller to the new controller
    1. Log traffic from both controllers during this changeover
    1. [Uninstall](#uninstall-chart) the old controller once traffic has fully drained from it
    1. For details on all of these steps see [Upgrading With Zero Downtime in Production](#upgrading-with-zero-downtime-in-production)

Note that there are some different and upgraded configurations between the two charts, described by Rimas Mocevicius from JFrog in the "Upgrading to ingress-nginx Helm chart" section of [Migrating from Helm chart nginx-ingress to ingress-nginx](https://rimusz.net/migrating-to-ingress-nginx). As the `ingress-nginx/ingress-nginx` chart continues to update, you will want to check current differences by running [helm configuration](#configuration) commands on both charts.

## Configuration

See [Customizing the Chart Before Installing](https://helm.sh/docs/intro/using_helm/#customizing-the-chart-before-installing). To see all configurable options with detailed comments, visit the chart's [values.yaml](./values.yaml), or run these configuration commands:

```console
helm show values ingress-nginx/ingress-nginx
```

### PodDisruptionBudget

Note that the PodDisruptionBudget resource will only be defined if the replicaCount is greater than one,
else it would make it impossible to evacuate a node. See [gh issue #7127](https://github.com/helm/charts/issues/7127) for more info.

### Prometheus Metrics

The Nginx ingress controller can export Prometheus metrics, by setting `controller.metrics.enabled` to `true`.

You can add Prometheus annotations to the metrics service using `controller.metrics.service.annotations`. Alternatively, if you use the Prometheus Operator, you can enable ServiceMonitor creation using `controller.metrics.serviceMonitor.enabled`.

### ingress-nginx nginx\_status page/stats server

Previous versions of this chart had a `controller.stats.*` configuration block, which is now obsolete due to the following changes in nginx ingress controller:

- In [0.16.1](https://github.com/kubernetes/ingress-nginx/blob/master/Changelog.md#0161), the vts (virtual host traffic status) dashboard was removed
- In [0.23.0](https://github.com/kubernetes/ingress-nginx/blob/master/Changelog.md#0230), the status page at port 18080 is now a unix socket webserver only available at localhost.
  You can use `curl --unix-socket /tmp/nginx-status-server.sock http://localhost/nginx_status` inside the controller container to access it locally, or use the snippet from [nginx-ingress changelog](https://github.com/kubernetes/ingress-nginx/blob/master/Changelog.md#0230) to re-enable the http server

### ExternalDNS Service Configuration

Add an [ExternalDNS](https://github.com/kubernetes-incubator/external-dns) annotation to the LoadBalancer service:

```yaml
controller:
  service:
    annotations:
      external-dns.alpha.kubernetes.io/hostname: kubernetes-example.com.
```

### AWS L7 ELB with SSL Termination

Annotate the controller as shown in the [nginx-ingress l7 patch](https://github.com/kubernetes/ingress-nginx/blob/master/deploy/aws/l7/service-l7.yaml):

```yaml
controller:
  service:
    targetPorts:
      http: http
      https: http
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:XX-XXXX-X:XXXXXXXXX:certificate/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXX
      service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
      service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "https"
      service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: '3600'
```

### AWS route53-mapper

To configure the LoadBalancer service with the [route53-mapper addon](https://github.com/kubernetes/kops/tree/master/addons/route53-mapper), add the `domainName` annotation and `dns` label:

```yaml
controller:
  service:
    labels:
      dns: "route53"
    annotations:
      domainName: "kubernetes-example.com"
```

### Additional Internal Load Balancer

This setup is useful when you need both external and internal load balancers but don't want to have multiple ingress controllers and multiple ingress objects per application.

By default, the ingress object will point to the external load balancer address, but if correctly configured, you can make use of the internal one if the URL you are looking up resolves to the internal load balancer's URL.

You'll need to set both the following values:

`controller.service.internal.enabled`
`controller.service.internal.annotations`

If one of them is missing the internal load balancer will not be deployed. Example you may have `controller.service.internal.enabled=true` but no annotations set, in this case no action will be taken.

`controller.service.internal.annotations` varies with the cloud service you're using.

Example for AWS:

```yaml
controller:
  service:
    internal:
      enabled: true
      annotations:
        # Create internal ELB
        service.beta.kubernetes.io/aws-load-balancer-internal: 0.0.0.0/0
        # Any other annotation can be declared here.
```

Example for GCE:

```yaml
controller:
  service:
    internal:
      enabled: true
      annotations:
        # Create internal LB
        cloud.google.com/load-balancer-type: "Internal"
        # Any other annotation can be declared here.
```

Example for Azure:

```yaml
controller:
  service:
      annotations:
        # Create internal LB
        service.beta.kubernetes.io/azure-load-balancer-internal: "true"
        # Any other annotation can be declared here.
```

An use case for this scenario is having a split-view DNS setup where the public zone CNAME records point to the external balancer URL while the private zone CNAME records point to the internal balancer URL. This way, you only need one ingress kubernetes object.

Optionally you can set `controller.service.loadBalancerIP` if you need a static IP for the resulting `LoadBalancer`.

### Ingress Admission Webhooks

With nginx-ingress-controller version 0.25+, the nginx ingress controller pod exposes an endpoint that will integrate with the `validatingwebhookconfiguration` Kubernetes feature to prevent bad ingress from being added to the cluster.
**This feature is enabled by default since 0.31.0.**

With nginx-ingress-controller in 0.25.* work only with kubernetes 1.14+, 0.26 fix [this issue](https://github.com/kubernetes/ingress-nginx/pull/4521)

### Helm Error When Upgrading: spec.clusterIP: Invalid value: ""

If you are upgrading this chart from a version between 0.31.0 and 1.2.2 then you may get an error like this:

```console
Error: UPGRADE FAILED: Service "?????-controller" is invalid: spec.clusterIP: Invalid value: "": field is immutable
```

Detail of how and why are in [this issue](https://github.com/helm/charts/pull/13646) but to resolve this you can set `xxxx.service.omitClusterIP` to `true` where `xxxx` is the service referenced in the error.

As of version `1.26.0` of this chart, by simply not providing any clusterIP value, `invalid: spec.clusterIP: Invalid value: "": field is immutable` will no longer occur since `clusterIP: ""` will not be rendered.


## Configuration

| Parameter                          | Description                                                                                                                                                                                                                                               | Default                                         |
|------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| `controller.name`          | The [anti-affinity][] topology key. By default this will prevent multiple Elasticsearch nodes from running on the same Kubernetes node                                                                                                                    | `kubernetes.io/hostname`                        |
| `controller.image.repository`                     | Setting this to hard enforces the [anti-affinity][] rules. If it is set to soft it will be done "best effort". Other values will be ignored                                                                                                               | `hard`                                          |
| `controller.image.tag`         | The [Elasticsearch cluster health status params][] that will be used by readiness [probe][] command                                                                                                                                                       | `controller.image.digest`              |
| `clusterName`                      | This will be used as the Elasticsearch [cluster.name][] and should be unique per cluster in the namespace                                                                                                                                                 | `elasticsearch`                                 |
| `enableServiceLinks`               | Set to false to disabling service links, which can cause slow pod startup times when there are many services in the current namespace.                                                                                                                    | `true`                                          |
| `envFrom`                          | Templatable string to be passed to the [environment from variables][] which will be appended to the `envFrom:` definition for the container                                                                                                               | `[]`                                            |
| `esConfig`                         | Allows you to add any config files in `/usr/share/elasticsearch/config/` such as `elasticsearch.yml` and `log4j2.properties`. See [values.yaml][] for an example of the formatting                                                                        | `{}`                                            |
| `esJavaOpts`                       | [Java options][] for Elasticsearch. This is where you should configure the [jvm heap size][]                                                                                                                                                              | `-Xmx1g -Xms1g`                                 |
| `esMajorVersion`                   | Deprecated. Instead, use the version of the chart corresponding to your ES minor version. Used to set major version specific configuration. If you are using a custom image and not running the default Elasticsearch version you will need to set this to the version you are running (e.g. `esMajorVersion: 6`)                                   | `""`                                            |
| `extraContainers`                  | Templatable string of additional `containers` to be passed to the `tpl` function                                                                                                                                                                          | `""`                                            |
| `extraEnvs`                        | Extra [environment variables][] which will be appended to the `env:` definition for the container                                                                                                                                                         | `[]`                                            |
| `extraInitContainers`              | Templatable string of additional `initContainers` to be passed to the `tpl` function                                                                                                                                                                      | `""`                                            |
| `extraVolumeMounts`                | Templatable string of additional `volumeMounts` to be passed to the `tpl` function                                                                                                                                                                        | `""`                                            |
| `extraVolumes`                     | Templatable string of additional `volumes` to be passed to the `tpl` function                                                                                                                                                                             | `""`                                            |
| `fullnameOverride`                 | Overrides the `clusterName` and `nodeGroup` when used in the naming of resources. This should only be used when using a single `nodeGroup`, otherwise you will have name conflicts                                                                        | `""`                                            |
| `hostAliases`                      | Configurable [hostAliases][]                                                                                                                                                                                                                              | `[]`                                            |
| `httpPort`                         | The http port that Kubernetes will use for the healthchecks and the service. If you change this you will also need to set [http.port][] in `extraEnvs`                                                                                                    | `9200`                                          |
| `imagePullPolicy`                  | The Kubernetes [imagePullPolicy][] value                                                                                                                                                                                                                  | `IfNotPresent`                                  |
| `imagePullSecrets`                 | Configuration for [imagePullSecrets][] so that you can use a private registry for your image                                                                                                                                                              | `[]`                                            |
| `imageTag`                         | The Elasticsearch Docker image tag                                                                                                                                                                                                                        | `8.0.0-SNAPSHOT`                                |
| `image`                            | The Elasticsearch Docker image                                                                                                                                                                                                                            | `docker.elastic.co/elasticsearch/elasticsearch` |
| `ingress`                          | Configurable [ingress][] to expose the Elasticsearch service. See [values.yaml][] for an example                                                                                                                                                          | see [values.yaml][]                             |
| `initResources`                    | Allows you to set the [resources][] for the `initContainer` in the StatefulSet                                                                                                                                                                            | `{}`                                            |
| `keystore`                         | Allows you map Kubernetes secrets into the keystore. See the [config example][] and [how to use the keystore][]                                                                                                                                           | `[]`                                            |
| `labels`                           | Configurable [labels][] applied to all Elasticsearch pods                                                                                                                                                                                                 | `{}`                                            |
| `lifecycle`                        | Allows you to add [lifecycle hooks][]. See [values.yaml][] for an example of the formatting                                                                                                                                                               | `{}`                                            |
| `masterService`                    | The service name used to connect to the masters. You only need to set this if your master `nodeGroup` is set to something other than `master`. See [Clustering and Node Discovery][] for more information                                                 | `""`                                            |
| `masterTerminationFix`             | A workaround needed for Elasticsearch < 7.2 to prevent master status being lost during restarts [#63][]                                                                                                                                                   | `false`                                         |
| `maxUnavailable`                   | The [maxUnavailable][] value for the pod disruption budget. By default this will prevent Kubernetes from having more than 1 unhealthy pod in the node group                                                                                               | `1`                                             |
| `minimumMasterNodes`               | The value for [discovery.zen.minimum_master_nodes][]. Should be set to `(master_eligible_nodes / 2) + 1`. Ignored in Elasticsearch versions >= 7                                                                                                          | `2`                                             |
| `nameOverride`                     | Overrides the `clusterName` when used in the naming of resources                                                                                                                                                                                          | `""`                                            |
| `networkHost`                      | Value for the [network.host Elasticsearch setting][]                                                                                                                                                                                                      | `0.0.0.0`                                       |
| `networkPolicy`                    | The [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) to set. See [`values.yaml`](./values.yaml) for an example                                                                                                  | `{http.enabled: false,transport.enabled: false}`|
| `nodeAffinity`                     | Value for the [node affinity settings][]                                                                                                                                                                                                                  | `{}`                                            |
| `nodeGroup`                        | This is the name that will be used for each group of nodes in the cluster. The name will be `clusterName-nodeGroup-X` , `nameOverride-nodeGroup-X` if a `nameOverride` is specified, and `fullnameOverride-X` if a `fullnameOverride` is specified        | `master`                                        |
| `nodeSelector`                     | Configurable [nodeSelector][] so that you can target specific nodes for your Elasticsearch cluster                                                                                                                                                        | `{}`                                            |
| `persistence`                      | Enables a persistent volume for Elasticsearch data. Can be disabled for nodes that only have [roles][] which don't require persistent data                                                                                                                | see [values.yaml][]                             |
| `podAnnotations`                   | Configurable [annotations][] applied to all Elasticsearch pods                                                                                                                                                                                            | `{}`                                            |
| `podManagementPolicy`              | By default Kubernetes [deploys StatefulSets serially][]. This deploys them in parallel so that they can discover each other                                                                                                                               | `Parallel`                                      |
| `podSecurityContext`               | Allows you to set the [securityContext][] for the pod                                                                                                                                                                                                     | see [values.yaml][]                             |
| `podSecurityPolicy`                | Configuration for create a pod security policy with minimal permissions to run this Helm chart with `create: true`. Also can be used to reference an external pod security policy with `name: "externalPodSecurityPolicy"`                                | see [values.yaml][]                             |
| `priorityClassName`                | The name of the [PriorityClass][]. No default is supplied as the PriorityClass must be created first                                                                                                                                                      | `""`                                            |
| `protocol`                         | The protocol that will be used for the readiness [probe][]. Change this to `https` if you have `xpack.security.http.ssl.enabled` set                                                                                                                      | `http`                                          |
| `rbac`                             | Configuration for creating a role, role binding and ServiceAccount as part of this Helm chart with `create: true`. Also can be used to reference an external ServiceAccount with `serviceAccountName: "externalServiceAccountName"`                       | see [values.yaml][]                             |
| `readinessProbe`                   | Configuration fields for the readiness [probe][]                                                                                                                                                                                                          | see [values.yaml][]                             |
| `replicas`                         | Kubernetes replica count for the StatefulSet (i.e. how many pods)                                                                                                                                                                                         | `3`                                             |
| `resources`                        | Allows you to set the [resources][] for the StatefulSet                                                                                                                                                                                                   | see [values.yaml][]                             |
| `roles`                            | A hash map with the specific [roles][] for the `nodeGroup`                                                                                                                                                                                                | see [values.yaml][]                             |
| `schedulerName`                    | Name of the [alternate scheduler][]                                                                                                                                                                                                                       | `""`                                            |
| `secretMounts`                     | Allows you easily mount a secret as a file inside the StatefulSet. Useful for mounting certificates and other secrets. See [values.yaml][] for an example                                                                                                 | `[]`                                            |
| `securityContext`                  | Allows you to set the [securityContext][] for the container                                                                                                                                                                                               | see [values.yaml][]                             |
| `service.annotations`              | [LoadBalancer annotations][] that Kubernetes will use for the service. This will configure load balancer if `service.type` is `LoadBalancer`                                                                                                              | `{}`                                            |
| `service.externalTrafficPolicy`    | Some cloud providers allow you to specify the [LoadBalancer externalTrafficPolicy][]. Kubernetes will use this to preserve the client source IP. This will configure load balancer if `service.type` is `LoadBalancer`                                    | `""`                                            |
| `service.httpPortName`             | The name of the http port within the service                                                                                                                                                                                                              | `http`                                          |
| `service.labelsHeadless`           | Labels to be added to headless service                                                                                                                                                                                                                    | `{}`                                            |
| `service.labels`                   | Labels to be added to non-headless service                                                                                                                                                                                                                | `{}`                                            |
| `service.loadBalancerIP`           | Some cloud providers allow you to specify the [loadBalancer][] IP. If the `loadBalancerIP` field is not specified, the IP is dynamically assigned. If you specify a `loadBalancerIP` but your cloud provider does not support the feature, it is ignored. | `""`                                            |
| `service.loadBalancerSourceRanges` | The IP ranges that are allowed to access                                                                                                                                                                                                                  | `[]`                                            |
| `service.nodePort`                 | Custom [nodePort][] port that can be set if you are using `service.type: nodePort`                                                                                                                                                                        | `""`                                            |
| `service.transportPortName`        | The name of the transport port within the service                                                                                                                                                                                                         | `transport`                                     |
| `service.type`                     | Elasticsearch [Service Types][]                                                                                                                                                                                                                           | `ClusterIP`                                     |
| `sidecarResources`                 | Allows you to set the [resources][] for the sidecar containers in the StatefulSet                                                                                                                                                                         | {}                                              |
| `sysctlInitContainer`              | Allows you to disable the `sysctlInitContainer` if you are setting [sysctl vm.max_map_count][] with another method                                                                                                                                        | `enabled: true`                                 |
| `sysctlVmMaxMapCount`              | Sets the [sysctl vm.max_map_count][] needed for Elasticsearch                                                                                                                                                                                             | `262144`                                        |
| `terminationGracePeriod`           | The [terminationGracePeriod][] in seconds used when trying to stop the pod                                                                                                                                                                                | `120`                                           |
| `tolerations`                      | Configurable [tolerations][]                                                                                                                                                                                                                              | `[]`                                            |
| `transportPort`                    | The transport port that Kubernetes will use for the service. If you change this you will also need to set [transport port configuration][] in `extraEnvs`                                                                                                 | `9300`                                          |
| `updateStrategy`                   | The [updateStrategy][] for the StatefulSet. By default Kubernetes will wait for the cluster to be green after upgrading each pod. Setting this to `OnDelete` will allow you to manually delete each pod during upgrades                                   | `RollingUpdate`                                 |
| `volumeClaimTemplate`              | Configuration for the [volumeClaimTemplate for StatefulSets][]. You will want to adjust the storage (default `30Gi` ) and the `storageClassName` if you are using a different storage class                                                               | see [values.yaml][]                             |

### Deprecated

| Parameter | Description                                                                                                   | Default |
|-----------|---------------------------------------------------------------------------------------------------------------|---------|
| `fsGroup` | The Group ID (GID) for [securityContext][] so that the Elasticsearch user can read from the persistent volume | `""`  
