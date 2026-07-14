---
title: "OpenSIPS Asterisk Integration"
description: "Build an OpenSIPS 4.0 and Asterisk integration using OpenSIPS Debian packages, an Asterisk 22 LTS install, a default OpenSIPS-based routing script, and minimal Asterisk PJSIP changes."
---

## Realtime **OpenSIPS - Asterisk** Integration

This tutorial shows how to set up OpenSIPS 4.0 and Asterisk together using
OpenSIPS Debian packages, an Asterisk 22 LTS install, a database-backed
OpenSIPS registrar and a minimal Asterisk PJSIP configuration.

OpenSIPS is the SIP edge server: users register with OpenSIPS, OpenSIPS
authenticates them and routes user-to-user calls. Asterisk is used as a media
application server. Calls to Asterisk services are sent from OpenSIPS by
prefixing the dialed extension with `*`:

* `*9196` is sent to Asterisk as `9196`
* `**1234` would be sent to Asterisk as `*1234`

The Asterisk side is intentionally small: keep the default configuration, keep
Asterisk SIP away from OpenSIPS when both run on the same host, add one PJSIP
endpoint that trusts traffic from OpenSIPS and add one dialplan extension used
by the working test call.

### Topology Used Below

Replace these values with your own:

```text
SIP domain:       example.com
OpenSIPS IP:      192.0.2.10
Asterisk IP:      192.0.2.30
OpenSIPS SIP:     udp:192.0.2.10:5060
Asterisk SIP:     udp:192.0.2.30:5070
Asterisk app:     9196 echo service
Database:         MariaDB/MySQL on the OpenSIPS host
```

If both services run on one host, keep the same idea and use different SIP
ports, for example OpenSIPS on `5060` and Asterisk on `5070`.

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

### Install Asterisk 22 LTS

Debian Trixie does not ship an `asterisk` binary package in its stable package
set. If your Debian suite or vendor repository provides Asterisk packages, use
the package manager:

```bash
apt-get update
apt-get install -y asterisk
```

On Debian Trixie, install Asterisk 22 LTS from upstream source. The upstream
downloads page lists Asterisk 22 as an LTS branch, and the Asterisk source
installation guide has the complete build details. A compact install sequence
for a new lab host is:

```bash
apt-get update
apt-get install -y build-essential ca-certificates curl tar

cd /usr/src
curl -LO https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-22-current.tar.gz
tar xzf asterisk-22-current.tar.gz
cd asterisk-22*/

./contrib/scripts/install_prereq install
./configure
make menuselect
make
make install
make samples
make config
ldconfig
systemctl enable --now asterisk
```

When using `make menuselect`, keep the PJSIP modules and `app_echo` enabled.
If you also want the optional realtime mapping chapter below, keep `res_odbc`
and `app_voicemail` enabled too. Run `make samples` only on a new installation
where overwriting the sample Asterisk configuration is acceptable.

Check that Asterisk starts and that PJSIP is available:

```bash
systemctl status asterisk
asterisk -rx 'module show like res_pjsip'
asterisk -rx 'core show version'
```

---

## OpenSIPS Configuration

The script below is based on the default OpenSIPS 4.0 residential script, with
the following additions:

* MySQL-backed authentication
* persistent, MySQL-backed user location
* SIP Digest authentication for `REGISTER` and subscriber-originated calls
* one-to-one mapping between SIP usernames and authentication usernames
* one `route[asterisk]` block for star-prefixed media services

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

	route(asterisk);

	if (!lookup("location", "method-filtering")) {
		t_reply(404, "Not Found");
		exit;
	}

	do_accounting("log", "missed");
	route(relay);
}

