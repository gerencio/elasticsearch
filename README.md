# ElasticSearch docker

## Basic Usage

To start an Elasticsearch data node that listens on the standard ports on your host's network interface:

    docker run -d -p 9200:9200 -p 9300:9300 gerencio/elasticsearch:<version>

You'll then be able to connect to the Elasticsearch HTTP interface to confirm it's alive:

http://DOCKERHOST:9200/

```json

    {
      "status" : 200,
      "name" : "Charon",
      "version" : {
        "number" : "1.3.5",
        "build_hash" : "4a50e7df768fddd572f48830ae9c35e4ded86ac1",
        "build_timestamp" : "2014-11-05T15:21:28Z",
        "build_snapshot" : false,
        "lucene_version" : "4.9"
      },
      "tagline" : "You Know, for Search"
    }

```

Where `DOCKERHOST` would be the actual hostname of your host running Docker.

## Simple, multi-node cluster

To run a multi-node cluster (3-node in this example) on a single Docker machine use:

```bash

    docker run -d --name es0 -p 9200:9200                    gerencio/elasticsearch:<version>
    docker run -d --name es1 --link es0 -e UNICAST_HOSTS=es0 gerencio/elasticsearch:<version>
    docker run -d --name es2 --link es0 -e UNICAST_HOSTS=es0 gerencio/elasticsearch:<version>

```

and then check the cluster health, such as http://192.168.99.100:9200/_cluster/health?pretty

```json

    {
      "cluster_name" : "elasticsearch",
      "status" : "green",
      "timed_out" : false,
      "number_of_nodes" : 3,
      "number_of_data_nodes" : 3,
      "active_primary_shards" : 0,
      "active_shards" : 0,
      "relocating_shards" : 0,
      "initializing_shards" : 0,
      "unassigned_shards" : 0
    }

```

## Configuration Summary

### Ports

* `9200` - HTTP REST
* `9300` - Native transport

### Volumes

* `/data` - location of `path.data`
* `/conf` - location of `path.conf`

## Configuration Details

The following configuration options are specified using `docker run` environment variables (`-e`) like

```bash

docker run ... -e NAME=VALUE ... gerencio/elasticsearch:<version>

```

Since Docker's `-e` settings are baked into the container definition, this image provides an extra feature to change any of the settings below for an existing container. Either create/edit the file `env` in the `/conf` volume mapping or edit within the running container's context using:

    docker exec -it CONTAINER_ID vi /conf/env

replacing `CONTAINER_ID` with the container's ID or name.

The contents of the `/conf/env` file are standard shell

```bash

-e NAME=VALUE

```

entries where `NAME` is one of the variables described below.

Configuration options not explicitly supported below can be specified via the `OPTS` environment variable. For example, by default `OPTS` is set with

```bash

 -e OPTS=-Dnetwork.bind_host=_non_loopback_

```

_NOTE: That option is a default since `bind_host` defaults to `localhost` as of 2.0, which isn't helpful for
port mapping out from the container_.

### Cluster Name

If joining a pre-existing cluster, then you may need to specify a cluster name different than the default "elasticsearch":

```bash

-e CLUSTER=dockers

```

### Zen Unicast Hosts

