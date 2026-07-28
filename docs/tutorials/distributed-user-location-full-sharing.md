---
title: "How To Configure a \"Full Sharing\" User Location Cluster"
author: "by Liviu Chircu"
description: "A deep-dive into the \"full sharing\" user location clustering strategy, with full-mesh data replication between cluster nodes."
---

## Description

*Tip:* For a broader view on the "full sharing" topology, see [this blog post](https://blog.opensips.org/2018/09/13/clustered-sip-user-location-the-full-sharing-topology/).


 ![full sharing](/public/uploads/full-sharing.png)


The *"full sharing"* clustering strategy for OpenSIPS 2.4+ user location implementations is a way of performing full-mesh data replication between the nodes of an OpenSIPS cluster.  Each node will hold the entire user location dataset, thus being able to serve lookups for any SIP UA registered to the cluster.  This type of clustering offers:

* high availability (any cluster node can properly serve the incoming SIP traffic)
* distributed NAT pinging support (NAT pinging origination can be spread across cluster nodes)
* restart persistency for all cluster nodes
* good horizontal scalability, capped by the maximum amount of data that a single node can handle


> [!IMPORTANT]
> A mandatory requirement of the *full sharing* clustering strategy is that **any node must be able to route to any registered SIP UA**. With simple *full sharing* setups, such as active/passive, this can be achieved by using a shared virtual IP address between the two nodes. If dealing with larger cluster sizes or if the endpoints register via TCP/TLS, then a front-ending entity (e.g. a SIP load balancer) must be placed in front of the cluster, with enabled [Path header](https://tools.ietf.org/html/rfc3327) support, so any network routing restrictions are alleviated.


Building upon this setup, the [federated user location](/tutorials/distributed-user-location-federation) clustering strategy ensures similar features as above, except it will not replicate user location data across different points of presence, allowing you to scale each POP according to the size of its subscriber pool.

## Active/passive "full sharing" setup

### Configuration

For the smallest possible setup (a 2-node active/passive with a virtual IP in front), you will need:

* two OpenSIPS instances
* a working shared/virtual IP between the instances (e.g. using *keepalived*, *vrrpd*, etc.)
* a MySQL instance, for provisioning


The relevant *opensips.cfg* sections:


```opensips

listen = sip:10.0.0.150 # virtual IP (same on both nodes)
listen = bin:10.0.0.177

loadmodule "usrloc.so"
modparam("usrloc", "working_mode_preset", "full-sharing-cluster")
modparam("usrloc", "location_cluster", 1)

loadmodule "clusterer.so"
modparam("clusterer", "my_node_id", 1) # node number #1
modparam("clusterer", "db_url", "mysql://opensips:opensipsrw@localhost/opensips")

loadmodule "proto_bin.so"

```

### Provisioning

```sql

INSERT INTO clusterer(id, cluster_id, node_id, url, state, no_ping_retries, priority, sip_addr, flags, description) VALUES \
(NULL, 1, 1, 'bin:10.0.0.177', 1, 3, 50, NULL, 'seed', NULL), \
(NULL, 1, 2, 'bin:10.0.0.178', 1, 3, 50, NULL, NULL, NULL);

```

| **id** | **cluster id** | **node_id** | **url** | **state** | **no_ping_retries** | **priority** | **sip_addr** | **flags** | **description** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  14 | 1 | 1 | bin:10.0.0.177 | 1 | 3 | 50 | NULL | seed | NULL |
|  15 | 1 | 2 | bin:10.0.0.178 | 1 | 3 | 50 | NULL | NULL | NULL |

**Native "full sharing" clusterer table**

### NAT pinging

Some setups require periodic SIP OPTIONS pings originated by the registrar towards some of the contacts in order to keep the NAT bindings alive. Here is an example configuration:

```opensips

loadmodule "nathelper.so"
modparam("nathelper", "natping_interval", 30)
modparam("nathelper", "sipping_from", "sip:pinger@localhost")
modparam("nathelper", "sipping_bflag", "SIPPING_ENABLE")
modparam("nathelper", "remove_on_timeout_bflag", "SIPPING_RTO")
modparam("nathelper", "max_pings_lost", 5)

```

We then enable these branch flags for some or all contacts before calling [save()](/modules/4-1/registrar#func_save):

```opensips

    ...
    setbflag("SIPPING_ENABLE");
    setbflag("SIPPING_RTO");

    if (!save("location"))
        sl_reply_error();
    ...

```


To prevent any "permission denied" error logs on the passive node that's trying to originate NAT pings, make sure to hook the [nh_enable_ping](/modules/4-1/nathelper#mi_nh_enable_ping) MI command into your active->passive and passive->active transitions of the VIP:

```bash

    opensips-mi nh_enable_ping 1 # run this on the machine that takes over the VIP (new active)
    opensips-mi nh_enable_ping 0 # run this on the machine that gives up the VIP (new passive)

```

## NoSQL "full sharing" cluster with a SIP front-end

This is the ultra-scalable version of the OpenSIPS user location, allowing you to support subscriber pool sizes exceeding the order of **millions**.  By letting an external, specialized database cluster manage all the registration data, we are able to decouple the SIP signaling and data storage systems.  This, in turn, allows each system to be scaled individually, in an efficient manner.

### Configuration

For the smallest possible setup, you will need:

* a SIP front-end proxy sitting in front of the cluster, with [SIP Path](https://tools.ietf.org/html/rfc3327) support
* two backend OpenSIPS instances, forming the cluster
* a NoSQL DB instance, such as Cassandra or MongoDB, to hold all registrations (you can upgrade it into a cluster later)
* a MySQL instance, for provisioning


On the backend layer (cluster instances), here are the relevant *opensips.cfg* sections:


```opensips

listen = sip:10.0.0.177
listen = bin:10.0.0.177

loadmodule "usrloc.so"
modparam("usrloc", "working_mode_preset", "full-sharing-cachedb-cluster")
modparam("usrloc", "location_cluster", 1)

# with Cassandra, make sure to create the keyspace and table beforehand:
# CREATE KEYSPACE IF NOT EXISTS opensips WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1'}  AND durable_writes = true;
# USE opensips;
# CREATE TABLE opensips.userlocation (
#     aor text,
#     aorhash int,
#     contacts map<text, frozen<map<text, text>>>,
#     PRIMARY KEY (aor));
loadmodule "cachedb_cassandra.so"
modparam("usrloc", "cachedb_url", "cassandra://10.0.0.180:9042/opensips.userlocation")

# with MongoDB, we don't need to create any database or collection...
loadmodule "cachedb_mongodb.so"
modparam("usrloc", "cachedb_url", "mongodb://10.0.0.180:27017/opensipsDB.userlocation")

loadmodule "clusterer.so"
modparam("clusterer", "my_node_id", 1) # node number #1
modparam("clusterer", "db_url", "mysql://opensips:opensipsrw@localhost/opensips")

loadmodule "proto_bin.so"

...

route {
    ...

    # store the registration into the NoSQL DB
    if (!save("location", "path-lazy,path-received")) {
        send_reply(500, "Server Internal Error");
        exit;
    }

    ...
}

```

### Provisioning

```sql

INSERT INTO clusterer(id, cluster_id, node_id, url, state, no_ping_retries, priority, sip_addr, flags, description) VALUES \
(NULL, 1, 1, 'bin:10.0.0.177', 1, 3, 50, NULL, 'seed', NULL), \
(NULL, 1, 2, 'bin:10.0.0.178', 1, 3, 50, NULL, NULL, NULL);

```

| **id** | **cluster id** | **node_id** | **url** | **state** | **no_ping_retries** | **priority** | **sip_addr** | **flags** | **description** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  14 | 1 | 1 | bin:10.0.0.177 | 1 | 3 | 50 | NULL | NULL | NULL |
|  15 | 1 | 2 | bin:10.0.0.178 | 1 | 3 | 50 | NULL | NULL | NULL |

**NoSQL "full sharing" clusterer table**

### Shared NAT pinging

```opensips

loadmodule "nathelper.so"
modparam("nathelper", "natping_interval", 30)
modparam("nathelper", "sipping_from", "sip:pinger@localhost")
modparam("nathelper", "sipping_bflag", "SIPPING_ENABLE")
modparam("nathelper", "remove_on_timeout_bflag", "SIPPING_RTO")
modparam("nathelper", "max_pings_lost", 5)

# partition pings across cluster nodes
modparam("usrloc", "shared_pinging", 1)

```

We then enable these branch flags for some or all contacts before calling [save()](/modules/4-1/registrar#func_save):

```opensips

    ...

    setbflag("SIPPING_ENABLE");
    setbflag("SIPPING_RTO");

    # store the registration, along with the Path header, into the NoSQL DB
    if (!save("location", "path-lazy,path-received")) {
        sl_reply_error();
        exit;
    }

    ...

```
