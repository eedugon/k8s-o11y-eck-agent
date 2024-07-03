# Kubernetes Observability with ECK and Elastic Agent

## Introduction

The purpose of this demonstration is to show ECK capabilities and how easy we can orchestrate Elasticsearch clusters and implement a Kubernetes Observability use case.

Components:

- GKE Cluster
- kube-state-metrics
- ECK (Elastic Cloud on Kubernetes) operator
- Elasticsearch Cluster with Kibana
- Elastic Agent acting as Fleet Server (Deployment)
- Elastic Agent for kubernetes node level metrics and logs (DaemonSet)
- Elastic Agent for kubernetes cluster level metrics (Deployment)
- APM Server (for APM demo)
- Example application (for APP demo)

There's an interesing guide added which doesn't use ECK: https://www.elastic.co/guide/en/starting-with-the-elasticsearch-platform-and-its-solutions/current/getting-started-kubernetes.html

## ECK Installation

Before installing ECK on GKE ensure your user has `cluster-admin` permissions.

```
kubectl create clusterrolebinding cluster-admin-binding-gke --clusterrole=cluster-admin --user=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")
```

- Deploy CRDs:

```bash
kubectl create -f https://download.elastic.co/downloads/eck/2.13.0/crds.yaml
```

- Deploy Operator:

```bash
kubectl apply -f https://download.elastic.co/downloads/eck/2.13.0/operator.yaml
```

The following commands should be run from the `resources` directory of the repository:

```bash
cd resources
```

## Kube-state-metrics installation

Kube-state-metrics is an external component used by some of the Kubernetes Integration metricsets in charge of cluster monitoring.

