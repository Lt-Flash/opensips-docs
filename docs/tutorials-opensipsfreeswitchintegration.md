---
title: "OpenSIPS FreeSWITCH Integration"
description: "Build a current OpenSIPS 4.0 and FreeSWITCH integration using Debian packages, a default OpenSIPS-based routing script, and minimal FreeSWITCH changes."
---

## Realtime **OpenSIPS - FreeSWITCH** Integration

This tutorial shows how to set up OpenSIPS 4.0 and FreeSWITCH together using
Debian packages, a database-backed OpenSIPS registrar and a minimal FreeSWITCH
configuration.

OpenSIPS is the SIP edge server: users register with OpenSIPS, OpenSIPS
authenticates them and routes user-to-user calls. FreeSWITCH is used as a media
application server. Calls to FreeSWITCH services are sent from OpenSIPS by
prefixing the dialed extension with `*`:

* `*9196` is sent to FreeSWITCH as `9196`
* `**1234` is sent to FreeSWITCH as `*1234`

The FreeSWITCH side is intentionally small: keep the default configuration,
avoid replacing whole XML files, move FreeSWITCH SIP away from OpenSIPS when
both run on the same host, add one ACL entry, add one dialplan entry that
accepts calls from OpenSIPS and add one service extension used by the working
test call.

### Topology Used Below

Replace these values with your own:

```text
SIP domain:       example.com
OpenSIPS IP:      192.0.2.10
FreeSWITCH IP:    192.0.2.20
OpenSIPS SIP:     udp:192.0.2.10:5060
FreeSWITCH SIP:   udp:192.0.2.20:5090
FreeSWITCH app:   9196 echo service
Database:         MariaDB/MySQL on the OpenSIPS host
```

If both services run on one host, keep the same idea and use different SIP
ports, for example OpenSIPS on `5060` and FreeSWITCH on `5090`.

---

## Installation

### Base System

Start from a supported Debian installation and install basic tools:

```bash
apt-get update
apt-get install -y ca-certificates curl gnupg lsb-release mariadb-server
```

The examples below use Debian's `VERSION_CODENAME`, so they work for supported
Debian releases listed by the OpenSIPS APT selector.

### Install OpenSIPS 4.0 from Debian Packages

OpenSIPS packages are published at <https://apt.opensips.org/packages.php>.
Select your OS and the `OpenSIPS 4.0` stable package there if you want the
generated repository commands.

```bash
. /etc/os-release

curl https://apt.opensips.org/opensips-org.gpg \
    -o /usr/share/keyrings/opensips-org.gpg

echo "deb [signed-by=/usr/share/keyrings/opensips-org.gpg] https://apt.opensips.org ${VERSION_CODENAME} 4.0-releases" \
    >/etc/apt/sources.list.d/opensips.list

echo "deb [signed-by=/usr/share/keyrings/opensips-org.gpg] https://apt.opensips.org ${VERSION_CODENAME} cli-nightly" \
    >/etc/apt/sources.list.d/opensips-cli.list

apt-get update
apt-get install -y opensips opensips-cli opensips-auth-modules \
    opensips-mysql-module opensips-mysql-dbschema
```

### Create the OpenSIPS Database

Create `/etc/opensips-cli.cfg`:

```ini
[default]
database_admin_url: mysql://root@localhost
database_url: mysql://opensips:opensipsrw@localhost/opensips
database_name: opensips
```

Then create the database:

```bash
opensips-cli -x database create
```

The command creates the `opensips` database, the `opensips` database user and
the standard tables. If your MariaDB/MySQL setup does not allow the
`root@localhost` URL above and you are working in a contained lab environment,
you can temporarily remove the local `root` database password while creating the
OpenSIPS database. On public, shared or production systems, keep the database
root account protected and adjust `database_admin_url` to match a proper local
database administrator account:

```ini
database_admin_url: mysql://<ADMIN_USER>:<ADMIN_PASSWORD>@localhost
```

