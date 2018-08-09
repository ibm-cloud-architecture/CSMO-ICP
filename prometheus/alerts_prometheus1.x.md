
***Loading alerts into ICP 2.1 and 2.1.0.1*** 

The following procedure is suitable for version 1.x of Prometheus which was provided up to version 2.1.0.1 of ICP. Higher versions of ICP use Prometheus 2.x which has a different rules format.

This procedure depends on the use of the tool [jq](https://stedolan.github.io/jq/)
A number of sample rule have been included in this repository. 

1. [Download the files](https://github.com/ibm-cloud-architecture/CSMO-ICP/tree/master/prometheus/rules) into a subdirectory (make sure they are the only files there). 
2. Choose the specific rulefile you want to load by running the command `File=<path_to_file>`. You can also specify an entire directory, but note that all the files in that directory will be loaded.
3. Run the command `kubectl get configmap alert-rules -o json --namespace=kube-system | jq --argjson newRule "$(kubectl create configmap test --from-file=$File --dry-run -o json | jq .data)" '.data |= . + $newRule' | kubectl replace -f -`
4. This command does 4 things:
* `kubectl get configmap alert-rules -o json --namespace=kube-system` extracts the existing ConfigMap which contains the rules
* `"$(kubectl create configmap test --from-file=$File --dry-run -o json | jq .data)"` creates a dummy ConfigMap out of the rulesfile and extracts the data section without loading it into Kubernetes
* `'.data |= . + $newRule'` appends the new data to the existing data
* `| kubectl replace -f -` replaces the old configmap with the updated one
5. View the ConfigMap and make sure that the rules have been loaded with the command `kubectl describe cm --namespace=kube-system alert-rules`
6. Reload the rules into Prometheus by browsing to `https://<master_ip>:8443/prometheus/reload` (or by restarting the Prometheus pod).
