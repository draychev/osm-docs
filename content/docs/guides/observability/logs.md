---
title: "Logs"
description: "Diagnostic logs from the OSM control plane"
type: docs
---

# Logs
Open Service Mesh (OSM) control plane components log diagnostic messages to stdout to aid in managing a mesh.

In the logs, users can expect to see the following kinds of information
alongside messages:
- Kubernetes resource metadata, like names and namespaces
- mTLS certificate common names

OSM will **not** log sensitive information, such as:
- Kubernetes Secret data
- entire Kubernetes resources

## Verbosity

Log verbosity controls when certain log messages are written, for example to
include more messages for debugging or to include fewer messages that only point
to critical errors.

OSM defines the following log levels in order of increasing verbosity:

| Log level | Purpose                                                                                |
| --------- | -------------------------------------------------------------------------------------- |
| disabled  | Disables logging entirely                                                              |
| panic     | *Currently unused*                                                                     |
| fatal     | For unrecoverable errors resulting in termination, usually on startup                  |
| error     | For errors that may require user action to resolve                                     |
| warn      | For recovered errors or unexpected conditions that may lead to errors                  |
| info      | For messages indicating normal behavior, such as acknowledging some user action        |
| debug     | For extra information useful in figuring out why a mesh may not be working as expected |
| trace     | For extra verbose messages, used primarily for development                             |

Each of the above log levels can be configured in the MeshConfig at
`spec.observability.osmLogLevel` or on install with the
`osm.controllerLogLevel` chart value.

## Fluent Bit
When enabled, Fluent Bit can collect these logs, process them and send them to an output of the user's choice such as Elasticsearch, Azure Log Analytics, BigQuery, etc.

[Fluent Bit](https://fluentbit.io/) is an open source log processor and forwarder which allows you to collect data/logs and send them to multiple destinations. It can be used with OSM to forward OSM controller logs to a variety of outputs/log consumers by using its output plugins.

OSM provides log forwarding by optionally deploying a Fluent Bit sidecar to the OSM controller using the `--set=osm.enableFluentbit=true` flag during installation. The user can then define where OSM logs should be forwarded using any of the available [Fluent Bit output plugins](https://docs.fluentbit.io/manual/pipeline/outputs).

### Configuring Log Forwarding with Fluent Bit
By default, the Fluent Bit sidecar is configured to simply send logs to the Fluent Bit container's stdout. If you have installed OSM with Fluent Bit enabled, you may access these logs using `kubectl logs -n <osm-namespace> <osm-controller-name> -c fluentbit-logger`. This command will also help you find how your logs are formatted in case you need to change your parsers and filters.

> Note: `<osm-namespace>` refers to the namespace where the osm control plane is installed.

To quickly bring up Fluent Bit with default values, use the `--set=OpenServiceMesh.enableFluentbit` option:
```console
osm install --set=osm.enableFluentbit=true
```
By default, logs will be filtered to emit info level logs. You may change the log level to "debug", "warn", "fatal", "panic", "disabled" or "trace" during installation using `--set osm.controllerLogLevel=<desired log level>` . To get _all_ logs, set the log level to trace.

Once you have tried out this basic setup, we recommend configuring log forwarding to your preferred output for more informative results.

To customize log forwarding to your output, follow these steps and then reinstall OSM with Fluent Bit enabled.

1. Find the output plugin you would like to forward your logs to in [Fluent Bit documentation](https://docs.fluentbit.io/manual/pipeline/outputs). Replace the `[OUTPUT]` section in [`fluentbit-configmap.yaml`](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/templates/fluentbit-configmap.yaml) with appropriate values.

1. The default configuration uses CRI log format parsing. If you are using a kubernetes distribution that causes your logs to be formatted differently, you may need to add a new parser to the `[PARSER]` section and change the `parser` name in the `[INPUT]` section to one of the parsers defined [here](https://github.com/fluent/fluent-bit/blob/master/conf/parsers.conf).

1. Explore available [Fluent Bit Filters](https://docs.fluentbit.io/manual/pipeline/filters) and add as many `[FILTER]` sections as desired.
    * The `[INPUT]` section tags ingested logs with `kube.*` so make sure to include `Match kube.*` key/value pair in each of your custom filters.
    * The default configuration uses a modify filter to add a `controller_pod_name` key/value pair to help you query logs in your output by refining results on pod name (see example usage below).

1. For these changes to take effect, run:
    ```console
    make build-osm
    ```

1. Once you have updated the Fluent Bit ConfigMap template, you can deploy Fluent Bit during OSM installation using:
    ```console
    osm install --set=osm.enableFluentbit=true [--set osm.controllerLogLevel=<desired log level>]
    ```
    You should now be able to interact with error logs in the output of your choice as they get generated.


### Example: Using Fluent Bit to send logs to Azure Monitor
Fluent Bit has an Azure output plugin that can be used to send logs to an Azure Log Analytics workspace as follows:
1. [Create a Log Analytics workspace](https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-create-workspace)

2. Navigate to your new workspace in Azure Portal. Find your Workspace ID and Primary key in your workspace under Agents management. In `values.yaml`, under `fluentBit`, update the `outputPlugin` to `azure` and keys `workspaceId` and `primaryKey` with the corresponding values from Azure Portal (without quotes). Alternatively, you may replace entire output section in `fluentbit-configmap.yaml` as you would for any other output plugin.

3. Run through steps 2-5 above.

4. Once you run OSM with Fluent Bit enabled, logs will populate under the Logs > Custom Logs section in your Log Analytics workspace. There, you may run the following query to view most recent logs first:
    ```
    fluentbit_CL
    | order by TimeGenerated desc
    ```
5. Refine your log results on a specific deployment of the OSM controller pod:
    ```
    | where controller_pod_name_s == "<desired osm controller pod name>"
    ```

Once logs have been sent to Log Analytics, they can also be consumed by Application Insights as follows:
1. [Create a Workspace-based Application Insights instance](https://docs.microsoft.com/en-us/azure/azure-monitor/app/create-workspace-resource).

2. Navigate to your instance in Azure Portal. Go to the Logs section. Run this query to ensure that logs are being picked up from Log Analytics:
    ```
    workspace("<your-log-analytics-workspace-name>").fluentbit_CL
    ```

You can now interact with your logs in either of these instances.

*Note: Fluent Bit is not currently supported on OpenShift.*

### Configuring Outbound Proxy Support for Fluent Bit
You may require outbound proxy support if your egress traffic is configured to go through a proxy server. There are two ways to enable this.

If you have already built OSM with the MeshConfig changes above, you can simply enable proxy support using the OSM CLI, replacing your values in the command below:
```
osm install --set=osm.enableFluentbit=true,osm.fluentBit.enableProxySupport=true,osm.fluentBit.httpProxy=<http-proxy-host:port>,osm.fluentBit.httpsProxy=<https-proxy-host:port>
```

Alternatively, you may change the values in the Helm chart by updating the following in `values.yaml`:
1. Change `enableProxySupport` to `true`

1. Update the httpProxy and httpsProxy values to `"http://<host>:<port>"`. If your proxy server requires basic authentication, you may include its username and password as: `http://<username>:<password>@<host>:<port>`

1. For these changes to take effect, run:
    ```console
    make build-osm
    ```

1. Install OSM with Fluent Bit enabled:
    ```console
    osm install --set=osm.enableFluentbit=true
    ```
> NOTE: Ensure that the [Fluent Bit image tag](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/values.yaml) is `1.6.4` or greater as it is required for this feature.
