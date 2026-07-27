---
title: Topology Hiding
description: The purpose of this tutorial is to help you understand how the
  topology_hiding module works and also how it should be used.
subtitle: Topology Hiding
subtitleHref: /modules/devel/topology_hiding
---
## Tutorial Overview

The purpose of this tutorial is to help you understand how the topology_hiding module works and also how it should be used.   

Topology hiding is usually utilized as an approach to enhance SIP network security. Since, in regular SIP traffic, critical IP address data is forwarded to other networks, the concern is that third parties can use that information in order to direct attacks at your internal SIP network.

## Current Features

By default, when engaged, the module will hide the VIA, Record-Route, Route and Contact headers. VIA, Record-Route and Route headers will be fully removed when routing message from one side to the other, while the Contact header will be mangled to point to the outgoing interface IP.   

Optionally, the module can also hide the Call-ID on the outbound side, since often enough the Call-ID will contain the IP of the traffic originator.    

Also, the topology_hiding module can help ease integration by propagating various parts of the Contact header from one side to the other. The Contact username can be propagated ( eg. useful when you are dipping against LRN servers, but don't want to disclose your full topology to the service provider ). Also, various Contact header parameters can be passed, in order to allow for end-to-end features that rely on such params.   

See the [full module README](/modules/head/topology_hiding) for all the available parameters and functions.

## How It Works

The module can work on top of the dialog and TM modules, or just on top of the TM module.

When running strictly on top of the TM module, the topology hiding SIP messages will be bigger when compared to the initial requests ( since OpenSIPS will encode all the needed information in a parameter of the Contact header ), but all type of SIP requests and dialogs will be supported ( INVITE dialogs, Presence dialogs, SIP MESSAGE, etc ).

When running on top of the DIALOG module, you will get shorter messages ( all the removed headers will be kept internally in the dialog module ) , but you will only be able to hide INVITE based dialogs.

## Example Script

```opensips
loadmodule "topology_hiding.so"

route {

    ....
    ....
    ....

    if (has_totag()) {

        # handle hop-by-hop ACK (no routing required)
        if ( is_method("ACK") && t_check_trans() ) {
            t_relay();
            exit;
        }

        if (!topology_hiding_match()) {
            send_reply(404,"Not here");
            exit;
        }

        xlog("Succesfully matched this request to a topology hiding dialog. \n");
        xlog("Calller side callid is $ci \n");
        xlog("Callee side callid  is $TH_callee_callid \n");
        t_relay();
        exit;
        
    }

    ....
    ....
    ....

    # if it's an INVITE dialog
    if (is_method("INVITE")) {

        # we can create the dialog now
        create_dialog();
        
        # we do topology hiding, preserving the Contact Username and also hiding the Call-ID
        topology_hiding("UC");
    }
    
    ...
    
    t_relay();
    exit;
}
```
