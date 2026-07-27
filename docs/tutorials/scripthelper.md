---
title: Script Helper usage
description: The purpose of this new module is to help you start in an easy way
  with the OpenSIPS script. The learning curve gets milder as the beginner...
subtitle: Script Helper
subtitleHref: /modules/devel/script_helper
---
## Tutorial Overview

The purpose of this module is to help you start in an easy way with the OpenSIPS script. The learning curve gets milder as the beginners can start with a simplified format of the script - the idea is to shift the main focus on routing the SIP initial requests (which define the service logic), while transparently handling a lot of "standard" SIP scripting logic, in order to both save script coding time and mitigate potential SIP scripting errors. This tutorial offers both a brief overview on the features of the new module and an example script.

## Current features

As stated in the [documentation](/modules/devel/script_helper), the module currently features:

* **transparent handling of SIP sequential requests**

  * sequential requests are properly and silently handled - they will not hit the script at all anymore
  * for initial SIP requests, the module performs record-routing (i.e. OpenSIPS adds itself in the signaling path using a *Record-Route* header), after which the main route is triggered, just as before
  * for sequential SIP requests, the module is automatically doing loose_route() for proper routing.
  * optionally a special route may be run just before all sequential requests are routed out
* **automatic dialog support**

  * automatically create dialog support for the INVITE-based sessions
  * for sequential requests within a dialog, a complete dialog matching (Call-ID, From-tag, To-tag) will be attempted if the quick 'did'-based Record-Route parameter matching fails

Future directions for the module include adding authentication and NAT support.

## Example script

SIP logic provided:

* simple standard SIP-2-SIP calling (with registering)
* initial and sequential SIP request routing (using the script helper)
* dialog support (using the script helper)

### Scripting examples

This is the OpenSIPS script with the script_helper support.

[OpenSIPS script with the script_helper support](../modules/devel/.samples/script_helper/samples/opensips-script-helper.cfg "include")

You can check it against the regular OpenSIPS script without the script_helper support.

[regular OpenSIPS script without the script_helper support](../modules/devel/.samples/script_helper/samples/opensips-regular.cfg "include")

### Scripts differences

```diff
--- modules/script_helper/samples/opensips-regular.cfg	2026-07-21 13:11:53.483884563 +0300
+++ modules/script_helper/samples/opensips-script-helper.cfg	2026-07-21 13:11:47.175918222 +0300
@@ -95,6 +95,13 @@
 loadmodule "dialog.so"
 modparam("dialog", "db_mode", 0) # no DB

+#### SCRIPT HELPER module
+loadmodule "script_helper.so"
+# this route will be run right before sequential requests are routed out
+modparam("script_helper", "sequential_route", "my_seq_route")
+modparam("script_helper", "use_dialog", 1)
+modparam("script_helper", "create_dialog_flags", "bye-on-timeout,options-ping-caller,options-ping-callee")
+
 loadmodule "proto_udp.so"
 
 
@@ -109,44 +116,6 @@
 		exit;
 	}
 
-	if (has_totag()) {
-
-		# handle hop-by-hop ACK (no routing required)
-		if ( is_method("ACK") && t_check_trans() ) {
-			t_relay();
-			exit;
-		}
-
-		# sequential request within a dialog should
-		# take the path determined by record-routing
-		if ( !loose_route() ) {
-			# we do record-routing for all our traffic, so we should not
-			# receive any sequential requests without Route hdr.
-			send_reply(404,"Not here");
-			exit;
-		}
-
-		if (is_method("BYE")) {
-			# do accounting even if the transaction fails
-			do_accounting("log","failed");
-		}
-
-		# route it out to whatever destination was set by loose_route()
-		# in $du (destination URI).
-		route(relay);
-		exit;
-	}
-
-	# CANCEL processing
-	if (is_method("CANCEL")) {
-		if (t_check_trans())
-			t_relay();
-		exit;
-	}
-
-	# absorb retransmissions, but do not create transaction
-	t_check_trans();
-
 	if ( !(is_method("REGISTER")  ) ) {
 		if (is_myself("$fd")) {
 					
@@ -159,19 +128,6 @@
 		}
 	}
 
-	# preloaded route checking
-	if (loose_route()) {
-		xlog("L_ERR",
-			"Attempt to route with preloaded Route's [$fu/$tu/$ru/$ci]");
-		if (!is_method("ACK"))
-			send_reply(403,"Preload Route denied");
-		exit;
-	}
-
-	# record routing
-	if (!is_method("REGISTER|MESSAGE"))
-		record_route();
-
 	# account only INVITEs
 	if (is_method("INVITE")) {
 		do_accounting("log","cdr");
@@ -214,6 +170,13 @@
 }
 
 
+route [my_seq_route]
+{
+	xlog("---- Sequential request handler ---- \n");
+	xlog("$rm | Call-ID: $ci | From-tag: $ft | To-tag: $tt\n");
+}
+
+
 route[relay] {
 	# for INVITEs enable some additional helper routes
 	if (is_method("INVITE")) {
```