route[asterisk] {
	if (!is_method("INVITE"))
		return;

	if ($rU =~ "^\*") {
		strip(1);
		$du = "sip:192.0.2.30:5070";   # CUSTOMIZE: Asterisk PJSIP listener
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

---

## Asterisk Configuration

Do not replace the Asterisk configuration tree. Add one PJSIP transport and
endpoint for OpenSIPS, then add one dialplan context for calls that arrive from
OpenSIPS.

### PJSIP Listener and OpenSIPS Endpoint

Add this to `/etc/asterisk/pjsip.conf` and update the `CUSTOMIZE` values:

```ini
[transport-udp-opensips]
type=transport
protocol=udp
bind=192.0.2.30:5070        ; CUSTOMIZE: Asterisk listener

[opensips-endpoint]
type=endpoint
transport=transport-udp-opensips
context=from-opensips
disallow=all
allow=ulaw,alaw
aors=opensips-aor
direct_media=no
trust_id_inbound=yes

[opensips-aor]
type=aor
contact=sip:192.0.2.10:5060 ; CUSTOMIZE: OpenSIPS listener

[opensips-identify]
type=identify
endpoint=opensips-endpoint
match=192.0.2.10/32         ; CUSTOMIZE: OpenSIPS source IP
```

If Asterisk and OpenSIPS run on the same host, make sure no other PJSIP
transport is binding the same IP and port as OpenSIPS.

Reload PJSIP:

```bash
asterisk -rx 'pjsip reload'
asterisk -rx 'pjsip show endpoint opensips-endpoint'
```

### Asterisk Service Extension

Add this to `/etc/asterisk/extensions.conf`:

```ini
[from-opensips]
exten => 9196,1,Answer()
 same => n,Echo()
 same => n,Hangup()
```

This gives the OpenSIPS-to-Asterisk handoff a complete target: dialing `*9196`
from an OpenSIPS subscriber reaches Asterisk as `9196` and starts an echo test
in the `from-opensips` context.

Reload the dialplan:

```bash
asterisk -rx 'dialplan reload'
asterisk -rx 'dialplan show from-opensips'
```

Use a firewall or host ACLs so only OpenSIPS can reach the Asterisk PJSIP port.
The PJSIP `identify` match is useful, but it is not a replacement for network
access control.

---

## Testing

Register `alice@example.com` and `bob@example.com` to OpenSIPS on port `5060`.

Call flow checks:

* `alice` calls `bob`: OpenSIPS authenticates the caller, looks up `bob` in
  `location` and relays the call to Bob's registered contact.
* `alice` calls `*9196`: OpenSIPS strips the first `*`, sends `9196` to
  Asterisk at `192.0.2.30:5070`, and Asterisk answers the echo service from
  the `from-opensips` dialplan context.
* `alice` calls `**1234`: OpenSIPS strips only one `*`, so Asterisk receives
  `*1234`. Add that extension in `from-opensips` if you need Asterisk star
  feature codes.

Useful checks:

```bash
opensips-cli -x mi ul:dump
opensips-cli -x mi ps
asterisk -rx 'pjsip show transports'
asterisk -rx 'pjsip show endpoint opensips-endpoint'
asterisk -rx 'core show channels'
```

OpenSIPS 4.0 uses the canonical `module:command` MI command names, such as
`ul:dump`.

---

## Optional Asterisk Realtime Views from OpenSIPS Subscribers

The handoff above does not require Asterisk user records. Add this section when
Asterisk applications need user data that follows the OpenSIPS subscriber list,
such as voicemail users, mailbox aliases or Asterisk-side PJSIP user objects.

The realtime version of this tutorial included this user-mapping information.
It is kept here as an optional chapter so the basic SIP handoff stays small,
while the realtime integration remains available when Asterisk services need it.

SIP endpoints still register and authenticate to OpenSIPS. These views let
Asterisk read user data from the OpenSIPS database when Asterisk realtime asks
for it.

### OpenSIPS Subscriber Fields for Asterisk

OpenSIPS 4.0 creates the `subscriber` table with `username`, `domain`,
`password` and `ha1` fields. Add Asterisk-specific fields for voicemail and
display data:

```sql
ALTER TABLE subscriber
  ADD COLUMN ast_vm_password VARCHAR(32) NOT NULL DEFAULT '1234',
  ADD COLUMN ast_first_name VARCHAR(64) NOT NULL DEFAULT '',
  ADD COLUMN ast_last_name VARCHAR(64) NOT NULL DEFAULT '',
  ADD COLUMN ast_email VARCHAR(128) NOT NULL DEFAULT '';
```

Set the values for the tutorial users:

```sql
UPDATE subscriber
  SET ast_vm_password = '1001',
      ast_first_name = 'Alice',
      ast_last_name = 'Example',
      ast_email = 'alice@example.com'
  WHERE username = 'alice' AND domain = 'example.com';

UPDATE subscriber
  SET ast_vm_password = '1002',
      ast_first_name = 'Bob',
      ast_last_name = 'Example',
      ast_email = 'bob@example.com'
  WHERE username = 'bob' AND domain = 'example.com';
```

OpenSIPS ignores these extra columns. Asterisk reads them only through the
views below.

### Asterisk Database and Views

Create a separate `asterisk` database for Asterisk realtime data:

```sql
CREATE DATABASE IF NOT EXISTS asterisk;

CREATE USER IF NOT EXISTS 'asterisk'@'localhost'
  IDENTIFIED BY 'asterisk_pwd';

GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER, INDEX
  ON asterisk.* TO 'asterisk'@'localhost';

GRANT SELECT ON opensips.* TO 'asterisk'@'localhost';

FLUSH PRIVILEGES;
```

Create the realtime voicemail view over the OpenSIPS `subscriber` table:

```sql
CREATE OR REPLACE VIEW asterisk.vmusers AS
SELECT
  CONCAT(username, '@', domain) AS uniqueid,
  username AS customer_id,
  'from-opensips' AS context,
  username AS mailbox,
  ast_vm_password AS password,
  TRIM(CONCAT(ast_first_name, ' ', ast_last_name)) AS fullname,
  ast_email AS email,
  NULL AS pager,
  NOW() AS stamp
FROM opensips.subscriber;
```

Create voicemail aliases from the OpenSIPS `dbaliases` table:

```sql
CREATE OR REPLACE VIEW asterisk.vmaliases AS
SELECT
  alias_username AS alias,
  'from-opensips' AS context,
  username AS mailbox
FROM opensips.dbaliases;
```

Create an optional table for ODBC voicemail message storage:

```sql
CREATE TABLE IF NOT EXISTS asterisk.voicemessages (
  id INT NOT NULL AUTO_INCREMENT,
  msgnum INT NOT NULL DEFAULT 0,
  dir VARCHAR(255) DEFAULT '',
  context VARCHAR(80) DEFAULT '',
  macrocontext VARCHAR(80) DEFAULT '',
  callerid VARCHAR(80) DEFAULT '',
  origtime VARCHAR(40) DEFAULT '',
  duration VARCHAR(20) DEFAULT '',
  mailboxuser VARCHAR(80) DEFAULT '',
  mailboxcontext VARCHAR(80) DEFAULT '',
  recording LONGBLOB,
  PRIMARY KEY (id),
  KEY dir (dir)
);
```

Create optional PJSIP realtime views for Asterisk user objects. The basic
OpenSIPS-to-Asterisk handoff still uses the static `opensips-endpoint`; these
views are useful only if an Asterisk application or dialplan also wants to look
up users such as `alice` and `bob` as PJSIP objects.

```sql
CREATE OR REPLACE VIEW asterisk.ps_endpoints AS
SELECT
  username AS id,
  'transport-udp-opensips' AS transport,
  CONCAT(username, '-aor') AS aors,
  CONCAT(username, '-auth') AS auth,
  'from-opensips' AS context,
  'all' AS disallow,
  'ulaw,alaw' AS allow,
  'no' AS direct_media,
  'yes' AS trust_id_inbound,
  CONCAT(TRIM(CONCAT(ast_first_name, ' ', ast_last_name)),
         ' <', username, '>') AS callerid,
  'sip:192.0.2.10:5060;lr' AS outbound_proxy
FROM opensips.subscriber;

CREATE OR REPLACE VIEW asterisk.ps_aors AS
SELECT
  CONCAT(username, '-aor') AS id,
  CONCAT('sip:', username, '@', domain) AS contact,
  1 AS max_contacts,
  'yes' AS remove_existing,
  60 AS qualify_frequency
FROM opensips.subscriber;

CREATE OR REPLACE VIEW asterisk.ps_auths AS
SELECT
  CONCAT(username, '-auth') AS id,
  'userpass' AS auth_type,
  username AS username,
  password AS password,
  ha1 AS password_digest,
  domain AS realm
FROM opensips.subscriber;
```

### ODBC Access for Asterisk

Install UnixODBC and a MariaDB/MySQL ODBC driver for your Debian release. On
Debian systems this is typically:

```bash
apt-get install -y unixodbc odbc-mariadb
```

Check the installed driver name:

```bash
odbcinst -q -d
```

Create `/etc/odbc.ini` and use the driver name reported on your system:

```ini
[asterisk]
Description = Asterisk database
Driver      = MariaDB Unicode
SERVER      = 127.0.0.1
PORT        = 3306
DATABASE    = asterisk
USER        = asterisk
PASSWORD    = asterisk_pwd
```

Check the DSN:

```bash
isql -v asterisk asterisk asterisk_pwd
```

Configure `/etc/asterisk/res_odbc.conf`:

```ini
[asterisk]
enabled => yes
dsn => asterisk
username => asterisk
password => asterisk_pwd
pre-connect => yes
```

Reload ODBC and check the connection:

```bash
asterisk -rx 'module reload res_odbc.so'
asterisk -rx 'odbc show'
```

### Asterisk Realtime Mapping

Add the realtime families to `/etc/asterisk/extconfig.conf`:

```ini
[settings]
voicemail => odbc,asterisk,vmusers
voicemail_data => odbc,asterisk,voicemessages
ps_endpoints => odbc,asterisk,ps_endpoints
ps_aors => odbc,asterisk,ps_aors
ps_auths => odbc,asterisk,ps_auths
```

Reload the affected modules:

```bash
asterisk -rx 'module reload res_config_odbc.so'
asterisk -rx 'module reload app_voicemail.so'
asterisk -rx 'module reload res_pjsip.so'
```

Check the mapped data:

```bash
asterisk -rx 'voicemail show users'
asterisk -rx 'pjsip show endpoint alice'
asterisk -rx 'pjsip show aor alice-aor'
```

If these commands show values from the OpenSIPS database, Asterisk can map
`alice@example.com` and `bob@example.com` to Asterisk realtime users whenever a
configured Asterisk application asks for that data.

---

## Resulting Functions

With the base setup:

* OpenSIPS authenticates SIP subscribers and stores their contacts.
* OpenSIPS routes subscriber-to-subscriber calls from its `location` table.
* OpenSIPS sends star-prefixed media service calls to Asterisk.
* Asterisk accepts trusted calls from OpenSIPS and answers the `9196` echo
  service.

With the optional realtime mapping:

* Asterisk resolves OpenSIPS subscribers as realtime voicemail users.
* Asterisk can expose matching PJSIP realtime user objects for services that
  consume Asterisk-side users.

---

## Data Used by This Setup

OpenSIPS stores subscriber credentials in the `subscriber` table and registered
contacts in the `location` table. The two working subscribers are created with:

```bash
opensips-cli -x user add alice@example.com alicepass
opensips-cli -x user add bob@example.com bobpass
```

Asterisk does not authenticate the client SIP endpoints in this setup. The
complete Asterisk data needed for the handoff is:

* `/etc/asterisk/pjsip.conf`, which defines the OpenSIPS-facing transport,
  endpoint, AOR and IP identify match
* `/etc/asterisk/extensions.conf`, which defines the `9196` echo service used
  by the test call

The optional realtime mapping adds:

* Asterisk-specific columns in the OpenSIPS `subscriber` table
* the `asterisk` database with realtime views over OpenSIPS subscriber data
* an ODBC DSN named `asterisk`
* `res_odbc.conf` and `extconfig.conf` realtime mappings

---

## References

* OpenSIPS APT repository: <https://apt.opensips.org/packages.php>
* Debian Asterisk package page: <https://packages.debian.org/sid/asterisk>
* Debian Asterisk package tracker: <https://tracker.debian.org/pkg/asterisk>
* Asterisk installing overview: <https://docs.asterisk.org/Getting-Started/Installing-Asterisk/>
* Asterisk source installation: <https://docs.asterisk.org/Getting-Started/Installing-Asterisk/Installing-Asterisk-From-Source/>
* Asterisk supported versions and downloads: <https://www.asterisk.org/downloads/asterisk/all-asterisk-versions/>
