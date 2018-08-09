***Base documentation for Prometheus Monitoring***

[Basic documentation can be found in the ICP Knowledge center](https://www.ibm.com/support/knowledgecenter/SSBS6K_2.1.0/manage_metrics/monitoring_service.html)

***Further information***

Prometheus is a systems and service monitoring system. It collects metrics from configured targets at given intervals, evaluates rule expressions, displays the results, and can trigger alerts if some condition is observed to be true. 
https://prometheus.io/

Both Prometheus and Kubernetes are developed under the Cloud Native Computing Foundation (CNCF) project.

Prometheus has several components for the collection of Time Series Data, an Alert Manager and a central Prometheus Server which scrapes  and stores the data. The data is visualized using a Grafana instance.

The Prometheus and Grafana stacks are deployed by default when ICP is installed, unless the value of **disabled_management_services** is changed from default in [config.yaml](https://www.ibm.com/support/knowledgecenter/SSBS6K_2.1.0/installing/config_yaml.html)

If you've configured ICP to use a [management node](https://www.ibm.com/support/knowledgecenter/SSBS6K_2.1.0/installing/hosts.html) then the stack will be deployed on that node. Otherwise they will co-exist with the rest of the workloads.

Within the ICP GUI, you can see the Prometheus/Grafana Deployments, Services and ConfigMaps by selecting **main menu**->**Platform** and then the component type you are interested in.
Filter by *monitoring* (and *alert* too in the case of ConfigMaps) to see the components that are dedicated to monitoring ICP.

![Deployments](images/deployments.png)

You can also see the components using the command line interface:

```
$kubectl get configmap --namespace=kube-system | grep "alert\|monitor"
alert-rules                           1         6d
alertmanager-router-nginx-config      1         6d
monitoring-grafana-config             1         6d
monitoring-grafana-dashboard-config   3         6d
monitoring-prometheus                 1         6d
monitoring-prometheus-alertmanager    1         6d
monitoring-router-entry-config        1         6d
```


***Creating alerts in Prometheus***

In order to generate alerts and notify people that an incident has occured, two items must be configured in Prometheus
1. The notification targets (i.e. who do we want to inform) and
2. The rules (i.e. what do we want to inform them of)

Definition of notification targets is explained in the [Integration](https://github.com/ibm-cloud-architecture/CSMO-ICP/tree/master/integration) section.
Basic documentation on the creation of Prometheus rules can be found in the [Prometheus documentation](https://prometheus.io/docs/alerting/rules/).

****Loading new rules into Prometheus****

If you have ICP 2.1.0.2 or higher, which uses Prometheus 2.x, use the [instructions here](https://github.com/ibm-cloud-architecture/CSMO-ICP/tree/master/prometheus/alerts_prometheus2.x).
If you have ICP 2.1.0.1 or lower, which uses Prometheus 1.x, use the [instructions here](https://github.com/ibm-cloud-architecture/CSMO-ICP/blob/master/prometheus/alerts_prometheus1.x.md).


****Viewing alerts in Prometheus****

Alerts that have actively *fired* can be seen in `https://<master_ip>:8443/alertmanager` or via *Menu*->*Platform*->*Alerting*
To see the list of rules you must access the Prometheus dashboard through a proxy.

1. Run the command `kubectl get pods --namespace=kube-system -l app=monitoring-prometheus,component=prometheus -o jsonpath="{.items[0].metadata.name}"` to get the name of the Prometheus server pod
2. Run the command `kubectl port-forward --namespace=kube-system <pod_name> 9090` to activate the proxy
3. Open `http://localhost:9090/alerts` to access Prometheus from your browser and see the available alerts (both fired and un-fired)


**Trouble shooting**

Your ICP installation might end up requiring more Prometheus resources than you initially have allocated. This may result in the Prometheus pod crashing as it tries to over-allocate memeory. [This document](https://github.com/ibm-cloud-architecture/CSMO-ICP/blob/master/prometheus/update_resources.md) will guide you in allocating more resources.
