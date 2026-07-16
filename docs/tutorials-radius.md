---
title: "RADIUS"
description: "RADIUS is a client/server networking protocol that provides centralized Authentication, Authorization, and Accounting management for clients to connect and use a network service."
---

RADIUS is a client/server networking protocol that provides centralized Authentication, Authorization and Accounting (AAA) management.

After completing this tutorial, you will be able to:

* make a basic FreeRADIUS server configuration
* configure OpenSIPS as a RADIUS client
* use RADIUS-backed accounting and authentication from the OpenSIPS script
* add and use custom attributes in RADIUS messages
* send custom authentication and accounting requests with `aaa_radius`

The examples below use `radcli` as the RADIUS client library. `radcli` is the preferred library for newer OpenSIPS deployments; it is source-compatible with the older `freeradius-client` and `radiusclient-ng` libraries, and OpenSIPS tries to use it first when building the `aaa_radius` module.

## Installing The RADIUS Server

Install FreeRADIUS. On Debian-based systems:

```bash
apt-get update
apt-get install -y freeradius
```

FreeRADIUS configuration paths vary by distribution and version. On newer Debian packages they are usually under `/etc/freeradius/3.0/`; older packages used `/etc/freeradius/` directly.

## Installing The RADIUS Client

Install `radcli`. If you build OpenSIPS from source, also install the development package before building `aaa_radius`.

```bash
apt-get install -y radcli libradcli-dev
```

For older systems where `radcli` is not available, `freeradius-client` and `radiusclient-ng` may still work, but the examples in this tutorial use `radcli` paths:

```text
/etc/radcli/radiusclient.conf
/etc/radcli/servers
/etc/radcli/dictionary.opensips
```

## Configuring The RADIUS Client And Server

### Share The Same OpenSIPS Dictionary

OpenSIPS accounting and authentication use OpenSIPS-specific RADIUS attributes such as `Sip-Method`, `Sip-Rpid`, `Sip-Call-Duration` and `SIP-AVP`. Both `radcli` and FreeRADIUS must know these attributes.

Install or copy OpenSIPS' `dictionary.opensips` to a local path, for example:

```bash
cp /usr/share/opensips/dictionary.opensips /etc/radcli/dictionary.opensips
cp /usr/share/opensips/dictionary.opensips /etc/freeradius/3.0/dictionary.opensips
```

If your package installs the file elsewhere, use that path instead. In source trees, the file is available as `etc/dictionary.opensips`.

Tell `radcli` to load it from `/etc/radcli/radiusclient.conf`:

```text
dictionary /etc/radcli/dictionary.opensips
```

Tell FreeRADIUS to load the same dictionary. With FreeRADIUS 3.x on Debian, add this line to `/etc/freeradius/3.0/dictionary`:

```text
$INCLUDE /etc/freeradius/3.0/dictionary.opensips
```

### Configure The Shared Secret

Configure the RADIUS servers in `/etc/radcli/radiusclient.conf`:

```text
authserver localhost
acctserver localhost
servers /etc/radcli/servers
```

Configure the matching client-side secret in `/etc/radcli/servers`:

```text
localhost testing123
```

Configure the same secret in FreeRADIUS. On FreeRADIUS 3.x this is usually `/etc/freeradius/3.0/clients.conf`:

```text
client localhost {
    ipaddr = 127.0.0.1
    secret = testing123
}
```

### Enable Digest Authentication

If you want FreeRADIUS to process SIP digest authentication requests, enable the `digest` module and make sure the active site calls it from both `authorize {}` and `authenticate {}`.

On FreeRADIUS 3.x, the module is usually enabled by linking it into `mods-enabled`:

```bash
ln -s ../mods-available/digest /etc/freeradius/3.0/mods-enabled/digest
```

Then check the active site, usually `/etc/freeradius/3.0/sites-available/default`, and make sure `digest` is present in both sections:

```text
authorize {
    ...
    digest
    ...
}

authenticate {
    ...
    digest
    ...
}
```

### Configure Test Users And Reply Attributes

FreeRADIUS 3.x stores the default "users" file content in `/etc/freeradius/3.0/mods-config/files/authorize`. Older packages used `/etc/freeradius/users`.

Example:

```text
testdig Cleartext-Password := "opensips.cfg"
        Reply-Message = "OpenSIPS Rules!",
        SIP-AVP = "sems:ann-account_locked",
        Sip-Rpid = "sip:+407433360111@opensips.org"

DEFAULT Auth-Type := Accept
        Reply-Message = "OpenSIPS Rules!",
        SIP-AVP = "sems:ann-account_locked",
        Sip-Rpid = "sip:+407433360111@opensips.org"
```

The entries are tested sequentially until a match is found. `DEFAULT` matches any user, so keep it after any specific users.

## Using RADIUS From The OpenSIPS Script

OpenSIPS uses RADIUS through its generic AAA interface:

* `aaa_radius` provides the RADIUS AAA backend.
* `acc` uses AAA for RADIUS accounting.
* `auth_aaa` uses AAA for RADIUS-backed SIP authentication and authorization.

Load `aaa_radius` before the modules that use the AAA backend:

```opensips
loadmodule "aaa_radius.so"
loadmodule "acc.so"
loadmodule "auth.so"
loadmodule "auth_aaa.so"
```

