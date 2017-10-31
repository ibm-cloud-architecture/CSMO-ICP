
Implementing outgoing alerts from Prometheus involve two components:
1. The AlarmServer component in Prometheus
2. The receiving component

***Configuring the AlertManager***

The most important definitions of the AlarmServer are the receivers of alerts and the routing of alerts.
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
 4. Reload the configuration by browsing to `https://<master_ip>:8443/alertmanager/reload`
 5. Once the configuration has been reloaded and alerts fired, you will see webhooks sent to your target.
 
 ***Configuring NOI as a webhook reciever***
 
 By using the [Netcool MessageBus Probe](https://www.ibm.com/support/knowledgecenter/en/SSSHTQ/omnibus/probes/message_bus/wip/concept/messbuspr_intro.html), you can receive Prometheus AlertServer events.
 1. [Download](http://www-01.ibm.com/support/docview.wss?uid=swg21970413) and [install](https://www.ibm.com/support/knowledgecenter/en/SSSHTQ/omnibus/probes/common/topicref/pro_install_intro_messbuspr.html) the MessageBus Probe.
 2. The files in the [messagebus directory](https://github.com/ibm-cloud-architecture/CSMO-ICP/tree/master/integration/messagebus) are the necessary configuration files for the probe. Add or merge them to your existing environment:
 * `httpTransport_prom.properties` should be placed in the $OMNIHOME/java/conf directory. Edit the file to set the port you wish to listen on (this will affect the webhook configuration in the AlertServer)
 * `message_bus_prom.props` and `message_bus_prom.rules` should be placed in the $OMNIHOME/probes/<arch>/ directory. Edit the files to suite your environment's settings.
 3. Start the Messagebus probe and verify that it is listening on the correct port
 4. Modify the AlertManager ConfigMap so that it will send alerts to the probe
 
