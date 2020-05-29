## Kubernetes Elastic Stack Implementation

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

_OBS_: This documentation explains how to publish your Elastic stack services outside your local kubernetes cluster. If you want to make it publicly available to the internet, additional configuration is needed, such as DNS registrar and firewall configuration, both of which are out of the scope of this document.

### Pre-Requisites

- a fully-functional kubernetes cluster (v1.16+)
- 1x DNS record for Kibana Web Interface (_kibana.in.k8s_)
- 1x DNS record for Elasticsearch Web API (_elastic.in.k8s_)
- NGINX Ingress Controller (instructions to deploy one [here](https://kubernetes.github.io/ingress-nginx/deploy/))


### How To Use This Repo

To implement the Elastic stack on Kubernetes, follow the steps described further below.

#### 1. Configuring Web Access URLs

__ADD__ the following lines to your `/etc/hosts` file:
```
127.0.0.1	elastic.in.k8s
127.0.0.1	kibana.in.k8s
```

The loopback IP address of your localhost now 'understands' these names.

If you wish to use different names, check out the _customizing Web Access URLs_ section.

If your DNS server is already configured to resolve these names, you might skip this step. You can also use different IP addresses, as long as they point to your kubernetes cluster.

#### 2. Applying the Kubernetes Configuration

Clone this repo and then run the following command at the root of the project:

`kubectl create -f manifests/`  

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

### Customizing Web Access URLs

If you want to customize the Web Access URLs, make sure you have downloaded the repo, edit your `/etc/hosts` file with appropriate names of your choosing, then navigate to the folder where you downloaded the repo, open the `manifests/ingress-nginx.yaml` file and change the following lines to match the new names:

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

#### Elasticsearch

##### Pod Stuck in Pending Status

```
NAME                        READY   STATUS              RESTARTS   AGE
elasticsearch-0             0/1     Pending             0          45s
```

This is usually due to insufficient memory.

Run `kubectl describe pod ${elasticsearch-[0...n]` to make sure that is indeed the case and look under the _Events_ section.

Free up memory and try to redeploy the cluster. If you are running Docker-for-Desktop on Windows or Mac, check the memory limit settings.

##### Cluster Health Issues

If the cluster health command does not return the expected number of nodes in green status, run the following command (for each node in the cluster):

`kubectl logs ${elasticsearch_[0...n]}`  

Scan through the logs to find the probable reason why the cluster is not running as it should.

Regardless if Kibana and Logstash are indeed sending logs to Elasticsearch, the cluster health request should return a satisfactory response.

Again, this is probably due to one or more nodes not starting due to insufficient memory. See _pod stuck in pending status_.

#### Kibana

If you do not see the filebeat index in Kibana, chances are that something is wrong with either filebeat or logstash pipelines.

If you edited filebeat's configmap, you might find that your logstash is not sending logs to Elasticsearch. 

Often times, the issue is _filebeat.yml_ configuration. This file is rather flimsy and does not always notify you of linting issues or mispelled sections in your filebeat configuration. Instead, filebeat rolls back to default configuration, which can be quite frustrating since it does not leave any hint or trace of this in the logs.

In any case, round up the usual suspects by running the following commands:

`kubectl logs ${filebeat_pod}`  
`kubectl logs ${logstash_pod}`  

Scan through the log files and see if there is any hint as to what could be the issue. If not, review your filebeat configuration __VERY__ carefully. A simple mispelling such as `contain` instead of `contains` in the _condition_ stanza could throw you off, because filebeat would simply ignore your conditions rather than notify you about them. 

For further information, please read the [filebeat](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-configuration.html) configuration documentation.

In logstash, try to find the following lines:

```
[2020-05-29T14:19:26,182][INFO ][logstash.agent           ] Pipelines running {:count=>2, :running_pipelines=>[:".monitoring-logstash", :main], :non_running_pipelines=>[]}
[2020-05-29T14:19:26,531][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
```

The '2' count for number of loaded pipelines refers to the default pipeline which is configured as part of the kubernetes deployment, and the xpack monitoring pipeline, which runs by default on this distro of Elasticsearch.

If logstash is running both pipelines, then the issue is filebeat.
