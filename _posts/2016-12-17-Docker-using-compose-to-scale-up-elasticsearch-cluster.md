---
layout: post
title:  "Docker - Using Compose to scale up elasticsearch cluster"
date:   2016-12-17 23:30:01 +0200
categories: docker elasticsearch
page.author: Konstantinos
---

In a previous <a href="http://blog.codingtimes.com/docker-elastic-search-create-a-cluster-with-multiple-nodes/" target="_blank">post</a> we' ve shown how to create a cluster in elasticsearch with manually defined nodes. In this post we use docker compose to scale things up!

Elasticsearch has many solutions to play with (i.e. logstash and kibana aka elk, hq etc) but I have tried to use a more simple example and comment on various issues that I have come across.

<h3>Creating the compose file</h3>

First create a docker-compose.yml file and add two services:
<pre><strong>$docker&gt;</strong>mkdir es-cluster
<strong>$docker&gt;</strong>cd es-cluster
<strong>$es-cluster&gt;</strong>touch docker-compose.yml</pre>

With a text editor open docker-compose file and add the following:
<pre>version:'2'
    services:
    master:
        image: library/elasticsearch
        command: elasticsearch -network.host=0.0.0.0 -node.master=true -cluster.name=cluster-01 -node.name="Master of Disaster"
        volumes:
        - ./elasticsearch/config/:/usr/share/elasticsearch/config/
        - ./elasticsearch/logs/:/usr/share/elasticsearch/logs/
        ports:
        - "9200:9200"
        restart: always
        container_name: es_master
    node:
        image: library/elasticsearch
        command: elasticsearch -network.host=0.0.0.0 -cluster.name=cluster-01 -discovery.zen.ping.unicast.hosts=es_master
        restart: always
        volumes:
        - ./elasticsearch/config/:/usr/share/elasticsearch/config/
        - ./elasticsearch/logs/:/usr/share/elasticsearch/logs/
        depends_on:
        - master
        links:
        - master:es_master</pre>

<strong>master: </strong> The first service named master will be the master of the cluster.

<strong>node: </strong> The second service named node will represent all data nodes that participate in the cluster.

<strong>image: library/elasticsearch</strong> pulls (if not existing) the image of elastic search.

<strong>-node.master=true</strong> we set that in the master service to declare it as the master of the cluster.

<strong>-node.name="Master of Disaster"</strong> optionally we give a name to the master node.

<strong>container_name: es_master</strong> we set the name es_master to the container so that the master will be discoverable by the rest data nodes.

<strong>-discovery.zen.ping.unicast.hosts=es_master</strong> each node will look up for the es_master container to connect.

<strong>- "9200:9200"</strong> in the ports section we expose the http port to the host.

<strong>-network.host=0.0.0.0</strong> defines the where the service will be published. If this is not set the service will be published in the localhost (127.0.0.2) of the container and it won't be available outside of it (at least not at Docker 1.12 Beta that I am using). By defining 0.0.0.0 as network host the service is exported in the containers network ip (i.e. publish_address {172.22.0.2:9200}).

<strong>-cluster.name=cluster-01</strong> defines the name of the cluster. This should unique and the same to both master and node services in order all nodes to participate in the same cluster.

In the <strong>volumes</strong> section we map the local directories to those in the containers:
<pre>- ./elasticsearch/config/:/usr/share/elasticsearch/config/
- ./elasticsearch/logs/:/usr/share/elasticsearch/logs/</pre>

<strong>restart: always</strong> so that the master will always restart automatically every time the system reboots.

<strong>- master</strong> in the <strong>depend_on</strong> section we say that node will be depending on the master.

<strong>- master:es_master</strong> finally, in the <strong>links</strong> section we declare the link towards the master service.

<h3>Preparing for the first run</h3>

Before running this configuration for the first time it would be nice to create a logging file and set it in the config directory so that the log4j instance of elasticsearch can export logs in a favourable format.
So, create the directories needed:
<pre><strong>$cluster&gt;</strong>mkdir elasticsearch
<strong>$cluster&gt;</strong>cd elasticsearch
<strong>$elasticsearch&gt;</strong>mkdir config</pre>
Then create the logging.yml file in the config directory:
<pre><strong>$elasticsearch&gt;</strong>cd config
<strong>$config&gt;</strong>touch logging.yml</pre>
To create two appenders (one for the console and one to write into a file) open logging.yml with a text editor and add the following:
<pre>es.logger.level: INFO
rootLogger: ${es.logger.level}, console, file
logger:
    action: TRACE
    appender:
        console:
            type: console
            layout:
                type: consolePattern
                conversionPattern: "[%d{ISO8601}][%-5p][%-25c] %m%n"
        file:
            type: dailyRollingFile
            file: ${path.logs}/${cluster.name}.log
            datePattern: "yyyy-MM-dd"
            layout:
                type: pattern
                conversionPattern: "[%d{ISO8601}][%-5p][%-25c] %m%n"</pre>

<h3>Run for the first time</h3>

Run to test without scaling.
<pre><strong>$es-cluster&gt;</strong>docker-compose up</pre>
This will create a default network for the containers, es-cluster_default, named after the directory that the compose file is placed. 
The services will be attached on that network to communicate.

