---
title: Concurrent calls limitation
description: Implementing Concurrent Call limitation using OpenSIPS dialog
  support with profiles.
---
OpenSIPS can implement concurrent calls limitation by using the *call profiles* support provided by the **[dialog](/modules/devel/dialog)** module.  The call profiles are a simple mechanism that allows to group certain calls and to count them. The grouping can be extended by using *labels* (values inside a group). This allows a better granularity when comes to counting calls. 

Different keys can be used to group calls (like the incoming IP address, the caller user, the outbound GW, etc). Once we create a profile for such a group of call, when handling the call in the OpenSIPS script, we add the call to such a group, eventually with an extra label (where the label is the actual value of the incoming IP or of the caller or of the GW).

In the below example we create a generic route to implement the limitation of concurrent calls from the same caller. The *route* takes two parameters, the caller SIP username and the value of the limit. This *route* is to be used when calls are created (when initial INVITEs are handled) after the dialog was created in script.  

```opensips
....
# define the profile
modparam("dialog", "profiles_with_value", "caller")
....

########################################################################
# This route is to be called for initial INVITEs, after 
#   the dialog was created
# Parameters:
#    * the caller URI (string)
#    * the limit (integer)
########################################################################
route[do_cc_limit]
{
	# first add to the profile, just to avoid "test and set" false results
	set_dlg_profile("caller", $param(1));

	# do the actual test - see how many calls the user has so far
	get_profile_size("caller", $param(1), $var(calls));
	xlog("User $param(1) has $var(calls) ongoing calls so far, limit is $param(2)\n");

	# check within limit
	if( $var(calls)>$param(2) )
	{	
		xlog("do_limit: user $param(1) exceeded number of calls [$var(calls)/$param(2)]\n");
		send_reply(487, "Request Terminated: Channel limit exceeded\n");
		exit;
		# terminating this call will automatically remove the call from the profile
	}

	# call was added to the profile without exceeding the limit, simply continue
}
....
....
route {
	.....
	# initial INVITES
	route( do_cc_limit, $fu, 10);
	.....
}
```

Here you can find a **[fully functional basic script](https://github.com/OpenSIPS/opensips/tree/master/modules/dialog/samples/opensips-dialog-cc-limit.cfg)** (end users registering and calling SIP to SIP) where this concurrent call limitation is used.
