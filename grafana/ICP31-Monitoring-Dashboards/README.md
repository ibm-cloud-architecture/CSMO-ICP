# ICP Monitoring Dashboards

A set of monitoring dashboards for IBM Cloud Private. Dashboards and recording rules for Prometheus are based on [Prometheus Monitoring Mixin for Kubernetes](https://github.com/kubernetes-monitoring/kubernetes-mixin/). Tested on ICP 3.1.

Click on the links below to see sample dashboard snaphots.

- [Cluster](https://snapshot.raintank.io/dashboard/snapshot/LuONx1P9OjnSTJh5tmKRtcFCDcKcmq3c)
- [Namespace](https://snapshot.raintank.io/dashboard/snapshot/pKjVTrS5E27ZWMRsDVy9W63o6Sc6J0JX)
- [Node](https://snapshot.raintank.io/dashboard/snapshot/OnK13Bcl5NkhuryyR9FlY60DK02mLQsr)
- [Pod](https://snapshot.raintank.io/dashboard/snapshot/vqpRSVZMLmT3K2v3durDycYqFW9g25eQ)
- [StatefulSet](https://snapshot.raintank.io/dashboard/snapshot/4yvMJaqnLYKn1i3CV829r0PdhypeaaQL)
- [Cluster (USE method)](https://snapshot.raintank.io/dashboard/snapshot/R0NT5SkyBe6loSw0E6PJF1qXDUyMENqv)
- [Node (USE method)](https://snapshot.raintank.io/dashboard/snapshot/TOCfdyuxoQ8xp5V64v4HtKO8AM3ee1go)

## Configuration

### Import recording rules
Add the following [recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/) to the `monitoring-prometheus-alertrules` ConfigMap: [recording-rules.yml](recording-rules.yml). 

Example ConfigMap: 
[monitoring-prometheus-alertrules-cm-rec-rules.yml](monitoring-prometheus-alertrules-cm-rec-rules.yml)

After a couple of minutes, verify the new recording rules have been properly imported by Prometheus. Use Prometheus UI: `https://<icp_ip>:8443/prometheus/rules` and scroll down to recording rules.
![](rules.png)

### Import Grafana dashboards
[Import](http://docs.grafana.org/reference/export_import/#importing-a-dashboard) the following dashboards to the ICP Grafana.

- [Cluster](ICP31-Grafana/K8s_Cluster.json)
- [Namespace](ICP31-Grafana/K8s_Namespace.json)
- [Node](ICP31-Grafana/K8s_Node.json)
- [Pod](ICP31-Grafana/K8s_Pod.json)
- [StatefulSet](ICP31-Grafana/K8s_Node.json)
- [Cluster (USE Method)](ICP31-Grafana/K8s_USE_Method_Cluster.json)
- [Node (USE Method)](ICP31-Grafana/K8s_USE_Method_Node.json)