# How to generate SNMP traps from Prometheus alerts

Prometheus Alertmanager natively supports a couple of different event destinations like Slack, email, PagerDuty, HipChat and [others](https://prometheus.io/docs/alerting/configuration/). For notification mechanisms not natively supported by the Alertmanager, the [webhook receiver](https://prometheus.io/docs/alerting/configuration/#webhook_config) allows for integration.

SNMP traps are still popular method of sending events informing about issues that needs to be reported.
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
Port `9999` is port of `prometheus_webhook_snmptrapper` webhook server and can be changed using input parameter of `prometheus_webhook_snmptrapper`. 

If your Prometheus instance is installed on IBM Cloud Private, then edit `monitoring-prometheus-alertmanager` ConfigMap in `kube-system` namespace.

```
kubectl edit cm monitoring-prometheus-alertmanager -n kube-system
```
Use provided [configuration](monitoring-prometheus-alertmanager.yml) as an example.
The example configuration is based on the alerts defined in the [base set of monitoring alerts for ICP](https://github.com/ibm-cloud-architecture/CSMO-ICP/tree/master/prometheus/alerts_prometheus2.x).

4). Start `snmptrapd` using:

```
snmptrapd -n -d -Lf trap.log -m PROMETHEUS-TRAPPER-MIB.txt 

```
use `PROMETHEUS-TRAPPER-MIB.txt` from [here](https://raw.githubusercontent.com/chrusty/prometheus_webhook_snmptrapper/master/PROMETHEUS-TRAPPER-MIB.txt).

5). Start `prometheus_webhook_snmptrapper`

```
./prometheus_webhook_snmptrapper --webhookaddress 0.0.0.0:9999 > prom_snmp.log 2>&1  &
```

6). Verify that some Prometheus alerts has been generated after you started `prometheus_webhook_snmptrapper` and check the contents of `trap.log` and `prom_snmp.log`.

The contents of prom_snmp.log` should be similar to the one below:

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
`trap.log` showing received SNMP trap with ICPMonitoringHeartbeat alert from ICP Prometheus:

```
Received 277 byte packet from UDP: [127.0.0.1]:49560->[127.0.0.1]:162
0000: 30 82 01 11  02 01 01 04  06 70 75 62  6C 69 63 A7    0........public.
0016: 82 01 02 02  04 21 A3 B1  C1 02 01 00  02 01 00 30    .....!.........0
0032: 81 F3 30 17  06 0A 2B 06  01 06 03 01  01 04 01 00    ..0...+.........
0048: 06 09 2B 06  01 03 8F 39  01 00 01 30  21 06 09 2B    ..+....9...0!..+
0064: 06 01 03 8F  39 01 01 07  04 14 32 30  31 38 2D 30    ....9.....2018-0
0080: 36 2D 32 36  54 31 31 3A  34 39 3A 32  39 5A 30 66    6-26T11:49:29Z0f
0096: 06 09 2B 06  01 03 8F 39  01 01 05 04  59 54 68 69    ..+....9....YThi
0112: 73 20 69 73  20 61 20 48  65 61 72 62  65 61 74 20    s is a Hearbeat 
0128: 65 76 65 6E  74 20 6D 65  61 6E 74 20  74 6F 20 65    event meant to e
0144: 6E 73 75 72  65 20 74 68  61 74 20 74  68 65 20 65    nsure that the e
0160: 6E 74 69 72  65 20 41 6C  65 72 74 69  6E 67 20 70    ntire Alerting p
0176: 69 70 65 6C  69 6E 65 20  69 73 20 66  75 6E 63 74    ipeline is funct
0192: 69 6F 6E 61  6C 2E 30 0D  06 09 2B 06  01 03 8F 39    ional.0...+....9
0208: 01 01 01 04  00 30 11 06  09 2B 06 01  03 8F 39 01    .....0...+....9.
0224: 01 04 04 04  6E 6F 6E 65  30 0D 06 09  2B 06 01 03    ....none0...+...
0240: 8F 39 01 01  03 04 00 30  0D 06 09 2B  06 01 03 8F    .9.....0...+....
0256: 39 01 01 02  04 00 30 0D  06 09 2B 06  01 03 8F 39    9.....0...+....9
0272: 01 01 06 04  00                                       .....
```