Add test subscribers:

```bash
opensips-cli -x user add alice@example.com alicepass
opensips-cli -x user add bob@example.com bobpass
```

### Install FreeSWITCH from Debian Packages

FreeSWITCH package installation is documented upstream in the FreeSWITCH source
tree under `scripts/packaging`. The current package path uses `fsget` and
requires a SignalWire Personal Access Token or Enterprise API token:

```bash
apt-get update
apt-get install -y curl

curl -sSL https://freeswitch.org/fsget | bash -s <SIGNALWIRE_TOKEN> release install
```

If you cannot use the authenticated package repository, use the FreeSWITCH
release source from <https://github.com/signalwire/freeswitch/releases> or build
Debian packages with the documented `fsdeb` flow:
<https://github.com/signalwire/freeswitch/tree/master/scripts/packaging/build>.

---

## OpenSIPS Configuration

The script below is based on the default OpenSIPS 4.0 residential script, with
the following additions:

* MySQL-backed authentication
* persistent, MySQL-backed user location
* SIP Digest authentication for `REGISTER` and subscriber-originated calls
* one-to-one mapping between SIP usernames and authentication usernames
* one `route[freeswitch]` block for star-prefixed media services

Save it as `/etc/opensips/opensips.cfg` and update the `CUSTOMIZE` values. Keep
the packaged `mpath` value if your Debian architecture is not `amd64`.

```opensips
####### Global Parameters #########

log_level=3
xlog_level=3
stderror_enabled=no
syslog_enabled=yes
syslog_facility=LOG_LOCAL0

udp_workers=4

socket=udp:192.0.2.10:5060   # CUSTOMIZE: OpenSIPS listener
alias=example.com            # CUSTOMIZE: SIP domain

####### Modules Section ########

mpath="/usr/lib/x86_64-linux-gnu/opensips/modules/"  # Debian amd64 path

loadmodule "signaling.so"
loadmodule "sl.so"

loadmodule "tm.so"
modparam("tm", "fr_timeout", 5)
modparam("tm", "fr_inv_timeout", 30)
modparam("tm", "restart_fr_on_each_reply", 0)
modparam("tm", "onreply_avp_mode", 1)

loadmodule "rr.so"
modparam("rr", "append_fromtag", 0)

loadmodule "maxfwd.so"
loadmodule "sipmsgops.so"

loadmodule "mi_fifo.so"
modparam("mi_fifo", "fifo_name", "/run/opensips/opensips_fifo")
modparam("mi_fifo", "fifo_mode", 0666)

loadmodule "db_mysql.so"

loadmodule "auth.so"
loadmodule "auth_db.so"
modparam("auth_db", "db_url", "mysql://opensips:opensipsrw@localhost/opensips")
modparam("auth_db", "use_domain", true)
modparam("auth_db", "load_credentials", "")

loadmodule "usrloc.so"
modparam("usrloc", "db_url", "mysql://opensips:opensipsrw@localhost/opensips")
modparam("usrloc", "use_domain", true)
modparam("usrloc", "working_mode_preset", "single-instance-sql-write-back")

loadmodule "registrar.so"
modparam("registrar", "tcp_persistent_flag", "TCP_PERSISTENT")

loadmodule "acc.so"
modparam("acc", "early_media", 0)
modparam("acc", "report_cancels", 0)
modparam("acc", "detect_direction", 0)

loadmodule "proto_udp.so"

####### Routing Logic ########

route {
	if (!mf_process_maxfwd_header(10)) {
		send_reply(483, "Too Many Hops");
		exit;
	}

	if (has_totag()) {
		if (is_method("ACK") && t_check_trans()) {
			t_relay();
			exit;
		}

		if (!loose_route()) {
			send_reply(404, "Not here");
			exit;
		}

		if (is_method("BYE"))
			do_accounting("log", "failed");

		route(relay);
		exit;
	}

	if (is_method("CANCEL")) {
		if (t_check_trans())
			t_relay();
		exit;
	}

	t_check_trans();

	if (!is_method("REGISTER")) {
		if (is_myself("$fd")) {
			if (!proxy_authorize("", "subscriber")) {
				proxy_challenge("", "auth");
				exit;
			}

			if ($au != $fU) {
				send_reply(403, "Forbidden auth ID");
				exit;
			}

			consume_credentials();
		} else if (!is_myself("$rd")) {
			send_reply(403, "Relay Forbidden");
			exit;
		}
	}

	if (loose_route()) {
		xlog("L_ERR", "Attempt to route with preloaded Route [$fu/$tu/$ru/$ci]\n");
		if (!is_method("ACK"))
			send_reply(403, "Preload Route denied");
		exit;
	}

	if (!is_method("REGISTER|MESSAGE"))
		record_route();

	if (is_method("INVITE"))
		do_accounting("log");

	if (!is_myself("$rd")) {
		append_hf("P-hint: outbound\r\n");
		route(relay);
	}

	if (is_method("PUBLISH|SUBSCRIBE")) {
		send_reply(503, "Service Unavailable");
		exit;
	}

	if (is_method("REGISTER")) {
		if (!www_authorize("", "subscriber")) {
			www_challenge("", "auth");
			exit;
		}

		if ($au != $tU) {
			send_reply(403, "Forbidden auth ID");
			exit;
		}

		if (!save("location"))
			xlog("L_ERR", "failed to register AoR $tu\n");

		exit;
	}

	if ($rU == NULL) {
		send_reply(484, "Address Incomplete");
		exit;
	}

	route(freeswitch);

	if (!lookup("location", "method-filtering")) {
		t_reply(404, "Not Found");
		exit;
	}

	do_accounting("log", "missed");
	route(relay);
}

route[freeswitch] {
	if (!is_method("INVITE"))
		return;

	if ($rU =~ "^\*") {
		strip(1);
		$du = "sip:192.0.2.20:5090";   # CUSTOMIZE: FreeSWITCH SIP listener
		route(relay);
	}
}

route[relay] {
	if (is_method("INVITE")) {
		t_on_branch("per_branch_ops");
		t_on_reply("handle_nat");
		t_on_failure("missed_call");
	}

	if (!t_relay())
		send_reply(500, "Internal Error");

	exit;
}

branch_route[per_branch_ops] {
	xlog("new branch at $ru\n");
}

onreply_route[handle_nat] {
	xlog("incoming reply\n");
}

failure_route[missed_call] {
	if (t_was_cancelled())
		exit;
}
```

