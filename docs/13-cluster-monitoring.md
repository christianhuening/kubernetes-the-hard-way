Cluster Monitoring
===============

We finally want Cluster Monitoring to see the status of all nodes, machine etc.
For that purpose we utilize Prometheus, the Alertmanager and Grafana.

Setup
------------------------

To deploy the monitoring setup, simply run the root of this repository:

```bash
export KUBECONFIG=<path>          # defaults to "~/.kube/config"
SupportingServices/Monitoring/kube-prometheus/hack/cluster-monitoring/deploy
```

This will deploy all necessary services in the monitoring namespace and allow
you to access them through kubectl proxy

Change Alertmanager Config
------------------------

1. Make your changes to SupportingServices/Monitoring/kube-prometheus/assets/alertmanager/alertmanager.yaml
2. Run SupportingServices/Monitoring/kube-prometheus/hack/scripts/generate-alertmanager-config-secret.sh
3. Update SupportingServices/Monitoring/kube-prometheus/manifests/alertmanager/alertmanager-config.yaml with the result
4. kubectl apply alertmanager-config.yaml to the 'monitoring' namespace
