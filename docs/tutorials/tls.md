---
title: "TLS"
author: "by Ionut-Razvan Ionita"
description: "Configuring TLS can sometimes be time consuming, most times because of badly generated or used certificates. This tutorial provides an OpenSIPS TLS setup."
---

## Introduction

This tutorial was built using OpenSIPS 4.1.

It shows a basic TLS setup that can be used as a known-good starting point for more advanced deployments. It covers certificate generation, file-based TLS domains, optional database provisioning and basic troubleshooting.

The examples use the following lab values:

```text
OpenSIPS TLS socket: 192.0.2.10:5061
Remote TLS UAS:      198.51.100.20:5061
TLS files:           /etc/opensips/tls
SIP domain:          example.com
```

Replace them with your own addresses and domains.

## Generating Certificates

Before configuring OpenSIPS, create a local CA and at least one certificate signed by that CA. For a lab setup, the [`opensips-cli` TLS module](https://github.com/OpenSIPS/opensips-cli/blob/master/docs/modules/tls.md) can generate these files directly.

Create `/etc/opensips-cli.cfg`:

```ini
[default]
tls_ca_dir: /etc/opensips/tls/rootCA
tls_ca_cert_file: cacert.pem
tls_ca_key_file: private/cakey.pem
tls_ca_overwrite: yes
tls_ca_common_name: example.com
tls_ca_country: RO
tls_ca_state: Bucharest
tls_ca_locality: Bucharest
tls_ca_organisation: OpenSIPS
tls_ca_organisational_unit: Project
tls_ca_notafter: 315360000
tls_ca_key_size: 4096
tls_ca_md: SHA256

tls_user_cert_file: cert.pem
tls_user_key_file: privkey.pem
tls_user_calist_file: calist.pem
tls_user_overwrite: yes
tls_user_cacert: /etc/opensips/tls/rootCA/cacert.pem
tls_user_cakey: /etc/opensips/tls/rootCA/private/cakey.pem
tls_user_country: RO
tls_user_state: Bucharest
tls_user_locality: Bucharest
tls_user_organisation: OpenSIPS
tls_user_organisational_unit: Project
tls_user_notafter: 315360000
tls_user_key_size: 4096
tls_user_md: SHA256
```

Create the CA:

```bash
opensips-cli -x tls rootCA
```

Create the certificate OpenSIPS presents on incoming TLS connections:

```bash
opensips-cli \
    -o tls_user_dir=/etc/opensips/tls/server \
    -o tls_user_common_name=sip.example.com \
    -x tls userCERT
```

Create a separate certificate for OpenSIPS to present when it acts as a TLS client:

```bash
opensips-cli \
    -o tls_user_dir=/etc/opensips/tls/client \
    -o tls_user_common_name=client.example.com \
    -x tls userCERT
```

For quick interoperability tests you may use the test certificates shipped with the OpenSIPS packages, but do not use them in production.

In the examples below, `$CERT_DIR` means `/etc/opensips/tls`.

## Script Example

### TLS Modules and Socket

OpenSIPS TLS support uses three parts:

* `proto_tls` for the SIP-over-TLS transport.
* `tls_mgm` for TLS domains, certificates and TLS parameters.
* One TLS backend module, such as `tls_openssl` or `tls_wolfssl`.

Load the TLS backend before `tls_mgm`, then load `proto_tls`:

```opensips
socket=tls:192.0.2.10:5061

loadmodule "tls_openssl.so"
loadmodule "tls_mgm.so"
loadmodule "proto_tls.so"
```

The newer socket syntax is `socket=proto:ip:port`. Older examples may use `listen=...`; update them when moving to newer style configuration.

### File-Based TLS Domains

The example has two TLS domains:

* `sv_dom`: server-side domain used for incoming TLS connections to `192.0.2.10:5061`.
* `cl_dom`: client-side domain used when OpenSIPS opens a TLS connection to `198.51.100.20:5061`.

In newer OpenSIPS configurations, `server_domain` and `client_domain` only define the domain name. Address matching is configured separately with `match_ip_address`, and domain-scoped values use the `[domain]value` syntax.

```opensips
modparam("tls_mgm", "server_domain", "sv_dom")
modparam("tls_mgm", "match_ip_address", "[sv_dom]192.0.2.10:5061")
modparam("tls_mgm", "certificate", "[sv_dom]/etc/opensips/tls/server/cert.pem")
modparam("tls_mgm", "private_key", "[sv_dom]/etc/opensips/tls/server/privkey.pem")
modparam("tls_mgm", "ca_list", "[sv_dom]/etc/opensips/tls/server/calist.pem")
modparam("tls_mgm", "tls_method", "[sv_dom]TLSv1_2-TLSv1_3")
modparam("tls_mgm", "verify_cert", "[sv_dom]0")
modparam("tls_mgm", "require_cert", "[sv_dom]0")

modparam("tls_mgm", "client_domain", "cl_dom")
modparam("tls_mgm", "match_ip_address", "[cl_dom]198.51.100.20:5061")
modparam("tls_mgm", "certificate", "[cl_dom]/etc/opensips/tls/client/cert.pem")
modparam("tls_mgm", "private_key", "[cl_dom]/etc/opensips/tls/client/privkey.pem")
modparam("tls_mgm", "ca_list", "[cl_dom]/etc/opensips/tls/client/calist.pem")
modparam("tls_mgm", "tls_method", "[cl_dom]TLSv1_2-TLSv1_3")
modparam("tls_mgm", "verify_cert", "[cl_dom]0")
modparam("tls_mgm", "require_cert", "[cl_dom]0")
```

The `verify_cert` and `require_cert` values are disabled here only to keep a lab handshake simple. For production, keep peer verification enabled and provision the proper CA chain.

### Full TLS Fragment

The following fragment contains only the TLS-related parts. Add it to a complete OpenSIPS routing script that loads your normal SIP modules.

```opensips
socket=tls:192.0.2.10:5061

loadmodule "tls_openssl.so"
loadmodule "tls_mgm.so"
loadmodule "proto_tls.so"

modparam("tls_mgm", "server_domain", "sv_dom")
modparam("tls_mgm", "match_ip_address", "[sv_dom]192.0.2.10:5061")
modparam("tls_mgm", "certificate", "[sv_dom]/etc/opensips/tls/server/cert.pem")
modparam("tls_mgm", "private_key", "[sv_dom]/etc/opensips/tls/server/privkey.pem")
modparam("tls_mgm", "ca_list", "[sv_dom]/etc/opensips/tls/server/calist.pem")
modparam("tls_mgm", "tls_method", "[sv_dom]TLSv1_2-TLSv1_3")
modparam("tls_mgm", "verify_cert", "[sv_dom]0")
modparam("tls_mgm", "require_cert", "[sv_dom]0")

modparam("tls_mgm", "client_domain", "cl_dom")
modparam("tls_mgm", "match_ip_address", "[cl_dom]198.51.100.20:5061")
modparam("tls_mgm", "certificate", "[cl_dom]/etc/opensips/tls/client/cert.pem")
modparam("tls_mgm", "private_key", "[cl_dom]/etc/opensips/tls/client/privkey.pem")
modparam("tls_mgm", "ca_list", "[cl_dom]/etc/opensips/tls/client/calist.pem")
modparam("tls_mgm", "tls_method", "[cl_dom]TLSv1_2-TLSv1_3")
modparam("tls_mgm", "verify_cert", "[cl_dom]0")
modparam("tls_mgm", "require_cert", "[cl_dom]0")
```

To send traffic to the remote UAS over TLS from script, relay to a TLS destination, for example:

```opensips
route[to_tls_uas] {
    $du = "sip:198.51.100.20:5061;transport=tls";

    if (!t_relay())
        send_reply(500, "Relay failed");
}
```

## Database Provisioning

`tls_mgm` domains can also be provisioned from SQL. This is useful when certificates are managed outside the OpenSIPS script or when you want to reload TLS domains without editing the config file.

Create the OpenSIPS database with the `tls_mgm` table. If you are creating a new database, include `tls_mgm` in `database_modules` or use `ALL`:

```ini
[default]
database_admin_url: mysql://root@localhost
database_url: mysql://opensips:opensipsrw@localhost/opensips
database_name: opensips
database_modules: tls_mgm
```

```bash
opensips-cli -x database create
```

If the database already exists, add only the `tls_mgm` table:

```bash
opensips-cli -x database add tls_mgm
```

Load the SQL module and point `tls_mgm` at the database:

```opensips
socket=tls:192.0.2.10:5061

loadmodule "db_mysql.so"
loadmodule "tls_openssl.so"
loadmodule "tls_mgm.so"
loadmodule "proto_tls.so"

modparam("tls_mgm", "db_url", "mysql://opensips:opensipsrw@localhost/opensips")
```

For MySQL/MariaDB, the certificate, private key, CA list and DH parameters are stored as BLOB values. The `type` value is `2` for a server domain and `1` for a client domain:

```sql
INSERT INTO tls_mgm
    (domain, match_ip_address, match_sip_domain, type, method,
     verify_cert, require_cert, certificate, private_key, ca_list)
VALUES
    ('sv_dom', '192.0.2.10:5061', NULL, 2, 'TLSv1_2-TLSv1_3',
     0, 0,
     LOAD_FILE('/etc/opensips/tls/server/cert.pem'),
     LOAD_FILE('/etc/opensips/tls/server/privkey.pem'),
     LOAD_FILE('/etc/opensips/tls/server/calist.pem')),
    ('cl_dom', '198.51.100.20:5061', NULL, 1, 'TLSv1_2-TLSv1_3',
     0, 0,
     LOAD_FILE('/etc/opensips/tls/client/cert.pem'),
     LOAD_FILE('/etc/opensips/tls/client/privkey.pem'),
     LOAD_FILE('/etc/opensips/tls/client/calist.pem'));
```

`LOAD_FILE()` requires the database server to be allowed to read the files. If your database blocks this through `secure_file_priv` or filesystem permissions, import the BLOB values using your database tooling instead, or use file-based script parameters.

After changing DB-provisioned domains, reload them with:

```bash
opensips-cli -x mi tls_mgm:reload
```

## Troubleshooting

### Check the TLS Socket

After OpenSIPS starts, verify that it listens on the TLS socket:

```bash
ss -ltnp | grep 5061
```

### Test with OpenSSL

Use `openssl s_client` to validate the handshake:

```bash
openssl s_client \
    -showcerts \
    -connect 192.0.2.10:5061 \
    -servername sip.example.com
```

For mutual TLS tests, also provide the client certificate and CA:

```bash
openssl s_client \
    -showcerts \
    -connect 192.0.2.10:5061 \
    -servername sip.example.com \
    -cert /etc/opensips/tls/client/cert.pem \
    -key /etc/opensips/tls/client/privkey.pem \
    -CAfile /etc/opensips/tls/client/calist.pem
```

### Inspect Loaded TLS Domains

Use the `tls_mgm:list` MI command to see the domains OpenSIPS loaded:

```bash
opensips-cli -x mi tls_mgm:list
```

Check that `sv_dom` is a server domain, `cl_dom` is a client domain and that the match addresses are the expected `ip:port` values.

### Wireshark Tracing

Wireshark can inspect TLS handshakes and, for older RSA key-exchange captures, decrypt traffic if it has the private key. Modern TLS deployments often use forward-secret key exchanges, so packet decryption may require key log material from the TLS endpoint instead of only the private key.

For handshake debugging, filter on the OpenSIPS TLS socket:

```text
tcp.port == 5061
```

## Legacy 2.1 Example

OpenSIPS 2.1 kept TLS certificate and domain parameters under `proto_tls`. It did not support database-backed TLS provisioning, so certificates and domains had to be defined directly in the script.

The example below is preserved as migration context only. For newer OpenSIPS versions, use the `tls_openssl`/`tls_wolfssl` + `tls_mgm` + `proto_tls` layout shown above.

```opensips
# OpenSIPS 2.1 legacy syntax
listen=tls:<your-ip-address>:<port>

loadmodule "proto_tls.so"

modparam("proto_tls", "verify_cert", "0")
modparam("proto_tls", "require_cert", "0")
modparam("proto_tls", "tls_method", "TLSv1")

modparam("proto_tls", "certificate", "$CERT_DIR/rootCA/cacert.pem")
modparam("proto_tls", "private_key", "$CERT_DIR/rootCA/private/cakey.pem")
modparam("proto_tls", "ca_list", "$CERT_DIR/rootCA/cacert.pem")
modparam("proto_tls", "ca_dir", "$CERT_DIR/rootCA/")

modparam("proto_tls", "server_domain", "sv_dom=<your-ip-address>:<port>")
modparam("proto_tls", "certificate", "sv_dom:$CERT_DIR/rootCA/cacert.pem")
modparam("proto_tls", "private_key", "sv_dom:$CERT_DIR/rootCA/private/cakey.pem")
modparam("proto_tls", "ca_list", "sv_dom:$CERT_DIR/rootCA/cacert.pem")

modparam("proto_tls", "client_domain", "cl_dom=<UAS-ip-address>:<port>")
modparam("proto_tls", "certificate", "cl_dom:$CERT_DIR/user/user-cert.pem")
modparam("proto_tls", "private_key", "cl_dom:$CERT_DIR/user/user-privkey.pem")
modparam("proto_tls", "ca_list", "cl_dom:$CERT_DIR/user/user-calist.pem")
```

When migrating that snippet to newer versions:

* Move TLS domain settings from `proto_tls` to `tls_mgm`.
* Load one TLS backend module before `tls_mgm`.
* Replace `listen=tls:...` examples with `socket=tls:...`.
* Replace `sv_dom:path` values with `[sv_dom]path`.
* Define domain names with `server_domain` / `client_domain`, then match addresses with `match_ip_address`.
