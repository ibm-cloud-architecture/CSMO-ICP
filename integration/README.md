**Integrations**
Implementing outgoing alerts from Prometheus involve two components:
1. The AlarmServer component in Prometheus
2. The recieving component

***Configuring the AlarmServer***
The most important definitions of the AlarmServer are the recievers of alerts and the routing of alerts.
[Prometheus AlertManager base documentation](https://prometheus.io/docs/alerting/configuration/)

In this example, we will be configuring the AlertServer to forward a webhook when an alert fires.
In order to test the success of the webhook, you need to create a suitable target. A website such as [RequestBin](https://requestb.in/) or even running the command `nc -l <port>` on your computer will be enough.

1. Edit the AlertManager ConfigMap using the command `kubectl edit cm --namespace=kube-system monitoring-prometheus-alertmanager`
2. Add the definition lines just after the default receiver definition:

old file:
```
    receivers:
      - name: default-receiver
    route:
```
new file:
```
    receivers:
      - name: default-receiver
        webhook_configs:
         - url : <target_url>
           send_resolved : true
    route:
 
```
This will ensure that all events will be sent to `target_url`.
3. By default, Prometheus will group similar events together so that you will get less messages and each message will contain multiple events.
If you want to avoid this, add a *group_by* configuration to the routing definition.

old file:
```
    route:
      group_wait: 10s
     
 ```
 new file:
 ```
     route:
      group_by: ['alertname','instance','kubernetes_namespace']
      group_wait: 10s
     
 ```
 