Validate and restart OpenSIPS:

```bash
opensips -C -f /etc/opensips/opensips.cfg
systemctl restart opensips
systemctl status opensips
```

## FreeSWITCH Configuration

Do not replace the default FreeSWITCH configuration tree. Only change the SIP
port if it conflicts with OpenSIPS, add one ACL entry, and add one dialplan
file.

### SIP IP and Port

If FreeSWITCH runs on the same IP as OpenSIPS, edit `/etc/freeswitch/vars.xml`
and move the internal SIP profile away from `5060`:

```xml
<X-PRE-PROCESS cmd="set" data="internal_sip_port=5090"/>
```

If you use the external profile too, keep it on another port:

```xml
<X-PRE-PROCESS cmd="set" data="external_sip_port=5091"/>
```

If FreeSWITCH should bind Sofia to a specific listener IP, edit
`/etc/freeswitch/sip_profiles/internal.xml` and set both `sip-ip` and
`ext-sip-ip` to the OpenSIPS-facing address:

```xml
<param name="sip-ip" value="192.0.2.20"/>
<param name="ext-sip-ip" value="192.0.2.20"/>
```

Do not leave the internal profile advertising a public or STUN-discovered
`ext-sip-ip` unless OpenSIPS reaches FreeSWITCH through that public address.

Restart FreeSWITCH after changing the profile IP or ports:

```bash
systemctl restart freeswitch
```

