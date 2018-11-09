# Installing a second ELK-Stack and Kibana Dashboard

The reasons for installing a second ELK stack and Kibana dashboard are:

- Keeping the application logs from custom namespaces separate from the logs from kube-system namespace
- Configure 14 days (different time) before cleaning up application logs in ELK. The kube-system logs are cleaned after 1 day.

ELK

- A custom ELK stack installation collects logfiles from custom namespaces (filebeat configuration)
- Logstash uploads logfiles into separate elasicsearch database
- Database stored in own Persistent Volumens

Kibana

- A custom Kibana dashboard is configured to show logs only from custom ELK stack
  - https://icp-master:8443/kibana-apps

# Installation Steps

Overview ELK stack installation steps
- Create persistance volumen and Filesystem /var/lib/appelk on Management Nodes.
- Create helm chart values.yaml
- Install helm chart ibm-icplogging from IBM charts https://github.com/IBM/charts.git
- Modify filebeat ConfigMap to collect logs from namespace.

Overview Kibana installation steps
- Create helm chart values.yaml
- Install helm chart ibm-icplogging-kibana from IBM charts https://github.com/IBM/charts.git
- Modify ConfigMap and change server.basePath: "/kibana" to : "/kibana-apps" 
- Restart/delete pod kibana.

# Download ICP Helm Charts

Download ICP Helm charts from git repository

    git clone https://github.com/IBM/charts.git

If offline without internet access, then download master.zip and copy to a client with access to ICP

    wget https://github.com/IBM/charts/archive/master.zip

# Install ELK Helm Chart

## Prepare ELK Helm chart values.yaml

Copy values.yaml from charts/stable/ibm-icplogging and modify

Append `-apps` to names

    name: logstash-apps
    name: filebeat-apps
    name: "elasticsearch-apps"
    name: client-apps
    name: master-apps

Set kibana install to false. We install Kibana from separate helm chart ibm-icplogging-kibana
install: false

Set filebeat configuration to collect logs from namespaces

    namespace-apps-a-*
    namespace-apps-b-*
    namespace-apps-c-*

Set mode managed. This ensures that elk stack pods are running on Management Nodes

    mode: managed

## Summary ELK Stack values.yaml

The following changes are made to values.yaml file. Other entries remain on their default value.

```
logstash: 
    name: logstash-apps
kibana: 
  install: false 
filebeat:
  name: filebeat-apps 
      - namespapce-apps-a*  
      - namespace-apps-b* 
elasticsearch:
  name: "elasticsearch-apps"  
  internalPort: 9301 
    name: client-apps  
    restPort: 9201  
    name: master-apps    
    replicas: 2 
    name: data-apps 
      size: 100Gi   
      storageClass: "logging-storage-datanode"
...
mode: managed   # Pods are running on Management Node
```

## Create Persistent Volume

Create filesystem /var/lib/appelk with 100 GB on Mgmt Nodes
Create a Persistent Volume for each Management Node

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-elk-apps-name-ibm-icplogging-data-apps-0
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 100Gi
  local:
    path: /var/lib/appelk
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - 10.125.1.16  # Use Management Node IP
  persistentVolumeReclaimPolicy: Retain
  storageClassName: logging-storage-datanode
```

##  Running helm install

Install elk helm chart from command line

```
helm install --name elk-icp-apps charts/stable/ibm-icplogging  --namespace kube-system –f values.yaml --tls 
 

helm ls elk-icp-apps --tls
NAME            REVISION        UPDATED                         STATUS          CHART                   NAMESPACE
elk-icp-apps    1               Thu Sep 13 09:40:12 2018        DEPLOYED        ibm-icplogging-1.0.1    kube-system
```

## ConfigMap filebeat configures the namespaces to collect logfiles from

Add filter (regex) to collect logs only from certain namespaces 

    kubectl edit cm -n kube-system elk-icp-apps-ibm-icplogging-filebeat-apps-config

Add or modify paths in ConfigMap to filter namespaces

    paths:
        - "/var/log/containers/*_namespace-apps-a-*_*.log"
        - "/var/log/containers/*_namespace-apps-b-*_*.log"
        - "/var/log/containers/*_namespace-apps-c-*_*.log“

Restart filebeat Pods

    kubectl get pod -n kube-system -l release=elk-icp-apps
    kubectl delete pod -n kube-system ...(filebeat pods)

## ConfigMap curator to cleanup  indices

The curator cleans old indices from elastic search. The default is 1 day. 
The requirement is to retain application logs for 14 days.

Edit curator ConfigMap

    kubectl edit cm -n kube-system elk-icp-apps-ibm-icplogging-elasticsearch-apps-curator-config -o yaml

Change unit_count from 1 to 14

       unit_count: 14

Restart elk client pod 

    kubectl get -n kube-system pod | grep client

    kubectl -n kube-system delete pod elk-icp-apps-ibm-icplogging-client-apps-675c4d7dcd-hk7mv


# Install Kibana

## Prepare Kibana Helm Chart values.yaml

Copy values.yaml from charts/stable/ibm-icplogging-kibana/ and modify

```
elasticsearch:
  service:
    name: elasticsearch-apps                                 
    port: 9201                                               

kibana:
  name: kibana-apps                                         
  managedMode: true     # Pods are running on Management Node
````

## Running helm install

``` 
helm install --name kibana-icp-apps charts/stable/ibm-icplogging-kibana/ -f values.yaml --namespace kube-system –tls

helm ls kibana-icp-apps --tls
NAME            REVISION        UPDATED                         STATUS          CHART                           NAMESPACE
kibana-icp-apps 1               Wed Sep 12 14:51:32 2018        DEPLOYED        ibm-icplogging-kibana-1.0.0     kube-system 
```

## Modify Kibana Ingress and ConfigMap

Edit Ingress and change path from /kibana to /kibana-apps

    kubectl edit -n kube-system ingress kibana-icp-apps-ibm-icplogging-kibana-ingress

     path: /kibana-apps

Edit ConfigMap and change server.basePath to "/kibana-apps"

    kubectl edit cm -n kube-system kibana-icp-apps-ibm-icplogging-kibana-config

    server.basePath: "/kibana-apps"

Restart Kibana Pod

    kubectl get pod -n kube-system -l release=kibana-icp-apps
    kubectl delete pod -n kube-system kibana-icp-apps-ibm-icplogging-kibana-5fc8f89c9-nwjbn

# Summary

A second elasticsearch has been deployed

A second kibana dashboard has been deployed and configured to use the new elasticsearch 
  -  https://icp-master:8443/kibana-apps


