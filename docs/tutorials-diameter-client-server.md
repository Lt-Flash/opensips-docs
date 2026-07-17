---
title: "Diameter Client/Server"
subtitle: "How to build and process Diameter requests using OpenSIPS scripting"
author: "by Liviu Chircu"
description: "How to use the aaa_diameter module to send custom Diameter requests and process incoming Diameter requests from OpenSIPS script."
---

## Setting up the freeDiameter libraries

The current **aaa_diameter** module dynamically links against the
freeDiameter libraries and requires **libfdcore** and **libfdproto** version
**1.2.1 or newer**.  On Debian/Ubuntu systems, install the development package
before building the OpenSIPS module from source:

```bash
sudo apt -y install libfreediameter-dev
```

## Installing the "aaa_diameter" OpenSIPS module

If you are building from source:

```bash
include_modules=aaa_diameter make install
```

Otherwise, if you are installing from Debian packages, install the
**opensips-diameter-module** package:

```bash
sudo apt -y install opensips-diameter-module
```

## Defining a Generic Diameter Request + Answer

The *freeDiameter* library only builds messages for Diameter Applications,
Commands and AVPs that exist in its dictionary.  This means that before
OpenSIPS can build and send a custom Diameter request, the request and answer
must be described in a dictionary file.

Attach the `;extra-avps-file:<path-to-dictionary.opensips>` option to the
[aaa_url](/modules/devel/aaa_diameter#param_aaa_url) modparam of
**aaa_diameter**.  The **dictionary.opensips** syntax supports custom Diameter
Applications, Commands and grouped AVPs.  See
[dm_send_request()](/modules/devel/aaa_diameter#func_dm_send_request) for the
full module reference.

For this tutorial, put the following definitions in
`/etc/opensips/dictionary.opensips`:

```text
ATTRIBUTE out_gw            232 string
ATTRIBUTE trunk_id          233 string

ATTRIBUTE rated_duration    234 integer
ATTRIBUTE call_cost         235 integer

ATTRIBUTE Exponent          429 integer32
ATTRIBUTE Value-Digits      447 integer64

ATTRIBUTE Cost-Unit 424 grouped
{
	Value-Digits | REQUIRED | 1
	Exponent | OPTIONAL | 1
}

ATTRIBUTE Currency-Code     425 unsigned32

ATTRIBUTE Unit-Value  445 grouped
{
	Value-Digits | REQUIRED | 1
	Exponent | OPTIONAL | 1
}

ATTRIBUTE Cost-Information  423 grouped
{
	Unit-Value | REQUIRED | 1
	Currency-Code | REQUIRED | 1
	Cost-Unit | OPTIONAL | 1
}

APPLICATION 42 My Diameter Application

REQUEST 92001 My-Custom-Request
{
	Origin-Host | REQUIRED | 1
	Origin-Realm | REQUIRED | 1
	Destination-Realm | REQUIRED | 1
	Sip-From-Tag | REQUIRED | 1
	Sip-To-Tag | REQUIRED | 1
	Sip-Call-Duration | REQUIRED | 1
	Sip-Call-Setuptime | REQUIRED | 1
	Sip-Call-Created | REQUIRED | 1
	Sip-Call-MSDuration | REQUIRED | 1
	out_gw | REQUIRED | 1
	call_cost | REQUIRED | 1
	Cost-Information | OPTIONAL | 1
}

ANSWER 92001 My-Custom-Answer
{
	Origin-Host | REQUIRED | 1
	Origin-Realm | REQUIRED | 1
	Destination-Realm | REQUIRED | 1
	Result-Code | REQUIRED | 1
}
```

## opensips.cfg: Sending Generic Diameter Requests

OpenSIPS **3.2** and newer can build generic Diameter requests with full
control over the request AVPs and full access to the answer AVPs.

Load **aaa_diameter**, point it to the freeDiameter client configuration, and
load the custom dictionary through `extra-avps-file`:

```opensips
log_stdout = yes
...
loadmodule "aaa_diameter.so"
modparam("aaa_diameter", "fd_log_level", 0)
modparam("aaa_diameter", "realm", "diameter.test")
modparam("aaa_diameter", "peer_identity", "server")
modparam("aaa_diameter", "aaa_url",
    "diameter:/etc/freeDiameter/freeDiameter-client.conf;extra-avps-file:/etc/opensips/dictionary.opensips")

loadmodule "json.so"
...
```

Build the request AVP list as a JSON array:

```opensips
# Building and sending a My-Custom-Request (92001) for the
# My Diameter Application (42)
$var(payload) = "[
	{ \"Origin-Host\": \"client.diameter.test\" },
	{ \"Origin-Realm\": \"diameter.test\" },
	{ \"Destination-Realm\": \"diameter.test\" },
	{ \"Sip-From-Tag\": \"dc93-4fba-91db\" },
	{ \"Sip-To-Tag\": \"ae12-47d6-816a\" },
	{ \"Session-Id\": \"a59c-dff0d9efd167\" },
	{ \"Sip-Call-Duration\": 6 },
	{ \"Sip-Call-Setuptime\": 1 },
	{ \"Sip-Call-Created\": 1652372541 },
	{ \"Sip-Call-MSDuration\": 5850 },
	{ \"out_gw\": \"GW-774\" },
	{ \"call_cost\": 10 },
	{ \"Cost-Information\": [
		{\"Unit-Value\": [{\"Value-Digits\": 1000}]},
		{\"Currency-Code\": 35}
		]}
]";
```

Then send the request in blocking mode:

```opensips
$var(rc) = dm_send_request(42, 92001, $var(payload), $var(rpl_avps));
xlog("rc: $var(rc), Reply AVPs: $var(rpl_avps)\n");
```

In this example, the Diameter Application ID is **42**, and the Command Code is
**92001**.  The return code is the `Result-Code` AVP value when an answer was
received.  The response AVPs are available under `$var(rpl_avps)`.  Use the
OpenSIPS [JSON module](/modules/devel/json) to parse and access them:

```opensips
$json(rpl) := $var(rpl_avps);
xlog("Reply AVPs: $json(rpl)\n");
$var(i) = 0;
while ($json(rpl[$var(i)])) {
  xlog("   AVP: $json(rpl[$var(i)])\n");
  $var(i) = $var(i) + 1;
}
```

For non-blocking requests, use the asynchronous
[dm_send_request()](/modules/devel/aaa_diameter#afunc_dm_send_request) form.

## opensips.cfg: Processing Generic Diameter Requests

Starting with OpenSIPS **3.5**, **aaa_diameter** can also act as a Diameter
server and process incoming Diameter requests.  Load
[event_route](/modules/devel/event_route) so OpenSIPS can consume the
**E_DM_REQUEST** events emitted for incoming Diameter requests:

```opensips
log_stdout = yes
...
loadmodule "aaa_diameter.so"
modparam("aaa_diameter", "fd_log_level", 0)
modparam("aaa_diameter", "realm", "diameter.test")
modparam("aaa_diameter", "peer_identity", "server")
modparam("aaa_diameter", "aaa_url",
    "diameter:/etc/freeDiameter/freeDiameter-client.conf;extra-avps-file:/etc/opensips/dictionary.opensips")

loadmodule "event_route.so"
...
event_route [E_DM_REQUEST]
{
    xlog("Req: $param(sess_id) / $param(app_id) / $param(cmd_code)\n");
    xlog("AVPs: $param(avps_json)\n");
}
```

The event provides the following parameters:

* **app_id** (integer) - the Diameter Application Identifier
* **cmd_code** (integer) - the Diameter Command Code
* **sess_id** (string) - the value of either the **Session-Id** AVP,
  **Transaction-Id** AVP or a **NULL** value if neither AVP is present
* **avps_json** (string) - a JSON array containing the AVPs of the request

After processing the request, pack the answer AVPs as a JSON array and call
[dm_send_answer()](/modules/devel/aaa_diameter#func_dm_send_answer):

```opensips
event_route [E_DM_REQUEST]
{
  xlog("Req: $param(sess_id) / $param(app_id) / $param(cmd_code)\n");
  xlog("AVPs: $param(avps_json)\n");

  $var(ans_avps) = "[
          { \"Vendor-Specific-Application-Id\": [{
                  \"Vendor-Id\": 0
                  }] },

          { \"Result-Code\": 2001 },
          { \"Auth-Session-State\": 0 },
          { \"Origin-Host\": \"opensips.diameter.test\" },
          { \"Origin-Realm\": \"diameter.test\" }
  ]";

  if (!dm_send_answer($var(ans_avps)))
    xlog("ERROR - failed to send Diameter answer\n");
}
```
