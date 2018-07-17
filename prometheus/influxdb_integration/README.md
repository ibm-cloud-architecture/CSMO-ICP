# InfluxDB installation on RedHat/Centos and configuration as a storage backend for Prometheus



Download and install the latest InfluxDB version (currently 1.6). For RedHat/Centos run as root user:

```
wget https://dl.influxdata.com/influxdb/releases/influxdb-1.6.0.x86_64.rpm
yum localinstall influxdb-1.6.0.x86_64.rpm
systemctl start influxdb
```

Create admin user.

```
root@csmo-s03-rs:~# influx
Connected to http://localhost:8086 version 1.6.0
InfluxDB shell version: 1.6.0
> CREATE USER prometheus WITH PASSWORD passw0rd WITH ALL PRIVILEGES
> exit
```

Enable basic authentication for InfluxDB. Edit InfluxDB configuration file `/etc/influxdb/influxdb.conf`. Uncomment parameter `auth-enabled` and set it to `true`.

```
[http]
auth-enabled = true
```

Restart InfluxDB:

```
systemctl restart influxdb
```

Create database `prometheus` which will be used by Prometheus to store data in InfluxDB.

```
root@csmo-s03-rs:~# influx –username prometheus –password passw0rd
Connected to http://localhost:8086 version 1.6.0
InfluxDB shell version: 1.6.0
> create database prometheus
> exit
```


Edit Prometheus configuraiton file `prometheus.yml`

Add sections:

```
remote_write:
  - basic_auth:
      password: passw0rd
      username: prometheus
    url: http://<influxdb_hostname>:8086/api/v1/prom/write?db=prometheus

remote_read:
  - basic_auth:
      password: passw0rd
      username: prometheus
    url: http://<influxdb_hostname>:8086/api/v1/prom/read?db=prometheus
```

Replace `<influxdb_hostname>` with the hostname of the InfluxDB server.

Restart Prometheus and verify that data is written to the Influx database. Example query that shows values of `node_memory_MemFree` metric collected in the last 2 minutes:

```
root@csmo-s03-rs:~# influx –username prometheus –password passw0rd
Connected to http://localhost:8086 version 1.6.0
InfluxDB shell version: 1.6.0

> SELECT "value" FROM "prometheus"."autogen"."node_memory_MemFree" WHERE time > now() - 2m group by "instance"
name: node_memory_MemFree
tags: instance=172.16.246.225:9100
time                value
----                -----
1531816110859000000 358305792
1531816125859000000 360271872
1531816140859000000 355459072
1531816155859000000 354742272
1531816170859000000 355041280
1531816185859000000 353742848
1531816200859000000 355794944
1531816215859000000 353067008

name: node_memory_MemFree
tags: instance=csmo-s01-rs:9100
time                value
----                -----
1531816123101000000 389955584
1531816138101000000 389574656
1531816153101000000 389193728
1531816168101000000 389066752
1531816183101000000 388050944
1531816198101000000 387796992
1531816213101000000 387416064
1531816228101000000 387162112

name: node_memory_MemFree
tags: instance=csmo-s03-rs:9100
time                value
----                -----
1531816123137000000 2404102144
1531816138137000000 2403577856
1531816153137000000 2402181120
1531816168137000000 2401800192
1531816183137000000 2399133696
1531816198137000000 2398625792
1531816213137000000 2398179328
1531816228137000000 2397736960
```