## Accounting

Accounting tracks SIP transactions or dialogs for billing, reporting or auditing. FreeRADIUS accounting logs are usually stored under `/var/log/freeradius/radacct/`.

Configure ACC to use RADIUS:

```opensips
modparam("acc", "aaa_url", "radius:/etc/radcli/radiusclient.conf")
modparam("acc", "service_type", 15)
```

The `aaa_url` value contains the AAA protocol name and the RADIUS client configuration path, separated by `:`.

Enable accounting in the request route with `do_accounting()`:

```opensips
if (is_method("INVITE") && !has_totag()) {
    do_accounting("aaa", "cdr|missed|failed");
}
```

You can still send an explicit one-shot accounting record with `acc_aaa_request()`:

```opensips
acc_aaa_request("403 Destination not allowed");
```

### Extra Accounting Attributes

Use `extra_fields` for additional RADIUS accounting attributes. For AAA accounting, the right-hand side after `->` must be an attribute known by the RADIUS dictionary.

For example, add these local attributes to the dictionary loaded by both `radcli` and FreeRADIUS:

```text
ATTRIBUTE Sip-Via      240 string
ATTRIBUTE Sip-Email    241 string
ATTRIBUTE Sip-Bcontact 242 string
```

Then map OpenSIPS accounting fields to those attributes:

```opensips
modparam("acc", "extra_fields",
    "aaa: via->Sip-Via; email->Sip-Email; bcontact->Sip-Bcontact")

route {
    $acc_extra(via) = $hdr(Via);
    $acc_extra(email) = $avp(email);
    $acc_extra(bcontact) = $ct;
}
```

Make sure any extra attributes you use are present in both the `radcli` and FreeRADIUS dictionaries.

## Authentication And Authorization

`auth_aaa` passes SIP digest credentials to the AAA backend and checks the RADIUS result. The RADIUS reply may also include attributes such as `SIP-AVP` and `Sip-Rpid`.

Configure `auth_aaa` to use RADIUS:

```opensips
modparam("auth_aaa", "aaa_url", "radius:/etc/radcli/radiusclient.conf")
```

Example usage:

```opensips
if (!aaa_www_authorize("opensips.org")) {
    www_challenge("opensips.org", "auth");
    exit;
}
```

Proxy authentication examples:

```opensips
if (!aaa_proxy_authorize("")) {
    proxy_challenge("", "auth");
    exit;
}

if (!aaa_proxy_authorize($pd, $pU)) {
    proxy_challenge($pd, "auth");
    exit;
}
```

In the second example, the realm and URI user are taken from the `P-Preferred-Identity` domain and user pseudo-variables.

## Custom RADIUS Queries

The `aaa_radius` module can send custom RADIUS authentication and accounting requests directly from the script and can extract selected attributes from authentication replies.

Configure the `aaa_radius` client file:

```opensips
modparam("aaa_radius", "radius_config", "/etc/radcli/radiusclient.conf")
```

### Custom RADIUS Attributes

RADIUS dictionaries must be consistent between OpenSIPS' RADIUS client and FreeRADIUS. For non-vendor-specific attributes, use values below 256. For vendor-specific attributes, use the standard Vendor-Specific Attribute encapsulation and define the same vendor and attribute names on both sides.

Add OpenSIPS-specific or local custom attributes to the dictionary file loaded by both sides. For example:

```text
ATTRIBUTE Custom-SIP-Reason 240 string
```

### Defining Attribute Sets

Custom queries use named sets. A set maps RADIUS attributes to OpenSIPS pseudo-variables or AVPs:

```text
set_name = ( Attribute-Name-1 = variable1 [, Attribute-Name-2 = variable2 ]* )
```

The left-hand side must be an attribute known by the RADIUS dictionary. The right-hand side must be an OpenSIPS variable or AVP.

Example:

```opensips
modparam("aaa_radius", "sets",
    "auth_in = (User-Name=$var(user), Sip-Group=$var(group), Service-Type=$var(type))")

modparam("aaa_radius", "sets",
    "auth_out = (Reply-Message=$var(reply), Sip-Rpid=$avp(rpid))")
```

### Custom Accounting Queries

Use `radius_send_acct(input_set_name)` to send a custom accounting request:

```opensips
radius_send_acct("auth_in");
```

Accounting replies do not carry the output attributes used by authentication replies, so only one set is needed.

### Custom Authentication Queries

Use `radius_send_auth(input_set_name, output_set_name)` to send a custom authentication request and fetch selected reply attributes:

```opensips
if (radius_send_auth("auth_in", "auth_out")) {
    xlog("RADIUS reply: $var(reply), RPID: $avp(rpid)\n");
}
```

## Useful Links

* [FreeRADIUS](https://freeradius.org/)
* [radcli](https://github.com/radcli/radcli)
* [OpenSIPS aaa_radius module](/modules/4-1/aaa_radius)
* [OpenSIPS acc module](/modules/4-1/acc)
* [OpenSIPS auth_aaa module](/modules/4-1/auth_aaa)
* [RFC 2865 - RADIUS](https://www.rfc-editor.org/rfc/rfc2865)
* [RFC 2866 - RADIUS Accounting](https://www.rfc-editor.org/rfc/rfc2866)
