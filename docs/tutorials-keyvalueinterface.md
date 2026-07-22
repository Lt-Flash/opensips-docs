---
title: "Key-Value Interface"
description: "Use the OpenSIPS Key-Value interface to access local and external cache backends through a common script API."
---

The Key-Value interface in OpenSIPS provides a common way for the routing script and modules to access different key-value backends. It is similar in purpose to the DB interface: the script uses a stable API, while the loaded `cachedb_*` module provides the actual storage implementation.

Modules that connect to local or external storage backends implement the Key-Value API and expose one or more connection identifiers. These identifiers are passed as the first parameter to the `cache_*` script functions.

---

## Available cachedb_* Backends

Current OpenSIPS cachedb backends are `cachedb_cassandra`, `cachedb_couchbase`, `cachedb_dynamodb`, `cachedb_local`, `cachedb_memcached`, `cachedb_mongodb`, `cachedb_redis` and `cachedb_sql`.

### cachedb_local

The [cachedb_local module](/modules/devel/cachedb_local) stores data in OpenSIPS shared memory using local hash tables called collections. It is the fastest option and has no external service dependency, but data is normally lost on restart unless local cache clustering or restart persistency is configured.

If no URL is configured, OpenSIPS provides a default `local` storage ID. Additional URLs can expose named groups such as `local:group2`.

```opensips
loadmodule "cachedb_local.so"

modparam("cachedb_local", "cachedb_url", "local://")
modparam("cachedb_local", "cachedb_url", "local:group2:///collection2")

cache_store("local", "key", "value");
cache_store("local:group2", "key", "value");
```

### cachedb_memcached

The [cachedb_memcached module](/modules/devel/cachedb_memcached) stores data in Memcached servers. It keeps cache memory outside OpenSIPS and survives an OpenSIPS restart, but values are still volatile if the Memcached service is restarted or flushed.

The URL prefix becomes the script storage ID, such as `memcached:group1`.

```opensips
loadmodule "cachedb_memcached.so"

modparam("cachedb_memcached", "cachedb_url",
	"memcached:group1://localhost:11211/")

cache_store("memcached:group1", "key", "value");
```

### cachedb_redis

The [cachedb_redis module](/modules/devel/cachedb_redis) stores data in Redis. It supports standalone Redis servers, Redis Cluster topology discovery, authentication, Unix sockets, TLS and Redis raw queries.

The URL prefix becomes the script storage ID, such as `redis:group1` or `redis:cluster1`.

```opensips
loadmodule "cachedb_redis.so"

modparam("cachedb_redis", "cachedb_url",
	"redis:group1://localhost:6379/")

cache_store("redis:group1", "key", "value");
```

### cachedb_cassandra

The [cachedb_cassandra module](/modules/devel/cachedb_cassandra) stores data in Cassandra. It is useful when persistent, distributed key-value data is needed across OpenSIPS instances.

Cassandra requires the database part of the URL to identify the keyspace and table. If integer counter operations are used, the URL can also include a dedicated counter table.

```opensips
loadmodule "cachedb_cassandra.so"

modparam("cachedb_cassandra", "cachedb_url",
	"cassandra:cluster1://10.0.0.10,10.0.0.15/keyspace.keys.counters")

cache_store("cassandra:cluster1", "key", "value");
```

### cachedb_couchbase

The [cachedb_couchbase module](/modules/devel/cachedb_couchbase) stores data in Couchbase buckets through `libcouchbase`. It can point to a single server or multiple Couchbase nodes.

The URL prefix becomes the script storage ID, such as `couchbase:cluster1`.

```opensips
loadmodule "cachedb_couchbase.so"

modparam("cachedb_couchbase", "cachedb_url",
	"couchbase:cluster1://host1:11210,host2:11210/my_bucket")

cache_store("couchbase:cluster1", "key", "value");
```

### cachedb_mongodb

The [cachedb_mongodb module](/modules/devel/cachedb_mongodb) stores data in MongoDB collections. Its URL syntax follows the MongoDB connection string format and it supports MongoDB raw queries.

The default storage ID is `mongodb`. A URL with a group exposes a storage ID such as `mongodb:cluster`.

