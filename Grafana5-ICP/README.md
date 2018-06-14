ICP 2.1.0.3 provides Grafana 4.6.3 as a part of default monitoring solution. This document describes the steps needed to deploy the latest, externally accessible Grafana 5, together with persistent storage which which permanently store configuration, custom dashboards, installed non-default datasource plugins and panel plugins.

**Note:** Grafana5 helm chart is actively developed by the community and it may change in the near future. Open an issue in case of problems.

## Prerequisites

- [IBM Cloud Private CLI](https://www.ibm.com/support/knowledgecenter/SSBS6K_2.1.0.3/manage_cluster/install_cli.html)
- [Helm CLI](https://www.ibm.com/support/knowledgecenter/SSBS6K_2.1.0.3/app_center/create_helm_cli.html)

## Installation

1). Add `https://kubernetes-charts.storage.googleapis.com` to the list of helm repositories in ICP.

```
helm repo add kubernetes-charts https://kubernetes-charts.storage.googleapis.com
```

2). Verify that new repository is visible in helm CLI client and helm client is properly configured:

```
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈

$ helm repo list
NAME  	URL
kubernetes-charts	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts

$ helm version --tls
Client: &version.Version{SemVer:"v2.7.2+icp", GitCommit:"d41a5c2da480efc555ddca57d3972bcad3351801", GitTreeState:"dirty"}
Server: &version.Version{SemVer:"v2.7.3+icp", GitCommit:"27442e4cfd324d8f82f935fe0b7b492994d4c289", GitTreeState:"dirty"}
```
Note, that Helm client version should match Tiller version installed on ICP. Lower client version may also work.
 
3). Configure PersistentVolume where Grafana configuration will be stored.

a. Create the following file `pv-grafana.yaml` or use the one attached to this document.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: g5-grafana
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/grafana-data
    type: ""
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: g5
```

b. Create PersistentVolume `g5-grafana` using command:

```
kubectl create -f pv-grafana.yml
```

4). Install latest version of Grafana (5.1.3 at the time of writing this procedure).

```
helm install --name g5 kubernetes-charts/grafana --set service.type=NodePort \
--set persistence.enabled=true --set persistence.accessModes={ReadWriteOnce} \
--set persistence.storageClassName=g5 --set persistence.size=1Gi --tls
```

5). Fix the permission issue (noticed in Grafana version 5.1.3) for Grafana container image using the following patch file (or use the one attached to this document):

```
spec:
  template:
    spec:
      containers:
      - name: grafana
        securityContext:
          runAsUser: 0
```
Execute command:

```
kubectl patch deployment g5-grafana --patch "$(cat grafana-patch.yml)"
``` 
6). Verify that PersistentVolume g5-grafana has status BOUND using:

```
kubectl get pv
```

7). Verify you can logon to new Grafana instance. 

a.Identify Grafana URL using:

```
export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services g5-grafana)
export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT
```
b. Collect initial admin password for your new Grafana 5 installation using:

```
kubectl get secret --namespace default g5-grafana -o jsonpath="{.data.admin-password}" \
| base64 --decode ; echo
```
c. Logon to Grafana usign URL collected in step 7a and user `admin` with a password collected in step 7b.

## Test plugin instalation

Now you can install custom plugins and create custom dashboards.
Grafana plugins (check `grafana.com/plugins` for details) can be installed within grafana container shell.

Get the name of the g5-grafana pod:

```
kubectl get pod|grep g5-grafana
```

and enter container shell using:

```
kubectl exec -it <g5-grafana-pod-name> -- bash
``` 

install Grafana plugin, for example:

```
grafana-cli plugins install ibm-apm-datasource
```
(it can be any other plugin from Grafana plugin repository)  and exit Grafana container shell using `exit`.

Delete Grafana pod to restart Grafana server:

```
kubectl delete pod <g5-grafana-pod-name>
```

Logon again to Grafana and verify that datasource plugin has been installed.

## Uninstallation

Delete helm release:

```
helm del --purge g5 --tls
```

Delete PersistentVolume:

```
kubectl delete pv g5-grafana
```
