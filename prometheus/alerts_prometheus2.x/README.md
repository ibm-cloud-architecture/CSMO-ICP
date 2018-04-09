# Base set of alerts for ICP 2.1.0.2

Alerts are defined in Prometheus 2.x format (IBM Cloud Private 2.1.0.2 or later).

Use the following command to replace existing alert definitions in Prometheus with provided alert rules:

```
kubectl replace -f alert-rules.yaml  
```

In order to add selected alert rules to existing configuration, edit `alert-rules` ConfigMap in `kube-system` namespace and copy selected rules into `data:` section.

```
kubectl edit cm alert-rules -n kube-system
```
Example alert rule:

```
- alert: ICPPreditciveHostDiskSpace
        expr: predict_linear(node_filesystem_free{mountpoint="/"}[4h], 4 * 3600) < 0
        for: 30m
        labels:
          severity: warning
        annotations:
          description: 'Based on recent sampling, the disk is likely to will fill on volume
            {{ $labels.mountpoint }} within the next 4 hours for instace: {{ $labels.instance_id
            }} tagged as: {{ $labels.instance_name_tag }}'
          summary: Predictive Disk Space Utilisation Alert
```

       
Example configuration of Prometheus AlertManager listed below, modifies behaviour of ICPMonitoringHeartbeat rule, to send heartbeat events to NOI Message Bus probe every 10 minutes:

```
apiVersion: v1
data:
  alertmanager.yml: |-
    global:
    receivers:
      - name: default-receiver
        webhook_configs:
        - url: 'http://<msgbus_probe_ip>:<msgbus_probe_port>/probe/webhook/prometheus'
          send_resolved: true
    route:
      group_by: ['alertname','instance','kubernetes_namespace','pod','container']
      group_wait: 10s
      group_interval: 5m
      receiver: default-receiver
      repeat_interval: 3h
      routes:
      - receiver: default-receiver
        match:
          alertname: ICPMonitoringHeartbeat
        repeat_interval: 5m
```

Use the following command to modify AlertManager ConfigMap in ICP:

```
kubectl edit cm monitoring-prometheus-alertmanager -n kube-system
```

The table below lists proposed base set of alerts:


