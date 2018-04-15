**Prometheus Federation**

When we first started blogging about monitoring ICP, we demoed federation of multiple ICP deployments to a central Prometheus server using the HTTP protocol. With the recent 2.1.0.1 release, federation should now be done securely using the HTTP/S protocol and the monitoring certificates within ICP. The instructions below will assist one with setting up Federation of mutiple ICP envivornments using the certificates. 

What is needed:

+ A central Prometheus server either running on Cloud or StandAlone.   
(We used a standalone Ubuntu Server)

+ Multiple ICP deployments

+ kubectl installed on your laptop or available to you, if logged in to the master node or elsewhere.

+ kubectl access to the deployments

Once you have established kubectl connectivity, you need to expose a NodePort for connectivity to the Prometheus endpoint in the ICP environment.  Set up a NodePortForPrometheus.yaml with the following entries:

```
apiVersion: v1
kind: Service
metadata:
 labels:
   app: monitoring-prometheus
   component: prometheus
 name: monitoring-prometheus-externalservice
 namespace: kube-system
spec:
 ports:
   - name: https
     port: 8443
     protocol: TCP
     targetPort: 8443
 selector:
   app: monitoring-prometheus
   component: prometheus
 type: "NodePort"
```

 Apply the YAML file"
```
kubectl apply -f NodePortForPrometheus.yaml
```
Get the random node port that was just created:
```
kubectl get --namespace kube-system -ojsonpath="{.spec.ports[0].nodePort}" services monitoring-prometheus-externalservice
```
Example:
```
kubectl get --namespace kube-system -o jsonpath="{.spec.ports[0].nodePort}" services monitoring-prometheus-externalservice
32574
```
You'll need this port for the configuration in the Prometheus server.

Next there are three certificate we need to extract from the ICP endpoint, extract the secret, decode from Base64 and create the certificate files for the Prometheus server.
You may need to install the utilities `jq` and `base64` for this.

***Get the CA certificate***
```
   kubectl get secrets/monitoring-ca-cert -n kube-system -o json | jq -r '.data."ca.pem"' | base64 -d > /etc/prometheus/certs/csmoicp-ca.pem
```

  Next  get the tls Certificate and the Client Key
```
  kubectl get secrets/monitoring-client-certs -n kube-system -o json | jq -r '.data."tls.crt" | base64 -d > /etc/prometheus/certs/csmoicp-tls.pem
  
  kubectl get secrets/monitoring-client-certs -n kube-system -o json | jq -r '.data."tls.key" | base64 -d > /etc/prometheus/certs/csmoicp-key.pem

```

Once complete there should be three files similar to the example below:

```
csmoicp1-ca.pem  
csmoicp1-rsa.pem  
csmoicp1-key.pem
```
Almost there, Next the Prometheus servers yaml file needs to be updated to scrape the endpoint. As shown here, Note: create a job for each endpoint. The IP address of the master node, the NodePort number returned from the kubectl command performed earlier and the full path to the certificate files are needed, 


```
- job_name: 'csmoicp1'
  scrape_interval: 15s
  scheme: https
  honor_labels: true
  metrics_path: '/federate'
  params:
   'match[]':
      - '{_name_=~"job:.*"}'
      - 'up'
      - 'node_memory_MemTotal'
      - 'kubelet_running_pod_count'
      - 'kube_node_info'
      - 'kube_pod_container_status_restarts'
      - 'kubelet_docker_operations_errors'
      - 'container_cpu_system_seconds_total'
      - 'container_cpu_usage_seconds_total'
      - 'container_cpu_user_seconds_total'
      - 'container_memory_usage_bytes'
      - 'kube_deployment_status_observed_generation'
      - 'kube_deployment_status_replicas_available'
      - 'kube_deployment_status_replicas_unavailable'
      - 'kube_pod_container_info'
      - 'node_cpu'
      - 'node_load1'
      - 'node_memory_MemTotal'
      - 'node_memory_MemFree'
      - 'node_memory_Buffers'
      - 'node_memory_Cached'
      - 'node_disk_bytes_read'
      - 'node_disk_bytes_written'
      - 'node_disk_io_time_ms'
      - 'node_filesystem_size'
      - 'node_filesystem_free'
      - 'node_network_receive_bytes'
      - 'node_network_transmit_bytes'

  static_configs:
      - targets: ['IP_ADDRESS:30489']

        labels:
           global_name: 'CSMO-1 ICP'   
           icp_console: 'https://IPADDRESS:8443'
  tls_config:
    ca_file: /etc/prometheus/certs/csmoicp1-ca.pem
    cert_file: /etc/prometheus/certs/csmoicp1-tls.pem
    key_file: /etc/prometheus/certs/csmoicp1-key.pem
    insecure_skip_verify: true

- job_name: 'csmoicp2'
.
.
.
.
```
Restart the Prometheus server, (I always tail the syslog to ensure proper startup).
You should see data from the endpoint note it may take up to a minute or two.
This is a screenshot of our dashboard used to verify the health of Prometheus.
![PromHeatlh](/images/PromHealth.png)