Installation instructions can be found [here](https://github.com/kubernetes/kube-state-metrics). In our case:

```bash
kubectl apply -f kube-state-metrics-v2.12.0/
```

After the installation, KSM will be available via `http://kube-state-metrics.kube-system.svc.cluster.local:8080`.

## Elasticsearch and Kibana deployment

Deploy Elasticsearch and Kibana:

```bash
kubectl apply -f 01_es_quickstart.yaml -f 02_kb_quickstart.yaml
```

Review all objects created by ECK:
  - Services
  - Secrets
  - ConfigMaps
  - StatefulSets for Elasticsearch
  - Deployment for Kibana

ECK creates and maintains a lot of secrets:

- Elastic user password
- CA certificate for HTTPS endpoints (separate secrets for Elasticsearch and Kibana)
- Configuration files
- Passwords for certain users

### Gain access to the platform

The `elastic` password can be obtained from one of the created Secrets:

```bash
kubectl get secret elasticsearch-quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode
```

- Access Kibana via the Kubernetes service `kibana-quickstart-kb-http`.


## Deploy RBAC for Elastic Agents

```bash
kubectl apply -f 10_RBAC_ElasticAgent.yaml
```

## Deploy Fleet Server

```bash
kubectl apply -f 11_FleetServer.yaml
```

## Deploy Elastic Agent

```bash
kubectl apply -f 12_ElasticAgent-DaemonSet.yaml
```

Verify all is running properly.

You will see that the DaemonSet agents are providing system monitoring information but nothing about Kubernetes. That's because the created policy didn't include Kubernetes integration. We will add it manually in the next step.

## Add Kubernetes Integration to the DaemonSet agents policy

Open the Fleet Policy used by the DaemonSet agents and add `Kubernetes integration` with the following inputs configuration:

- All Kubelet API related metrics
- All `kube-state-metrics` related metrics (Leader Election enabled). Ensure to use the right host for all the state metricsets: `kube-state-metrics.kube-system.svc.cluster.local:8080`
- Kubernetes Events (Leader Election enabled)
- Apiserver metrics (Leader Election enabled)
- Container logs

After applying the changes all the data should be flowing. Interesting views:
- Discover for troubleshooting, filtering data or checking available values of different fields.
- Kubernetes Overview Dashboard
- Kubernetes Pods Dashboard
- Fleet UI
- Observability UI

__Note__: If the cluster level metrics (with leader election) are not flowing, you might need to delete the lease so it's recreated:

```bash
kubectl delete lease elastic-agent-cluster-leader
```

This could happen if an unexpected agent has acquired the lease (for example the Fleet Server).

# APM Demo

We can add an [APM Standalone Server](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-apm-server.html) as described in the ECK documentation.

__Prerequisite__: The APM Server, even when being run in standalone mode (not managed by Fleet) requires the APM Integration to be added in Kibana.

- Deploy APM Server:

```bash
kubectl apply -f 40_apm_server-8.14.1.yaml
```

- Obtain the token for the clients connections:

```bash
kubectl get secret/apm-server-quickstart-apm-token -o go-template='{{index .data "secret-token" | base64decode}}'
```

- Obtain the URL for the clients connections:

```bash
kubectl get service --selector='common.k8s.elastic.co/type=apm-server'
```

Configure the URL and Token in the application (`app-example/app-deployment.yaml`), like:

```yaml
          - name: ELASTIC_APM_SERVER_URL
            value: "https://apm-server-quickstart-apm-http.default.svc.cluster.local:8200"
          - name: ELASTIC_APM_SECRET_TOKEN
            value: "db2KXI7Z011CvE54t5DJ37nQ"
```

Deploy the application:

```bash
kubectl apply -f apm-example/
```

Generate some traffic:

```bash
kubectl port-forward svc/flask-counter-svc 5000:5000

# and in another window
curl localhost:5000
```

# Stack Monitoring demo

- Deploy 2 extra clusters that will be sending monitoring data to the previously created clusters.

- Create a central monitoring cluster and ship logs and metrics from all clusters to this new one instead.

# Troubleshooting notes

- Elastic agent possible issues:

Check for lease adquisition, needed for the data streams that require a leader election.

```bash
 k logs elastic-agent-quickstart-agent-8gbgt | grep -i lease
I0703 14:19:14.082880     996 leaderelection.go:248] attempting to acquire leader lease default/elastic-agent-cluster-leader...
I0703 14:19:42.378425     996 leaderelection.go:258] successfully acquired lease default/elastic-agent-cluster-leader
```

Check that all expected metricsets are being provided, for example `state_*`:

```log
{"log.level":"info","@timestamp":"2024-07-03T14:52:09.977Z","message":"Non-zero metrics in the last 30s","component":{"binary":"metricbeat","dataset":"elastic_agent.metricbeat","id":"kubernetes/metrics-default","type":"kubernetes/metrics"},"log":{"source":"kubernetes/metrics-default"},"log.logger":"monitoring","log.origin":{"file.line":187,"file.name":"log/log.go","function":"github.com/elastic/beats/v7/libbeat/monitoring/report/log.(*reporter).logSnapshot"},"service.name":"metricbeat","monitoring":{"ecs.version":"1.6.0","metrics":{"beat":{"cgroup":{"memory":{"mem":{"usage":{"bytes":0}}}},"cpu":{"system":{"ticks":780,"time":{"ms":140}},"total":{"ticks":5400,"time":{"ms":1060},"value":5400},"user":{"ticks":4620,"time":{"ms":920}}},"handles":{"limit":{"hard":1048576,"soft":1048576},"open":22},"info":{"ephemeral_id":"6c01ac15-4c51-41a5-8d41-8872aee5ae7d","uptime":{"ms":631602},"version":"8.14.1"},"memstats":{"gc_next":98676648,"memory_alloc":53293520,"memory_sys":26364944,"memory_total":742908704,"rss":228708352},"runtime":{"goroutines":314}},"filebeat":{"harvester":{"open_files":0,"running":0}},"libbeat":{"config":{"module":{"running":23}},"output":{"events":{"acked":1872,"active":0,"batches":3,"duplicates":2,"total":1874},"read":{"bytes":49958,"errors":3},"write":{"bytes":578975,"latency":{"histogram":{"count":56,"max":981,"mean":103.03571428571429,"median":67,"min":49,"p75":83.75,"p95":287.5499999999994,"p99":981,"p999":981,"stddev":145.7396117696747}}}},"pipeline":{"clients":23,"events":{"active":211,"published":1854,"total":1854},"queue":{"acked":1874}}},"metricbeat":{"kubernetes":{"apiserver":{"events":993,"success":993},"container":{"events":57,"success":57},"node":{"events":3,"success":3},"pod":{"events":33,"success":33},"proxy":{"events":18,"success":18},"state_container":{"events":201,"success":201},"state_daemonset":{"events":81,"success":81},"state_deployment":{"events":36,"success":36},"state_namespace":{"events":24,"success":24},"state_node":{"events":9,"success":9},"state_persistentvolume":{"events":9,"success":9},"state_persistentvolumeclaim":{"events":9,"success":9},"state_pod":{"events":108,"success":108},"state_replicaset":{"events":45,"success":45},"state_resourcequota":{"events":18,"success":18},"state_service":{"events":42,"success":42},"state_statefulset":{"events":9,"success":9},"state_storageclass":{"events":9,"success":9},"system":{"events":9,"success":9},"volume":{"events":141,"success":141}}},"registrar":{"states":{"current":0}},"system":{"load":{"1":0.42,"15":0.65,"5":0.44,"norm":{"1":0.105,"15":0.1625,"5":0.11}}}}},"ecs.version":"1.6.0"}
```


REDEPLOYING EVERYTHING:

Be careful with the enrollment token and the data directory, there's a limitation on Elastic Agent side and the state directory needs to be cleaned some times.


# Suggested Improvements

- Divide the Kubernetes Observability workload in a DaemonSet + Deployment:
  - DaemonSet for node level metrics and logs
  - Deployment for cluster level metrics (kube-state-metrics related metrics, events, api-server, etc.)

That would require 2 different policies, both with Kubernetes integration, but each of them configured with a different set of inputs.

- Agents showing with the real hostname on Fleet UI instead of the pod name (for the DaemonSet).


