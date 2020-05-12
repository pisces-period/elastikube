# Kubernetes Elastic Stack Implementation

This documentation section is under construction :hammer:

This project provisions a not _quite_ production-grade Elastic stack cluster with filebeat's [_kubernetes autodiscovery_](https://www.elastic.co/guide/en/beats/filebeat/current/configuration-autodiscover-hints.html) functionality.

It provides, out of the box:

- 3x node Elasticsearch cluster
- 2x Kibana Web Interface (service load-balancing)
- 1x Logstash
- 1x Filebeat
- 1x Nginx Ingress with rules for the Elasticsearch Web API and Kibana Web Interfaces

The kubernetes manifests are the building blocks for a production-grade cluster.    

You can add as many Elasticsearch nodes as your kubernetes cluster can support and tweak with each component's _configmaps_ to add more advanced configuration.

## Pre-Requisites

- a fully-functional kubernetes cluster (v1.16+)
- 1x DNS record for Kibana Web Interface (_kibana.in.k8s_)
- 1x DNS record for Elasticsearch Web API (_elastic.in.k8s_)
- NGINX Ingress Controller (instructions to deploy one [here](https://kubernetes.github.io/ingress-nginx/deploy/))


## How To Use This Repo

To  implement the Elastic stack on Kubernetes, follow the steps described further below.

#### 1. Configuring Web Access URLs

__ADD__ the following lines to your `/etc/hosts` file:
```
127.0.0.1	elastic.in.k8s
127.0.0.1	kibana.in.k8s
```

If you wish to use different names, check out the _customizing Web Access URLs_ section.

#### 2. Applying the Kubernetes Configuration

You have 2 options to apply the configuration to your kubernetes cluster:

1) Clone GitHub Repo:

Clone this repo and then run the following command at the root of the project:

`kubectl apply -f manifests/`

2) Run via HTTPS:

Simply run the command:

`kubectl apply -f https://${URL}/`

Run `kubectl get po` and you should see the an output similar to this:

```
NAME                        READY   STATUS    RESTARTS   AGE
elasticsearch-0             1/1     Running   0          1m
elasticsearch-1             1/1     Running   0          1m
elasticsearch-2             1/1     Running   0          1m
filebeat-p4gng              1/1     Running   4          1m
kibana-867cc6cf58-dwmgs     1/1     Running   0          1m
logstash-674685d55f-6dxmf   1/1     Running   0          1m
```

_OBS:_ Do not worry about filebeat restarts, it runs a liveness probe that keeps restarting the container until Logstash is ready to receive logs.

#### 3. Enabling Autodiscovery

We're half-way there. Now that we have the logging and indexing components in place, we must 'help' filebeat determine how and which pods to harvest logs from.

Create a new deployment (any container of your choosing) and add the following lines to the metadata section of the pod template:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${some_name}
  labels:
    ${label}: ${value}
spec:
  replicas: 1
  selector:
    matchLabels:
      ${label}: ${value}
  template:
    metadata:
      annotations:
        co.elastic.logs/enabled: true # this line tells filebeat to harvest logs from this pod
```

Now that you have running pods annotated for log collection, let's proceed to the testing if our setup works.

#### 4. Testing

##### Elasticsearch

It's time to test all of our hard work.

Open your web browser and go to `http://${elastic_web_url}:9200/_cluster/health?`

WHERE ${elastic_web_url} corresponds to the name(s) you specified in your `/etc/hosts` file.

You should see something like this:

```
{"cluster_name":"elastic-cluster","status":"green","timed_out":false,"number_of_nodes":3,"number_of_data_nodes":3,"active_primary_shards":4,"active_shards":8,"relocating_shards":0,"initializing_shards":0,"unassigned_shards":0,"delayed_unassigned_shards":0,"number_of_pending_tasks":0,"number_of_in_flight_fetch":0,"task_max_waiting_in_queue_millis":0,"active_shards_percent_as_number":100.0}
```

If you do not see it, proceed to _troubleshooting_ section.

##### Kibana 

Open your web browser and go to `http://${kibana_web_url}`

WHERE ${kibana_web_url} corresponds to the name(s) you specified in your `/etc/hosts` file.

Click on the `gear` icon at the bottom-left corner of the screen, then click on `Index Patterns` under Kibana. Click on `Create Index Pattern`, type `filebeat-*` and then proceed to `Next Step`.

Click on the `compass` icon on the top-left corner of the screen to see logs corresponding to this index. 

If you do not see it, proceed to _troubleshooting_ section.

### Customizing URLs

If you want to customize the Web Access URLs, it is easier if you download this repo. Make sure you have downloaded the repo, edit your `/etc/hosts` file with appropriate names of your choosing, then navigate to the folder where you downloaded the repo, open the `manifests/ingress-nginx.yaml` file and change the following lines to match the new names:

```
<omitted>
  - host: elastic.in.k8s # this name must be resolvable by your DNS server
<omitted>
  - host: kibana.in.k8s # this name must be resolvable by your DNS server
```

Then, redeploy the ingress:

`kubectl delete -f manifests/ingress-nginx.yaml`
`kubectl create -f manifests/ingress-nginx.yaml`

That's it. =) 

You don't need to update the name anywhere else in the code.

### Troubleshooting

This section is under construction.



