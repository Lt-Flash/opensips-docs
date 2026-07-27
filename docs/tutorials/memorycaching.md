---
title: "Memory Caching"
description: "Use OpenSIPS memory caching support to store runtime data and reduce repeated database lookups, with a password caching example for digest authentication."
---

## MemCache

The **Memory Caching** support in **OpenSIPS** provides a way to cache runtime data from the routing script. Cached data is globally available to the script and shared between all **OpenSIPS** processes that use the same cache backend.

The goal is to store custom values for later use, especially data that is expensive to fetch and does not need to be read from the database on every request. Database queries are a common bottleneck in SIP platforms, so caching stable data can significantly reduce backend load.

---

## MemCache Design

The memory caching support in **OpenSIPS** is exposed through the generic Key-Value API. The script uses the same functions regardless of the selected backend, while modules provide the actual storage implementation.

The core API offers the basic cache operations:
* store - [cache_store()](/manual/devel/script-corefunctions#cache_store)
* fetch - [cache_fetch()](/manual/devel/script-corefunctions#cache_fetch)
* remove - [cache_remove()](/manual/devel/script-corefunctions#cache_remove)

When storing a value, the script can also set a lifetime for the attribute. This allows OpenSIPS to automatically expire cached data and periodically refresh it from the authoritative source.

Cache backends are provided by modules. For local shared-memory caching, use the [cachedb_local module](/modules/devel/cachedb_local). Other deployments can use external cache backends, such as Redis or Memcached, through their corresponding cachedb modules.

---

## MemCache Usage: Password Caching for DB Authentication

The best way to describe memory caching is with a practical example: reducing the number of database queries caused by subscriber authentication.

### The Idea

After a successful database authentication, OpenSIPS stores the subscriber password in cache for a limited time. The next authentication attempt first checks the cache. If the password is available and not expired, OpenSIPS authenticates directly from script variables, without a database lookup.

Because passwords for multiple users are cached at the same time, the cache attribute name must include the authenticated user. For REGISTER authentication, the example below uses the To URI, resulting in keys such as `passwd_$tu`.

### The Logic

* REGISTER request received
* check whether `passwd_$tu` exists in cache
* if it exists, load it into `$var(auth_password)` and run `pv_www_authorize()`
* if it does not exist, run `www_authorize()` against the database
* after successful database authentication, store the loaded password as `passwd_$tu`

### Script

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

The cache lifetime above is 1200 seconds, or 20 minutes. After that interval, the password is removed from cache and the next authentication refreshes it from the database.

References:
* [auth module](/modules/devel/auth)
* [auth_db module](/modules/devel/auth_db)
* [cachedb_local module](/modules/devel/cachedb_local)

---

### Improvements

#### Force Password Re-Fetching

When cached authentication fails with return code `-2`, `pv_www_authorize()` found a valid user but the password did not match. This can happen when the subscriber password changed in the database while the old value was still cached.

In that case, remove the cached password with `cache_remove()` and challenge the user again. The next request will miss the cache, run database authentication and store the new password value.

#### WWW and PROXY Authentication

The same approach works for both WWW authentication, typically used for REGISTER requests, and PROXY authentication, typically used for INVITE, MESSAGE and other in-dialog or out-of-dialog requests.

For WWW authentication, the authentication name is extracted from the To header, so build the cache key with the To URI or To user, such as `passwd_$tu` or `passwd_$tU`. For PROXY authentication, the authentication name is extracted from the From header, so use a From-based key, such as `passwd_$fu` or `passwd_$fU`.

### Performance Estimation

Assuming a 30 minute registration period and one call every 20 minutes, without caching there are five authentication database queries per hour per user.

For 1000 users, that means `1000 * 3 * 24 = 72000` authentication queries per day.

With a two hour cache lifetime, each user needs one database query every two hours.

For the same 1000 users, that means `1000 * 0.5 * 24 = 12000` authentication queries per day.

This reduces authentication query load by about 83%.