<a href="{{site.url}}/assets/Screenshot-2016-07-01-10.56.25-1.png"><img src="{{site.url}}/assets/Screenshot-2016-07-01-10.56.25-1.png" alt="Screenshot 2016-07-01 10.56.25"/></a>
To see cluster's health, enter the following URL:
<pre>localhost:9200/_cluster/health?pretty=true</pre>
Something like this should be shown:
<pre>{
    "cluster_name" : "cluster-01",
    "status" : "green",
    "timed_out" : false,
    "number_of_nodes" : 2,
    "number_of_data_nodes" : 2,
    "active_primary_shards" : 0,
    "active_shards" : 0,
    "relocating_shards" : 0,
    "initializing_shards" : 0,
    "unassigned_shards" : 0,
    "delayed_unassigned_shards" : 0,
    "number_of_pending_tasks" : 0,
    "number_of_in_flight_fetch" : 0,
    "task_max_waiting_in_queue_millis" : 0,
    "active_shards_percent_as_number" : 100.0
}</pre>
<h3>Scale up the cluster</h3>
Now that we are positive that everything runs smoothly, we can stop the running processes (either with Ctrl C or with docker-compose stop).
Beware only of one thing. If you terminate the processes by running docker-compose down it will stop and remove not only the running containers but also the network es-cluster_default. To avoid this and not depend on this network, it is better to create an external network and use it in the compose file:
<pre><strong>$es-cluster&gt;</strong>docker network create es-network</pre>
Edit the docker-compose.yml file and add the following lines at the end of the file:
<pre>networks:
    default:
    external:
        name: es-network</pre>
This section defines that the default network for the containers will the es-network instead of the es-cluster_default. And because it is the default network there is no need to add a <strong>networks</strong> section in each service. So, now we are ready to fire up our cluster with a master node and, let's say, 5 more nodes:
<pre><strong>$es-cluster&gt;</strong>docker-compose scale master=1 node=5</pre>
Once again, to see our cluster's health, in the browser give the address:
<pre>localhost:9200/_cluster/health?pretty=true</pre>
Something like this should be shown:
<pre>{
    "cluster_name" : "cluster-01",
    "status" : "green",
    "timed_out" : false,
    "number_of_nodes" : 6,
    "number_of_data_nodes" : 6,
    "active_primary_shards" : 0,
    "active_shards" : 0,
    "relocating_shards" : 0,
    "initializing_shards" : 0,
    "unassigned_shards" : 0,
    "delayed_unassigned_shards" : 0,
    "number_of_pending_tasks" : 0,
    "number_of_in_flight_fetch" : 0,
    "task_max_waiting_in_queue_millis" : 0,
    "active_shards_percent_as_number" : 100.0
}</pre>

<h3>Loading data</h3>

At that point you need to load some data into elasticsearch. From <a href="https://www.elastic.co/guide/en/elasticsearch/reference/current/_exploring_your_data.html#_loading_the_sample_dataset" target="_blank">Loading the Sample Dataset</a> you can download the sample dataset (accounts.json), extract it to your current directory and load it into your cluster as follows:
<pre><strong>$es-cluster&gt;</strong>curl -XPOST 'localhost:9200/bank/account/_bulk?pretty' --data-binary "@accounts.json"</pre>
After loading it run:
<pre><strong>$es-cluster&gt;</strong>curl 'localhost:9200/_cat/indices?v'</pre>
and it will show:
<pre>health status index pri rep docs.count docs.deleted store.size pri.store.size
green open bank 5 1 889 0 344.3kb 189.7kb</pre>
If we refresh the page in the browser it will show:
<pre>{
    "cluster_name" : "cluster-01",
    "status" : "green",
    "timed_out" : false,
    "number_of_nodes" : 6,
    "number_of_data_nodes" : 6,
    "active_primary_shards" : 5,
    "active_shards" : 10,
    "relocating_shards" : 0,
    "initializing_shards" : 0,
    "unassigned_shards" : 0,
    "delayed_unassigned_shards" : 0,
    "number_of_pending_tasks" : 0,
    "number_of_in_flight_fetch" : 0,
    "task_max_waiting_in_queue_millis" : 0,
    "active_shards_percent_as_number" : 100.0
}</pre>

<h3>Adding the HQ plugin</h3>

A nice plugin for visualising the cluster and query the data within is the HQ. The easiest way to install it is to run the plugin install command for the master:
<pre><strong>$es-cluster&gt;</strong>docker exec es_master plugin install royrusso/elasticsearch-HQ</pre>
This will download hq and it will install it. Then, in the browser go to the address <em>localhost:9200/_plugin/hq</em> and connect to the <em>http://localhost:9200</em> in the top of the screen.

<a href="{{site.url}}/assets/Screenshot-2016-07-01-12.13.42.png">
    <img src="{{site.url}}/assets/Screenshot-2016-07-01-12.13.42.png" alt="Screenshot 2016-07-01 12.13.42" />
</a>

<h3>Summary</h3>
This was yet another simple example of creating clusters with Docker compose. I hope though, I sed some light in various parts of the docker compose and networking&#8230;   
Enjoy coding!