| Alert Name | Category | Severity | Summary | Message |
|----------------------------------------|------------------------|----------|---------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ICPMonitoringTargetDown | general | warning | Targets are down | {{ $value }}% or more of {{ $labels.job }} targets are down. |
| ICPMonitoringHeartbeat | general | none | Alerting Heartbeat | This is a Heartbeat event meant to ensure that the entire Alerting pipeline is functional. |
| ICPTooManyOpenFile Descriptors | general | critical | Too many open file descriptors for the monitoring target | {{ $labels.job }}: {{ $labels.namespace }}/{{ $labels.pod }} ({{ $labels.instance }}) is using {{ $value }}% of the available file/socket descriptors. |
| ICPFdExhaustionClose | general | warning | File descriptors soon exhausted for the monitoring target | {{ $labels.job }}: {{ $labels.namespace }}/{{ $labels.pod }} ({{ $labels.instance }}) instance will exhaust in file/socket descriptors soon |
| ICPFdExhaustionClose | general | critical | File descriptors soon exhausted for the monitoring target | {{ $labels.job }}: {{ $labels.namespace }}/{{ $labels.pod }} ({{ $labels.instance }}) instance will exhaust in file/socket descriptors soon |
| ICPPodFrequentlyRestarting | general | warning | Pod is restarting frequently | Pod {{$labels.namespaces}}/{{$labels.pod}} is was restarted {{$value}} times within the last hour |
| ICPApiserverDown | kube-apiserver | critical | API server unreachable | Prometheus failed to scrape API server(s) or all API servers have disappeared from service discovery. |
| ICPApiServerLatency | kube-apiserver | warning | Kubernetes apiserver latency is high | 99th percentile Latency for {{ $labels.verb }} requests to the kube-apiserver is higher than 1s. |
| ICPControllerManagerDown | kube-controler-manager | critical | Controller manager is down | There is no running ICP controller manager. Deployments and replication controllers are not making progress. |
| ICPNodeNotReady | kubelet | warning | Node status is NotReady | The Kubelet on {{ $labels.node }} has not checked in with the API or has set itself to NotReady for more than an hour |
| ICPManyNodesNotReady | kubelet | critical | Many Kubernetes nodes are Not Ready | {{ $value }} Kubernetes nodes (more than 10% are in the NotReady state). |
| ICPKubeletDown | kubelet | warning | Many Kubelets cannot be scraped | Prometheus failed to scrape {{ $value }}% of kubelets. |
| ICPKubeletDown | kubelet | critical | Many Kubelets cannot be scraped | Prometheus failed to scrape {{ $value }}% of kubelets or all Kubelets have disappeared from service discovery. |
| ICPKubeletTooManyPods | kubelet | warning | Kubelet is close to pod limit | Kubelet {{$labels.instance}} is running {{$value}} pods close to the limit of 110 |
| ICPNodeExporterDown | node | warning | node-exporter cannot be scraped | Prometheus could not scrape a node-exporter for more than 10m or node-exporters have disappeared from discovery. |
| ICPNodeOutOfDisk | node | critical | Node ran out of disk space. | {{ $labels.node }} has run out of disk space. |
| ICPNodeMemoryPressure | node | warning | Node is under memory pressure. | {{ $labels.node }} is under memory pressure. |
| ICPNodeDiskPressure | node | warning | Node is under disk pressure. | {{ $labels.node }} is under disk pressure. |
| ICPHostCPUUtilisation | node | warning | CPU Utilisation Alert | High CPU utilisation detected for instance {{ $labels.instance_id }} tagged as: {{ $labels.instance_name_tag }} the utilisation is currently: {{ $value }}% |
| ICPPreditciveHostDiskSpace | node | warning | Predictive Disk Space Utilisation Alert | Based on recent sampling the disk is likely to will fill on volume {{ $labels.mountpoint }} within the next 4 hours for instace: {{ $labels.instance_id }} tagged as: {{ $labels.instance_name_tag }} |
| ICPHostHighMemeoryLoad | node | warning | Memory utilization Alert | Memory of a host is almost full for instance {{ $labels.instance_id }} |
| ICPNodeSwapUsage | node | warning | {{$labels.instance}}: Swap usage detected | {{$labels.instance}}: Swap usage usage is above 75% (current value is: {{ $value }}) |
| PrometheusConfigReload Failed | prometheus | warning | Prometheus configuration reload has failed | Reloading Prometheus' configuration has failed for {{ $labels.namespace }}/{{ $labels.pod}}. |
| PrometheusIngestionThrottling | prometheus | warning | Prometheus is (or borderline) throttling ingestion of metrics | Prometheus cannot persist chunks to disk fast enough. It's urgency value is {{$value}}. |
| PrometheusNotificationQueue RunningFull | prometheus | warning | Prometheus alert notification queue is running full | Prometheus alert notification queue is running full for {{$labels.namespace}}/{{$labels.pod}} |
| PrometheusErrorSendingAlerts | prometheus | warning | Errors while sending alert from Prometheus | Errors while sending alerts from Prometheus {{$labels.namespace}}/{{$labels.pod}} to Alertmanager {{$labels.Alertmanager}} |
| PrometheusErrorSendingAlerts | prometheus | critical | Errors while sending alert from Prometheus | Errors while sending alerts from Prometheus {{$labels.namespace}}/{{$labels.pod}} to Alertmanager {{$labels.Alertmanager}} |
| PrometheusNotConnectedTo Alertmanagers | prometheus | warning | Prometheus is not connected to any Alertmanagers | Prometheus {{ $labels.namespace }}/{{ $labels.pod}} is not connected to any Alertmanagers |