# Base set of alerts for ICP 3.1.2

Most of the rules are based on the [Prometheus Operator](https://github.com/coreos/prometheus-operator) rules modified to work with IBM Cloud Private and recommendations from [etcd.io](https://github.com/etcd-io). 

Use the following commands to deploy the provided alert rules on ICP 3.1.2:

```
wget https://github.com/ibm-cloud-architecture/CSMO-ICP/raw/master/prometheus/alerts_icp_3.1.2/alerts312.tar.gz
tar xvfz alerts312.tar.gz
cd alerts312
for i in *.yml;do kubectl apply -f $i;done
```

Verify the alert rules were successfully imported:

```
$ kubectl get alertrules
NAME                                        AGE
api-server-down                             1h
api-server-latency                          23h
controller-manager-down                     1h
elasticsearch-cluster-health                32d
etcd-high-commit-durations                  1h
etcd-high-fsync-durations                   1h
etcd-high-number-of-failed-proposals        1h
etcd-high-number-of-leader-changes          1h
etcd-insufficient-members                   1h
etcd-no-leader                              1h
failed-jobs                                 32d
high-cpu-usage                              32d
host-cpu-utilization                        1h
host-high-memory-load                       1h
kubelet-down                                1h
kubelet-too-many-pods                       1h
many-nodes-not-ready                        1h
monitoring-heartbeat                        1h
monitoring-target-down                      1h
node-disk-pressure                          1h
node-exporter-down                          1h
node-memory-pressure                        1h
node-memory-usage                           32d
node-not-ready                              1h
node-out-of-disk                            1h
node-swap-usage                             1h
pod-frequently-restarting                   1h
pods-restarting                             32d
pods-terminated                             32d
predictive-host-disk-space                  1h
prometheus-config-reload-failed             1h
prometheus-error-sending-alerts             1h
prometheus-not-connected-to-alertmanagers   1h
prometheus-notification-queue-full          1h
too-many-open-file-descriptors              1h
```

The alert rules can be edited using:

```
kubectl edit alertrules/<rule_name>
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
More detailed article about integration of ICP Prometheus AlertManager with IBM Netcool Operations Insight, can be found [here](https://apps.na.collabserv.com/blogs/c83b42e5-2186-42f1-b498-2871621e2984/entry/ICP_Forwarding_alerts_from_Prometheus_Alertmanager_to_Omnibus?lang=en_us) (IBM Internal).

[Here](https://apps.na.collabserv.com/blogs/c83b42e5-2186-42f1-b498-2871621e2984/entry/ICP_Deploy_NOI_in_30_minutes?lang=en_us) (IBM Internal) you can find instructions, how to deploy NOI in IBM Cloud Private.

Use the following command to modify AlertManager ConfigMap in ICP:

```
kubectl edit cm monitoring-prometheus-alertmanager -n kube-system
```

The table below lists proposed base set of alerts:

| Alert Name | Severity | Summary | Message | Action |
|-----------------------------------------|----------|-----------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ICPMonitoringTargetDown | warning | Targets are down | {{ $value }}% or more of {{ $labels.job }} targets are down. | Verify which scraping targets are down. Check the following URL: https://<icp_console>:8443/prometheus/targets to identify which instances are not available for scraping (metrics polling). More information about scraping jobs and instances: https://prometheus.io/docs/concepts/jobs_instances |
| ICPMonitoringHeartbeat | none | Alerting Heartbeat | This is a Heartbeat event meant to ensure that the entire Alerting pipeline is functional. | This notification is excpected every 10 minutes. Alert should be raised by the Event Manager (like Netcool OMNIBus) if notification is missing. In that case (alert because of missing heartbeat notification), verify that Promethus and AlertManager pods are running, network connectivity within alerting pipeline Prometheus->AlertManager->Netcool is functional and if there were any changes in the Prometheus or Netcool configuration. Check logs of Prometheus and Alertmanager Pods using: kubectl logs <prometheus_pod> -n kube-system, kubectl logs <prometheus-alertmanager_pod> -n kube-system |
| ICPTooManyOpenFileDescriptors | critical | Too many open file descriptors for the monitoring target | {{ $labels.job }}: {{ $labels.namespace }}/{{ $labels.pod }} ({{ $labels.instance }}) is using {{ $value }}% of the available file/socket descriptors. | This alert allows to identify which Pod instance is close to the open file descriptors limit. File descriptor exhaustion leads to an instance being unavailable, requiring a restart to function properly again. Depending on the root cause, the solution may be an increasing of file descriptor limit, increasing the number of instances (scale up), application code change etc. |
| ICPPodFrequentlyRestarting | warning | Pod is restarting frequently | Pod {{$labels.namespaces}}/{{$labels.pod}} is was restarted {{$value}} times within the last hour | A Pod is restarting several times an hour. Verify Pod logs using ICP Kibana console. If the Pod name is known, use the following search query in the Kibana console: kubernetes.pod: <pod_name> If Pod is a part of k8s Deployment you can also use ICP Management console Workload->Deployment->Logs |
| ICPApiserverDown | critical | API server unreachable | Prometheus failed to scrape API server(s) or all API servers have disappeared from service discovery. | Prometheus failed to scrape the API server, or API server have disappeared from service discovery. Verify status of the k8s apiserver using the following instructions: https://stackoverflow.com/a/48669203. You can also use the following command locally on ICP master node to check if the apiserver properly exposes Prometheus metrics: curl http://localhost:8888/metrics. Check k8s apiserver logs in the ICP Kibana console using the following search query: kubernetes.container_name: apiserver |
| ICPApiServerLatency | warning | Kubernetes apiserver latency is high | 99th percentile Latency for {{ $labels.verb }} requests to the kube-apiserver is higher than 1s. | Alert is raised when 1% of the API requests to the k8s apiserver in the last 10 minutes took more than 1 second. Check the apiserver logs to identify slow requests. Tip: Kubernets apiserver logs are ingested by the ICP ElasticStack. Annotate response time filed in the apiserver logs to easily search for slow apiserver requests ICP Kibana. |
| ICPNodeNotReady | warning | Node status is NotReady | The Kubelet on {{ $labels.node }} has not checked in with the API or has set itself to NotReady for more than an hour | List the status of the nodes using: kubectl get nodes and kubectl describe node <node_name>. Look for the following sections: conditions, capacity and allocatable. Check the node resource utilization (cpu, memory, storage) for the problematic node on the ICP Grafana dashboard: Kubernetes Cluster Monitoring. Logon via ssh to the problematic node and check kubelet logs using: journalctl -u kubelet |
| ICPManyNodesNotReady | critical | Many Kubernetes nodes are Not Ready | {{ $value }} Kubernetes nodes (more than 10% are in the NotReady state). | More than 10% of the listed number of Kubernetes nodes are NotReady. List status of the nodes: kubectl get nodes and kubectl describe node <node_name>. Look for the following sections: conditions, capacity and allocatable. Check node resource utilization for the problematic node on the ICP Grafana dashboard: Kubernetes Cluster Monitoring. Logon via ssh to the problematic node and check kubelet logs using: journalctl -u kubelet. |
| ICPKubeletDown | warning | Many Kubelets cannot be scraped | Prometheus failed to scrape {{ $value }}% of kubelets. | Identify which kubelet cannot be scraped by Prometheus using the following PromQL: up{job="kubernetes-nodes"} in the Prometheus console https://<icp_console>:8443/prometheus or check the status of the Prometheus job: kubernetes-nodes using: https://<icp_console>:8443/prometheus/targets. Check the status of the kubelet service on the worker node using command systemctl status kubelet. Check kubelet logs using: journalctl -u kubelet. |
| ICPKubeletDown | critical | Many Kubelets cannot be scraped | Prometheus failed to scrape {{ $value }}% of kubelets or all Kubelets have disappeared from service discovery. | Identify which kubelect cannot be scraped by Prometheus using the following PromQL: up{job="kubernetes-nodes"} in the Prometheus console https://<icp_console>:8443/prometheus or check the status of the Prometheus job: kubernetes-nodes using: https://<icp_console>:8443/prometheus/targets. Check the status of the kubelet service on the worker node using command systemctl status kubelet. Check kubelet logs using: journalctl -u kubelet. |
| ICPKubeletTooManyPods | warning | Kubelet is close to pod limit | Kubelet {{$labels.instance}} is running {{$value}} pods close to the limit of 110 | Add worker nodes in order to distribute Pods in your ICP cluster or reduce the number of running pods. |
| ICPNodeExporterDown | warning | node-exporter cannot be scraped | Prometheus could not scrape a node-exporter for more than 10m or node-exporters have disappeared from discovery. | Prometheus failed to scrape the node-exporter on one on more worker nodes, or node exporter have disappeared from service discovery. Verify status of the monitoring-prometheus-nodeexporter DaemontSet using kubectl get ds monitoring-prometheus-nodeexporter -n kube-system. Node exporter pod should run on every worker node. Check the monitoring-prometheus-nodeexporter DaemonSet on the ICP console: https://<ICP_console>:8443/console/workloads/daemonsets/kube-system/monitoring-prometheus-nodeexporter. Check logs of the problematic node-exporter pod using kubectl logs <node_exporter-pod-name> -n kube-system |
| ICPNodeOutOfDisk | critical | Node ran out of disk space. | {{ $labels.node }} has run out of disk space. | Kubelet reported that there node is running out of disk space. Check the disk space on the node reported in the alert. Alert based on the node condition: OutOfDisk. Verify with kubectl describe node <node_ip> section: Conditions. |
| ICPNodeMemoryPressure | warning | Node is under memory pressure. | {{ $labels.node }} is under memory pressure. | Kubelet reported that the node is running out of memory. Check the available memory on the node instance reported in the alert message. Alert based on the node condidtion: MemoryPressure. Verify with kubectl describe node <node_ip> section Conditions. |
| ICPNodeDiskPressure | warning | Node is under disk pressure. | {{ $labels.node }} is under disk pressure. | Kubelet reported that the node is running out of memory. Check the available memory on the node instance reported in the alert message. Alert based on the node condidtion: MemoryPressure. Verify with kubectl describe node <node_ip> section Conditions. |
| ICPHostCPUUtilisation | warning | CPU Utilisation Alert | High CPU utilisation detected for instance {{ $labels.instance_id }} tagged as: {{ $labels.instance_name_tag }} the utilisation is currently: {{ $value }}% | High CPU utilization on the ICP cluster host. Use the ICP Grafana dashboard: Kubernetes Cluster monitoring to identify which process or Pod is responsible for high CPU utilization and how CPU utilization changed in time for host, Pods and containers. |
| ICPPreditciveHostDiskSpace | warning | Predictive Disk Space Utilisation Alert | Based on recent sampling the disk is likely to will fill on volume {{ $labels.mountpoint }} within the next 4 hours for instace: {{ $labels.instance_id }} tagged as: {{ $labels.instance_name_tag }} | If disks keep filling up at the current pace they will run out of free space within the next hours. Alert message identifies the node instance and the mount point. Use Linux system commands to identify growing directory/files. |
| ICPHostHighMemeoryLoad | warning | Memory utilization Alert | Memory of a host is almost full for instance {{ $labels.instance_id }} | High memory utilization on the ICP cluster host. Use ICP Grafana dashboard: Kubernetes Cluster monitoring to identify which process or Pod is responsible for high memory utilization and how memory utilization changed in time for host, Pods and containers. |
| ICPNodeSwapUsage | warning | {{$labels.instance}}: Swap usage detected | {{$labels.instance}}: Swap usage usage is above 75% (current value is: {{ $value }}) | Swap useage is higher than 75% which means that host server needs more RAM, there are too many containers running on the node or there is a memory leak in the running code. |
| PrometheusConfigReloadFailed | warning | Prometheus configuration reload has failed | Reloading Prometheus' configuration has failed for {{ $labels.namespace }}/{{ $labels.pod}}. | Check the Prometheus pod logs using kubectl logs <prometheus_pod_name> -n kube-system. Most probably, there is a syntax error in the Prometheus ConfigMap. |
| PrometheusNotificationQueueRunningFull | warning | Prometheus alert notification queue is running full | Prometheus alert notification queue is running full for {{$labels.namespace}}/{{$labels.pod}} | Prometheus is generating more alerts than it can send to Alertmanagers in time. Check the status of the Prometheus AlertManager Pod. Check the logs of the Prometheus Pod. Check recent Prometheus alert configuration changes. |
| PrometheusErrorSendingAlerts | warning | Errors while sending alert from Prometheus | Errors while sending alerts from Prometheus {{$labels.namespace}}/{{$labels.pod}} to Alertmanager {{$labels.Alertmanager}} | Check the status of the AlertManager Pod. Verify configured AlertManagers: https://<icp_console>:8443/prometheus/status. Verify AlertManager status and configuration: https://<icp_console>:8443/alertmanager/#/status. Check the Prometheus pod logs using kubectl logs <prometheus_pod_name> -n kube-system. |
| PrometheusErrorSendingAlerts | critical | Errors while sending alert from Prometheus | Errors while sending alerts from Prometheus {{$labels.namespace}}/{{$labels.pod}} to Alertmanager {{$labels.Alertmanager}} | Check the status of the AlertManager Pod. Verify configured AlertManagers: https://<icp_console>:8443/prometheus/status. Verify AlertManager status and configuration: https://<icp_console>:8443/alertmanager/#/status. Check the Prometheus pod logs using kubectl logs <prometheus_pod_name> -n kube-system. |
| PrometheusNotConnectedToAlertmanagers | warning | Prometheus is not connected to any Alertmanagers | Prometheus {{ $labels.namespace }}/{{ $labels.pod}} is not connected to any Alertmanagers | Check the status of the AlertManager Pod. Verify configured AlertManagers: https://<icp_console>:8443/prometheus/status. Verify AlertManager status and configuration: https://<icp_console>:8443/alertmanager/#/status. Check the Prometheus pod logs using kubectl logs <prometheus_pod_name> -n kube-system. |
| ICPetcdInsufficientMembers | critical | etcd cluster insufficient members | If one more etcd member goes down the cluster will be unavailable | Applicable only for etcd clusters. Alert is raised if another failed member of the etcd cluster will result in an unavailable etcd cluster. |
| ICPetcdNoLeader | critical | etcd member has no leader | etcd member {{ $labels.instance }} has no leader | Applicable only for etcd clusters |
| ICPetcdHighNumberOfLeaderChanges | warning | a high number of leader changes within the etcd cluster are happening | etcd instance {{ $labels.instance }} has seen {{ $value }} leader changes within the last hour | Applicable only for etcd clusters. The alert is fired if there are lots of leader changes |
| ICPetcdHighNumberOfFailedProposals | warning | a high number of proposals within the etcd cluster are failing | etcd instance {{ $labels.instance }} has seen {{ $value }} proposal failures within the last hour | Writes and configuration changes sent to etcd are called proposals. The raft protocol ensures that the proposals are applied correctly to the cluster. Rate of failed proposals should be low. Check etcd logs. |
| ICPetcdHighFsyncDurations | warning | high fsync durations | etcd instance {{ $labels.instance }} fync durations are high | Disk performance is the major driver for etcd server performance as proposals must be written to disk  and fsync’ed, before followers can acknowledge a proposal from the leader. Slow storage may highly degrade etcd performance. |
| ICPetcdHighCommitDurations | warning | high commit durations | etcd instance {{ $labels.instance }} commit durations are high | Disk performance is the major driver for etcd server performance as proposals must be written to disk  and fsync’ed, before followers can acknowledge a proposal from the leader. Slow storage may highly degrade etcd performance. |