```opensips
loadmodule "cachedb_mongodb.so"

modparam("cachedb_mongodb", "cachedb_url",
	"mongodb:cluster://localhost,10.0.0.10:27017/opensipsDB.dialog")

cache_store("mongodb:cluster", "key", "value");
```

### cachedb_sql

The [cachedb_sql module](/modules/devel/cachedb_sql) stores key-value data in a regular SQL database through the OpenSIPS DB interface. It is useful when an SQL backend is already available and persistence is more important than cache speed.

The URL format is `sql:connection-id-dburl`, and the script storage ID is the `sql:*` prefix before `://`.

```opensips
loadmodule "db_mysql.so"
loadmodule "cachedb_sql.so"

modparam("cachedb_sql", "cachedb_url",
	"sql:main-mysql://opensips:opensipsrw@localhost/opensips")

cache_store("sql:main-mysql", "key", "value");
```

### cachedb_dynamodb

The [cachedb_dynamodb module](/modules/devel/cachedb_dynamodb) stores data in Amazon DynamoDB, or in a local DynamoDB instance for testing. It supports the regular key-value operations and integer counter operations.

The default storage ID is `dynamodb`.

```opensips
loadmodule "cachedb_dynamodb.so"

modparam("cachedb_dynamodb", "cachedb_url",
	"dynamodb:///opensips_cache?region=eu-central-1")

cache_store("dynamodb", "key", "value");
```

---

## Core API

The OpenSIPS core API operates on a storage ID and an attribute name. The storage ID selects the backend connection, while the attribute name selects the key inside that backend.

The generic Key-Value API includes:
* [cache_store()](/manual/devel/script-corefunctions) - stores or replaces a string value, with an optional timeout
* [cache_fetch()](/manual/devel/script-corefunctions) - fetches a string value into a script variable
* [cache_remove()](/manual/devel/script-corefunctions) - removes a key
* [cache_counter_fetch()](/manual/devel/script-corefunctions) - fetches an integer counter value
* [cache_add()](/manual/devel/script-corefunctions) - increments an integer counter and optionally returns the new value
* [cache_sub()](/manual/devel/script-corefunctions) - decrements an integer counter and optionally returns the new value
* [cache_raw_query()](/manual/devel/script-corefunctions) - runs a backend-specific raw query when the selected module supports it

String values and integer counters are separate concepts in some backends. For example, Cassandra requires a dedicated counter table for `cache_add()`, `cache_sub()` and `cache_counter_fetch()`, while SQL uses a dedicated counter column and currently supports counter operations only with MySQL. Check the selected module documentation before using integer operations.

`cache_raw_query()` is backend-specific. The Redis and MongoDB backends document raw-query support; other backends may not implement it.

```opensips
cache_store("local", "total_minutes_$fU", "$var(minutes)", 1200);

if (cache_fetch("local", "total_minutes_$fU", $var(total_minutes)))
	xlog("[$ci] cached total minutes: $var(total_minutes)\n");

cache_add("redis:stats", "calls_$fU", 1, 3600, $var(new_call_count));
cache_counter_fetch("redis:stats", "calls_$fU", $var(call_count));
cache_sub("redis:stats", "credit_$fU", 5, 0, $var(new_credit));
cache_raw_query("redis:stats", "PING", "$avp(redis_reply)");

cache_remove("local", "total_minutes_$fU");
```

---

## Use Cases and Examples

### Password Caching for DB Authentication

After a successful database authentication, OpenSIPS can store the subscriber password in a cache backend for a limited time. The next authentication attempt first checks the cache. If the password is available and not expired, OpenSIPS authenticates from script variables without a database lookup.

Because passwords for multiple users are cached at the same time, the cache key must include the authenticated user. For REGISTER authentication, the example below uses the To URI, resulting in keys such as `passwd_$tu`.

#### Logic

* REGISTER request received
* check whether `passwd_$tu` exists in cache
* if it exists, load it into `$var(auth_password)` and run `pv_www_authorize()`
* if it does not exist, run `www_authorize()` against the database
* after successful database authentication, store the loaded password as `passwd_$tu`

#### Script Example