When joining a multi-physical-host cluster, multicast may not be supported on the physical network. In that case, your node can reference specific one or more hosts in the cluster via the [Zen Unicast Hosts](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-discovery-zen.html#unicast) capability as a comma-separated list of `HOST:PORT` pairs:

```bash

-e UNICAST_HOSTS=HOST:PORT[,HOST:PORT]

```

such as

```bash

-e UNICAST_HOSTS=192.168.0.100:9300

```

<!-- ### Plugins

You can install one or more plugins before startup by passing a comma-separated list of plugins.

    -e PLUGINS=ID[,ID]

In this example, it will install the Marvel plugin

    -e PLUGINS=elasticsearch/marvel/latest

Many more plugins [are available here](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-plugins.html#known-plugins). -->

### Publish As

Since the container gives the Elasticsearch software an isolated perspective of its networking, it will most likely advertise its published address with a container-internal IP address. This can be overridden with a physical networking name and port using:

```bash

-e PUBLISH_AS=DOCKERHOST:9301

```

If you use a cloud provider, you can use a private ip from the host as publish ip, you need only specify the provider with ```PROVIDER``` variable:

```bash

-e PROVIDER=AWS

```

the supported providers is:

* AWS

**more providers will be comming soon.**

_Author Note: I have yet to hit a case where this was actually necessary. Other
than the cosmetic weirdness in the logs, Elasticsearch seems to be quite tolerant._

### Node Name

Rather than use the randomly assigned node name, you can indicate a specific one using:

```bash

-e NODE_NAME=Docker

```

### Node Type

If you refer to [the Node section](https://www.elastic.co/guide/en/elasticsearch/reference/2.3/modules-node.html)
of the Elasticsearch reference guide, you'll find that there's three main types of nodes: master-eligible, data, and client.

In larger clusters it is important to dedicate a small number (>= 3) of master nodes. There are also cases where a large cluster may need dedicated gateway nodes that are neither master nor data nodes and purely operate as "smart routers" and have large amounts of CPU and memory to handle client requests and search-reduce.

To simplify all that, this image provides a `TYPE` variable to let you amongst these combinations. The choices are:

* (not set, the default) : the default node type which is both master-eligible and a data node

* `MASTER` : master-eligible, but holds no data. It is good to have three or more of these in a large cluster

* `DATA` : holds data and serves search/index requests. Scale these out for elastic-y goodness.

* `GATEWAY` : only operates as a client node or a "smart router". These are the ones whose HTTP port 9200 will need to be exposed

* `NO_MASTER` : only not operates master type, it is type gateway and type data together.

A [Docker Compose](https://docs.docker.com/compose/overview/) file will serve as a good example of these three node types:

```yml

gateway:
image: gerencio/elasticsearch:<version>
environment:
    UNICAST_HOSTS: master
    TYPE: GATEWAY
ports:
    - "9200:9200"

master:
image: gerencio/elasticsearch:<version>
environment:
    TYPE: MASTER
    MIN_MASTERS: 2

data:
image: gerencio/elasticsearch:<version>
environment:
    UNICAST_HOSTS: master
    TYPE: DATA

```

### Minimum Master Nodes

In combination with the `TYPE` variable above, you will also want to configure the minimum master nodes to [avoid split-brain](https://www.elastic.co/guide/en/elasticsearch/reference/2.3/modules-node.html#split-brain) during network outages.

The minimum, which can be calculated as `(master_eligible_nodes / 2) + 1`, can be set with the `MIN_MASTERS` variable.

Using the Docker Compose file above, a value of `2` is appropriate when scaling the cluster to 3 master nodes:

```bash

docker-compose scale master=3

```

### Auto transport/http discovery (simple mode)

By default we use eth0 interface for discovery ip and binding, you can change this using ```DISCOVER_IP```:

```bash

-e DISCOVER_IP=eth0

```

replacing `eth0` with another interface within the container, if needed.

### Auto transport/http discovery with EC2 discovery Mode

When using EC2, you can use the AWS internal api for discover others machines and discovery all nodes for the elastic
cluster.

You need to set two essencial variables: ```EC2_REGION``` (default: us-east-1) and ```PROVIDER```

the provider there not default value. so you need set to ```AWS``` like this:

```bash

-e PROVIDER=AWS -e EC2_REGION=us-west-2

```

for discovery you need some configurations.

You can use this in two modes:

* Tag mode: with tag mode, you can create array of tags like this:

```bash

-e EC2_TAG_KEY=role -e EC2_TAG_VALUE=es-master

```

Only hosts with this tags will be discovered for the cluster, we accept only *one* tag pair

* Security Group mode: with sg mode, you can create array of security groups like this:

```bash

-e EC2_SG_HOSTS=sg12345,sg9999

```

Only hosts in this security groups will be discovered for the cluster

### Auto transport/http discovery with Consul discovery Mode

When using Consul, you can use the consul api for discover all nodes from the cluster, by default we use health-check to join in the cluster.
you can use consul discovery with this option:

```bash

-e CONSUL_SERVICE_NAMES=es-data,es  -e CONSUL_SERVICE_HOST=172.17.0.1 -e CONSUL_SERVICE_PORT=8500

```

In this example we discover all nodes with services es-data.service.consul and es.service.consul

For better use of the cluster, we suggest use of ```PUBLISH_AS``` or ```PROVIDER``` setup, because you will join
with internal ip of conteiner, like 172.17.0.1, and if the conteiners is located in diferent hosts, the cluster will not join.

using consul agent located in ```http://CONSUL_SERVICE_HOST:CONSUL_SERVICE_PORT```

We use this [plugin](https://github.com/vvanholl/elasticsearch-consul-discovery).

### Heap size and other JVM options

By default this image will run Elasticsearch with a Java heap size of *75% from all host memory (without swap)*.

You can change this manage the variable `PCT_HEAP_MEM`:
the value is between 1 and 99, with default value equals 75, the value means % of memory host used
by the java heap.

For example, if server have 16 GB of physical memory:

```bash

-e PCT_HEAP_MEM="50"

```

The conteiner will use 8GB of memory.

If that value or any other JVM options need to be adjusted manually, then replace the `ES_JAVA_OPTS`
environment variable.

For example, this would allow for the use of 16 GB of heap:

```bash

-e ES_JAVA_OPTS="-Xms16g -Xmx16g"

```

Refer to [this page](https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html)
for more information about why both the minimum and maximum sizes were set to
the same value.