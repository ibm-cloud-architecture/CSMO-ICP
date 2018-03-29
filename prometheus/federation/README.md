**Prometheus Federation**
When we first start bloggin about monitoring ICP, we demoed federation if mutilple deploymnets over the http protocol. With the recent 2.1.0.1 release, federation can now be done securly using the https protocol and the monitoring certificates within ICP.

What you need:
A central Prometheus server either running on Cloud or StandAlone. (We used a standalone Ubuntu Server)
Mutiple ICP deployments
kubectl access to the deployments

