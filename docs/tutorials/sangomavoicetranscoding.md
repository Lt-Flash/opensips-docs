---
title: "Sangoma Voice Transcoding"
author: "by Liviu Chircu"
description: "This tutorial illustrates the required steps in order to perform audio transcoding with the D-series cards manufactured by Sangoma, using the sngtc_server da..."
---

![tc setup](/public/uploads/tc-setup.png)

## Tutorial Overview

This tutorial illustrates the required steps in order to perform audio transcoding with the D-series cards manufactured by Sangoma, using the *sngtc_server* daemon and the OpenSIPS *sngtc* module as its client. The latest version of the transcoding library, as of 1 August 2013 is: **sng-tc-linux-1.3.4.1**

  

> [!WARNING]
> As mentioned in the [module documentation](/modules/2-2/sngtc), transcoding may only be done if the UAC performs early SDP negotiation and the UAS supports late SDP negotiation.

---

### Installing the transcoding card

The necessary firmware for the D-series transcoding cards can be set up using the *sngtc_tool*. The cards which require PCI connectivity (D100 and D500) also need additional kernel drivers. Please refer to the [Sangoma wiki](http://wiki.sangoma.com/Sangoma-Media-Transcoding-Product-Selection) for installation tutorials.

---

#### Setting up the *sngtc_server*

Installing the *sngtc_server* is straightforward and also documented in the [Sangoma wiki](http://wiki.sangoma.com/FreeSWITCH-D100-Single-Server-Installation). After doing the configuration, it can be started using the init script:
```bash

   $ /etc/init.d/sngtc_server_ctrl start

```
By default it logs to /var/log/sngtc_server.log

---

#### Call flow with the *sngtc* OpenSIPS module

Due to the limitations of the transcoding library, transcoding sessions can only be created if both codec A and codec B are known. Since this cannot be accomplished with a standard SIP call flow, the module *restricts* the receiving UA to perform *late SDP negotiation*. The diagram below explains this better:

![dialog establish tc](/public/uploads/dialog-establish-tc.png)

---

#### Using the *sngtc* module

The **sngtc module** exports 3 functions. They are called upon receiving INVITE, 200 OK and ACK messages. A short overview on their purpose would be:
* sngtc_offer - called at any INVITE request (re-INVITES too), deletes the INVITE SDP body
* sngtc_callee_answer - called at 200 OK responses, intersects codec offers, creates transcoding sessions if necessary
* sngtc_caller_answer - called at ACK requests, adds an SDP body to the ACK request

All functions properly handle retransmissions. More details can be found in the [module documentation](/modules/devel/uri). A basic configuration file is shown below:

  

```opensips
####### Global Parameters #########

log_level=3
xlog_level=3
stderror_enabled=yes
syslog_enabled=no
syslog_facility=LOG_LOCAL0

udp_workers=4

socket=udp:127.0.0.1:5060

####### Modules ########

mpath = "modules/"

loadmodule "exec.so"
loadmodule "signaling.so"
loadmodule "sl.so"
loadmodule "maxfwd.so"
loadmodule "sipmsgops.so"
loadmodule "db_mysql.so"
loadmodule "registrar.so"
loadmodule "sngtc.so"

loadmodule "tm.so"
modparam("tm", "fr_timeout", 5)
modparam("tm", "fr_inv_timeout", 30)
modparam("tm", "restart_fr_on_each_reply", 0)
modparam("tm", "disable_6xx_block", 1)
modparam("tm", "onreply_avp_mode", 1)

#### Record Route Module
loadmodule "rr.so"
/* do not append from tag to the RR (no need for this script) */
modparam("rr", "append_fromtag", 0)

#### FIFO Management Interface
loadmodule "mi_fifo.so"
modparam("mi_fifo", "fifo_name", "/var/tmp/opensips_fifo")
modparam("mi_fifo", "fifo_mode", 0666)

#### DIALOG module
loadmodule "dialog.so"
modparam("dialog", "db_mode", 0)
modparam("dialog", "default_timeout", 3600)

#### USeR LOCation module
loadmodule "usrloc.so"
modparam("usrloc", "working_mode_preset", "single-instance-no-db")
modparam("usrloc", "db_url",	"mysql://opensips:opensipsrw@localhost/opensips")

loadmodule "proto_udp.so"

####### Routing Logic ########

route {

	if (!mf_process_maxfwd_header(10)) {
		sl_send_reply(483, "Too Many Hops");
		exit;
	}

	force_rport();

	if (has_totag()) {
		# sequential request withing a dialog should
		# take the path determined by record-routing

		if (loose_route()) {

			if (is_method("BYE")) {
				setsflag(FAIL_TRANS_FLAG);
			} else if (is_method("INVITE")) {
				sngtc_offer();
			} else if (is_method("ACK")) {
				sngtc_caller_answer();
			}

			# route it out to whatever destination was set by loose_route()
			# in $du (destination URI).
			route(relay);
		} else {

			if (is_method("ACK")) {
				if (t_check_trans()) {
					# non loose-route, but stateful ACK; must be an ACK after
					# a 487 or e.g. 404 from upstream server
					t_relay();
					exit;
				} else {
					# ACK without matching transaction ->
					# ignore and discard
					exit;
				}
			}
			sl_send_reply(404, "Not here");
		}
		exit;
	}

	# CANCEL processing
	if (is_method("CANCEL")) {
		if (t_check_trans())
			t_relay();
		exit;
	}

	# absorb retransmissions, but do not create transaction
	t_check_trans();

	# preloaded route checking
	if (loose_route()) {
		xlog("L_ERR",
			"Attempt to route with preloaded Route's [$fu/$tu/$ru/$ci]");
		if (!is_method("ACK"))
			send_reply(403, "Preload Route denied");
		exit;
	}

	# record routing
	if (!is_method("REGISTER|MESSAGE"))
		record_route();

	# requests for my domain
	if (is_method("PUBLISH|SUBSCRIBE")) {
		send_reply(503, "Service Unavailable");
		exit;
	}

	if (is_method("REGISTER")) {
		if (!save("location"))
			sl_reply_error();

		exit;
	}

	if (!$rU) {
		# request with no Username in RURI
		send_reply(484, "Address Incomplete");
		exit;
	}

	# do lookup with method filtering
	if (!lookup("location","method-filtering")) {
		t_reply(404, "Not Found");
		exit;
	}

	if (is_method("INVITE"))
		sngtc_offer();

	route(relay);
}

route[relay] {
	# for INVITEs enable some additional helper routes
	if (is_method("INVITE")) {
		t_on_reply("handle_answer");
		t_on_failure("missed_call");
	}

	if (!t_relay()) {
		send_reply(500, "Internal Error");
	}

	exit;
}

onreply_route[handle_answer] {
	if ($rs == 200)
		sngtc_callee_answer("11.12.13.14", "11.12.13.14");
}

failure_route[missed_call] {
	if (t_was_cancelled()) {
		exit;
	}

	if (next_branches()) {
		t_on_reply("handle_answer");
		t_on_failure("missed_call");
		t_relay();
	}
}
```

The transcoding sessions on the Sangoma cards are closed in a transparent manner, based on dialog callbacks.