A normal database-backed REGISTER authentication block looks like this:

```opensips
if (!www_authorize("", "subscriber")) {
	www_challenge("", "auth");
	exit;
}
```

With password caching, the configuration can look like this:

```opensips
loadmodule "db_mysql.so"
loadmodule "signaling.so"
loadmodule "auth.so"
loadmodule "auth_db.so"
loadmodule "cachedb_local.so"

modparam("auth", "username_spec", "$var(auth_user)")
modparam("auth", "password_spec", "$var(auth_password)")
modparam("auth", "calculate_ha1", 1)

modparam("auth_db", "calculate_ha1", 1)
modparam("auth_db", "password_column", "password")
modparam("auth_db", "db_url", "mysql://opensips:opensipsrw@localhost/opensips")
modparam("auth_db", "load_credentials", "$avp(cached_password)=password")
```

The `auth_db` module loads the database password into the named AVP `$avp(cached_password)` after a successful `www_authorize()` call. The `auth` module uses `$var(auth_user)` and `$var(auth_password)` when `pv_www_authorize()` validates credentials from cached data.

```opensips
route[AUTH_REGISTER] {
	if (cache_fetch("local", "passwd_$tu", $var(auth_password))) {
		$var(auth_user) = $tU;

		xlog("[$ci] using cached password for $tu\n");

		if (!pv_www_authorize("")) {
			if ($rc == -2)
				cache_remove("local", "passwd_$tu");

			www_challenge("", "auth");
			exit;
		}
	} else {
		if (!www_authorize("", "subscriber")) {
			www_challenge("", "auth");
			exit;
		}

		xlog("[$ci] storing password in cache for $tu\n");

		cache_store("local", "passwd_$tu", "$avp(cached_password)", 1200);
	}
}
```

References:
* [auth module](/modules/devel/auth)
* [auth_db module](/modules/devel/auth_db)
* [cachedb_local module](/modules/devel/cachedb_local)

#### Performance Improvement

Assuming a 30 minute registration period and one call every 20 minutes, without caching there are five authentication database queries per hour per user.

For 1000 users, that means `1000 * 3 * 24 = 72000` authentication queries per day.

With a two hour cache lifetime, each user needs one database query every two hours.

For the same 1000 users, that means `1000 * 0.5 * 24 = 12000` authentication queries per day.

This reduces authentication query load by about 83%.

### Simple Distributed Billing

Multiple OpenSIPS servers can serve the same domain for redundancy or load balancing. If each node reads and writes user credit independently from a local database, billing decisions can become inconsistent.

Using a shared key-value backend such as Redis lets all OpenSIPS nodes read and update the same credit counter. Because `cache_add()` and `cache_sub()` are atomic in suitable backends, each server can safely update credit after a call and check the current credit before allowing a new INVITE.

#### Prerequisites

Run a Redis Server or Redis Cluster reachable by all OpenSIPS nodes, then configure `cachedb_redis` with a shared storage ID.

#### Script Example

Rejecting calls due to low credit and updating credit after each call can be done as follows:

```opensips
loadmodule "dialog.so"
loadmodule "cachedb_redis.so"

modparam("cachedb_redis", "cachedb_url",
	"redis:billing://192.168.2.1:6379/")

route {
	if (has_totag()) {
		if (is_method("BYE")) {
			$var(cost) = $DLG_lifetime * $(dlg_val(cost){s.int});
			cache_sub("redis:billing", "credit_$fU", $var(cost), 0,
				$var(remaining_credit));

			xlog("[$ci] remaining credit for $fU is $var(remaining_credit)\n");
		}

		# handle sequential requests here
		exit;
	}

	if (is_method("INVITE")) {
		if (!cache_counter_fetch("redis:billing", "credit_$fU",
			$var(user_credit))) {
			sl_send_reply(403, "Forbidden - No credit record");
			exit;
		}

		if ($var(user_credit) <= 1) {
			xlog("[$ci] user $fU does not have enough credit\n");
			sl_send_reply(403, "Forbidden - Not enough credit");
			exit;
		}

		create_dialog();
		$dlg_val(cost) = "3";
	}

	# continue request processing here
}
```
