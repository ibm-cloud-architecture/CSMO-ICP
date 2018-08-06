# How to generate SNMP traps from Prometheus alerts

Prometheus Alertmanager natively supports a couple of different event destinations like Slack, email, PagerDuty, HipChat and [others](https://prometheus.io/docs/alerting/configuration/). For notification mechanisms not natively supported by the Alertmanager, the [webhook receiver](https://prometheus.io/docs/alerting/configuration/#webhook_config) allows for integration.

SNMP traps are still a popular method of sending events informing about issues that needs to be reported.
The instructions below allows to quickly setup a simple itegration mechanism which will allow to generate SNMP traps for selected Prometheus alerts using open source [Prometheus WebHook to SNMP-trap forwarder](https://github.com/chrusty/prometheus_webhook_snmptrapper).

1). Copy `prometheus_webhook_snmptrapper` to a Linux server accesible from your Prometheus AlertManager instance (lets name it `linux1`). It is a single binary file which can be compiled from Golang [sources](https://github.com/chrusty/prometheus_webhook_snmptrapper) available on GitHub. You can also download the compiled Linux 64-bit version from [here](https://ibm.box.com/s/sn45d6n2mviwwi3iusmzpxa8fzd75l2p).

2). Install `net-snmp` and `snmptrapd` packages. On Ubuntu run:

```
sudo apt-get install net-snmp snmptrapd
```
3). In the AlertManager configuration file, define the webhook receiver address in section `receivers`:

```
    receivers:
      - name: default-receiver
        webhook_configs:
        - url: 'http://<linux1_ip>:9999'
```
Port `9999` is port of `prometheus_webhook_snmptrapper` webhook server (make sure if it not already taken by other process) and can be changed using input parameter of `prometheus_webhook_snmptrapper`. 

If your Prometheus instance is installed on IBM Cloud Private, then edit `monitoring-prometheus-alertmanager` ConfigMap in `kube-system` namespace.

```
kubectl edit cm monitoring-prometheus-alertmanager -n kube-system
```
Use provided [configuration](monitoring-prometheus-alertmanager.yml) as an example.
The example configuration was tested on the alerts defined in the [base set of monitoring alerts for ICP](https://github.com/ibm-cloud-architecture/CSMO-ICP/tree/master/prometheus/alerts_prometheus2.x).

4). Copy the following MIB files: [PROMETHEUS-TRAPPER-MIB.txt](https://raw.githubusercontent.com/chrusty/prometheus_webhook_snmptrapper/master/PROMETHEUS-TRAPPER-MIB.txt) and [SNMPv2-SNI](https://raw.githubusercontent.com/hardaker/net-snmp/master/mibs/SNMPv2-SMI.txt) to your MIB directory ex. `/usr/share/snmp/mibs`. You can find you MIBDIR using the following command:

```
snmptranslate -Dinit_mib .1.3 2>&1 |grep MIBDIR
```

5). Start `snmptrapd` using:

```
snmptrapd -n -Ls d -m PROMETHEUS-TRAPPER-MIB
```

It will start snmptrapd as a daemon and SNMP traps will be logged to local syslog file (`/var/log/syslog` by default on Ubuntu).

6). Start `prometheus_webhook_snmptrapper`

```
./prometheus_webhook_snmptrapper --webhookaddress 0.0.0.0:9999 > prom_snmp.log 2>&1  &
```


7). Verify that some Prometheus alerts has been generated after you started `prometheus_webhook_snmptrapper` and check the contents of syslog and `prom_snmp.log`.

The contents of `prom_snmp.log` should be similar to the one below:

