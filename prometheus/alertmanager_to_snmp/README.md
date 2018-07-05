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