### Allow OpenSIPS in the FreeSWITCH ACL

The default internal SIP profile uses the `domains` ACL before accepting
unauthenticated calls into the dialplan. Add the OpenSIPS source IP to that ACL
in `/etc/freeswitch/autoload_configs/acl.conf.xml`:

```xml
<list name="domains" default="deny">
  <node type="allow" domain="$${domain}"/>
  <node type="allow" cidr="192.0.2.10/32"/>
</list>
```

Reload ACLs:

```bash
fs_cli -x 'reloadacl'
```

### Dialplan Entry for OpenSIPS

Create `/etc/freeswitch/dialplan/public/00_from_opensips.xml`:

```xml
<include>
  <extension name="from_opensips" continue="false">
    <condition field="network_addr" expression="^192\.0\.2\.10$">
      <action application="transfer" data="${destination_number} XML default"/>
    </condition>
  </extension>
</include>
```

Replace `192.0.2.10` with the OpenSIPS source IP as seen by FreeSWITCH. This
keeps the public context closed to arbitrary callers while allowing OpenSIPS to
hand trusted, already-authenticated users into the default FreeSWITCH context.

### FreeSWITCH Service Extension

Create `/etc/freeswitch/dialplan/default/20_opensips_services.xml`:

```xml
<include>
  <extension name="opensips_echo_test">
    <condition field="destination_number" expression="^9196$">
      <action application="answer"/>
      <action application="echo"/>
    </condition>
  </extension>
</include>
```

This gives the OpenSIPS-to-FreeSWITCH handoff a complete target: dialing
`*9196` from an OpenSIPS subscriber reaches FreeSWITCH as `9196` and starts an
echo test in the default context.

Reload XML:

```bash
fs_cli -x 'reloadxml'
```

Use a firewall or host ACLs so only OpenSIPS can reach the FreeSWITCH SIP port.
The dialplan check is useful, but it is not a replacement for network access
control.

---

## Testing

Register `alice@example.com` and `bob@example.com` to OpenSIPS on port `5060`.

Call flow checks:

* `alice` calls `bob`: OpenSIPS authenticates the caller, looks up `bob` in
  `location` and relays the call to Bob's registered contact.
* `alice` calls `*9196`: OpenSIPS strips the first `*`, sends `9196` to
  FreeSWITCH at `192.0.2.20:5090`, and FreeSWITCH answers the echo service from
  the `default` dialplan context.
* `alice` calls `**1234`: OpenSIPS strips only one `*`, so FreeSWITCH receives
  `*1234`.

Useful checks:

```bash
opensips-cli -x mi ul:dump
opensips-cli -x mi ps
fs_cli -x 'sofia status'
fs_cli -x 'show calls'
```

OpenSIPS 4.0 uses the canonical `module:command` MI command names, such as
`ul:dump`.

---

## Data Used by This Setup

OpenSIPS stores subscriber credentials in the `subscriber` table and registered
contacts in the `location` table. The two working subscribers are created with:

```bash
opensips-cli -x user add alice@example.com alicepass
opensips-cli -x user add bob@example.com bobpass
```

FreeSWITCH does not authenticate the client SIP endpoints in this setup. The
complete FreeSWITCH data needed for the handoff is:

* `/etc/freeswitch/vars.xml`, which sets the internal SIP port
* `/etc/freeswitch/sip_profiles/internal.xml`, which sets the Sofia `sip-ip`
  and `ext-sip-ip`
* the `domains` ACL entry allowing `192.0.2.10/32`
* `/etc/freeswitch/dialplan/public/00_from_opensips.xml`, which accepts calls
  from OpenSIPS and transfers them to the `default` context
* `/etc/freeswitch/dialplan/default/20_opensips_services.xml`, which defines
  the `9196` echo service used by the test call

---

## References

* OpenSIPS APT repository: <https://apt.opensips.org/packages.php>
* FreeSWITCH releases: <https://github.com/signalwire/freeswitch/releases>