```
time="2018-07-05T09:47:53-05:00" level=info msg="Preparing the alerts channel" logger=main
time="2018-07-05T09:47:53-05:00" level=info msg="Starting the SNMP trapper" address="127.0.0.1:162" logger=SNMP-trapper
time="2018-07-05T09:47:53-05:00" level=info msg="Starting the Webhook server" address="0.0.0.0:9999" logger=Webhook-server
time="2018-07-05T09:51:05-05:00" level=debug msg="Received a valid webhook alert" logger=Webhook-server payload="{\"receiver\":\"default-receiver\",\"status\":\"firing\",\"alerts\":[{\"status\":\"firing\",\"labels\":{\"alertname\":\"ICPMonitoringHeartbeat\",\"severity\":\"none\"},\"annotations\":{\"description\":\"This is a Hearbeat event meant to ensure that the entire Alerting pipeline is functional.\",\"summary\":\"Alerting Heartbeat\"},\"startsAt\":\"2018-06-26T11:49:29.035689542Z\",\"endsAt\":\"0001-01-01T00:00:00Z\",\"generatorURL\":\"http://monitoring-prometheus-77d664c568-jzqnp:9090/graph?g0.expr=vector%281%29\\u0026g0.tab=1\"}],\"groupLabels\":{\"alertname\":\"ICPMonitoringHeartbeat\"},\"commonLabels\":{\"alertname\":\"ICPMonitoringHeartbeat\",\"severity\":\"none\"},\"commonAnnotations\":{\"description\":\"This is a Hearbeat event meant to ensure that the entire Alerting pipeline is functional.\",\"summary\":\"Alerting Heartbeat\"},\"externalURL\":\"http://monitoring-prometheus-alertmanager-7fdbf6c5cd-xdlhh:9093\",\"version\":\"4\",\"groupKey\":\"{}/{alertname=\\\"ICPMonitoringHeartbeat\\\"}:{alertname=\\\"ICPMonitoringHeartbeat\\\"}\"}\n"
time="2018-07-05T09:51:05-05:00" level=debug msg="Forwarding an alert to the SNMP trapper" index=0 labels="map[alertname:ICPMonitoringHeartbeat severity:none]" logger=Webhook-server status=firing
time="2018-07-05T09:51:05-05:00" level=debug msg="Received an alert" logger=SNMP-trapper status=firing
time="2018-07-05T09:51:05-05:00" level=debug msg="Created snmpgo.SNMP object" address="127.0.0.1:162" community=public logger=SNMP-trapper retries=1
time="2018-07-05T09:51:05-05:00" level=info msg="It's a trap!" logger=SNMP-trapper status=firing
```

Example Prometheus SNMP traps logged to syslog:

```
Aug  6 07:55:20 csmo-icp2103-rs snmptrapd[7453]: 2018-08-06 07:55:20 UDP: [127.0.0.1]:55091->[127.0.0.1]:162 [UDP: [127.0.0.1]:55091->[127.0.0.1]:162]:#012SNMPv2-SMI::snmpModules.1.1.4.1.0 = OID: PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperFiringNotification#011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationTimestamp = STRING: 2018-08-06T12:14:09Z#011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationDescription = STRING: 99th percentile Latency for LIST requests to the kube-apiserver is higher than 1s.#011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationInstance = STRING: #011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationSeverity = STRING: warning#011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationLocation = STRING: #011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationService = STRING: #011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationJob = STRING: kubernetes-apiservers
Aug  6 07:57:30 csmo-icp2103-rs snmptrapd[7453]: 2018-08-06 07:57:30 UDP: [127.0.0.1]:39868->[127.0.0.1]:162 [UDP: [127.0.0.1]:39868->[127.0.0.1]:162]:#012SNMPv2-SMI::snmpModules.1.1.4.1.0 = OID: PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperFiringNotification#011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationTimestamp = STRING: 2018-08-06T12:00:29Z#011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationDescription = STRING: This is a Hearbeat event meant to ensure that the entire Alerting pipeline is functional.#011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationInstance = STRING: #011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationSeverity = STRING: none#011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationLocation = STRING: #011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationService = STRING: #011PROMETHEUS-TRAPPER-MIB-1::prometheusTrapperNotificationJob = STRING:
```