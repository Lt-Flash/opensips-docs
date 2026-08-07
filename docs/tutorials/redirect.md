---
title: SIP Redirect Server
description: How to use OpenSIPS as a SIP redirect server, know-how and examples
---
A SIP Redirect Server is a SIP server replying with a 30x code having a **CONTACT** that carries a list of SIP URIs to be used as alternatives to the URI indicated by the received request.

OpenSIPS can be configured a SIP Redirect server. There are two simple ways to generate, from script level, the **CONTACT** header to be inserted into the 30x reply:

* build the CONTACT header manually, via string operations, at script level
* use the message branches, via the [$msg.branch](/manual/devel/script-corevar/#msg.branch) family of variables

 The replying part is simple, it can be done with the**[ t_reply()](< /manual/devel/modules/tm/#func_t_reply>)** function provided by the **TM** module. Of course, you can also do it in a stateless way, using the **[sl_send_reply()](/manual/devel/modules/sl/#func_sl_send_reply)** function from the **SL** module.

```opensips
t_reply( 302, "Temporary redirected");
```

You can redirect to one or more SIP URIs (in the **CONTACT** header). The SIP URIs may be the end result of a custom logic you have in the script, like loading them from a database, querying from a REST HTTP service, fetching from a registrar database , etc.

## Building the CONTACT manually

In this snippet I am restricting the redirect service only for INVITE requests and by a IP range (just for the sake of the example) .

Here we assume we load the SIP URIs from a DB, using the [sql_query()](/manual/devel/modules/sqlops/#func_sql_query) function. In this example we query based on the received R-URI username, trying to find a list of SIP gateways (as SIP URI's) where the call should be redirected to (kind a simple LCR).

```opensips
route {
    if ( $(si{ip.matches,192.168.1.1/24})==1 && $rm=="INVITE") {
    
        # the the SQL query to load the set of GW + PRIOrities (Q value)
        sql_query("select gw_uri,gw_prio from lcr_table where destination_did='$rU'",
  "$avp(gw_uri);$avp(gw_prio)");
        if ($rc<0) {
            t_reply( 404, "Destination not found");
            exit;
        }
        
        # here we have a set of GW URIs and GW PRIOs (same size) to be added
        # to the contact hdr.
        $var(i) = 0;
        $var(hdr) = "";
        while ( $(avp(gw_uri)[$var(i)])!=NULL ) {

            # we add each set of GW and PRIO as ", <gw_uri>;q=val"
            if ($var(i)!=0)
              $var(hdr) = $var(hdr) + ", ";
            $var(hdr) = $var(hdr) + "<"+$(avp(gw_uri)[$var(i)])+">;q="+$(avp(gw_prio)[$var(i)]);
            
            # move to the next GW + PRIO set
            $var(i) = $var(i) + 1;
        }
        
        # add the Contact hdr
        append_to_reply("Contact: $var(hdr)\r\n");
        
        # send the reply now
        t_reply( 302,"LCR Redirect");
    } else {
        send_reply( 403, "You are not allowed here!" );
    }
    exit;
}
```

## Building the CONTACT via $msg.branch

The internal core function responsible for building the SIP replies (both stateful and stateless) has an interesting, largely unknown, behavior: 

*If any message branches are present, it will automatically push them into the CONTACT hdr, if building a 30x reply*. 

So, you could simply push the new URIs into message branches and they will be automatically pushed further in the 30x reply CONTACT.

Also there are several modules which queries and returns new multiple SIP branches (SIP URIs) via this message branches (see the **[lookup("location")](/manual/devel/modules/registrar/#func_lookup)** for SIP registration lookup, or the **[alias_db_lookup()](modules/devel/alias_db/#func_alias_db_lookup)** )

```opensips
route {
    if ( $(si{ip.matches,192.168.1.1/24})==1 && $rm=="INVITE") {
    
        # do the registrations lookup, to find ALL user's registrations
        if ( lookup( "location", "to-branches-only") ) {
        
            # user's registrations are already pushed into message
            # branches, with Q values, sockets and everything
            xlog("found branches are < $(msg.branch.uri[*]) >\n");
        }
        
        # just send the reply now, the message branches will automatically
        # be pushed as CONTACT in the reply
        t_reply( 302,"LCR Redirect");
    } else {
        send_reply( 403, "You are not allowed here!" );
    }
    exit;
}
```

Also, if we go back to the first example (with manual building of the CONTACT), the would be the corresponding script but using the message branches.

```opensips
route {
    if ( $(si{ip.matches,192.168.1.1/24})==1 && $rm=="INVITE") {
    
        # the the SQL query to load the set of GW + PRIOrities (Q value)
        sql_query("select gw_uri,gw_prio from lcr_table where destination_did='$rU'",
  "$avp(gw_uri);$avp(gw_prio)");
        if ($rc<0) {
            t_reply( 404, "Destination not found");
            exit;
        }
        
        # here we have a set of GW URIs and GW PRIOs (same size) to be added
        # to the contact hdr.
        while ( $avp(gw_uri)!=NULL ) {
        
            # add the SIP URI
            $msg.branch = $avp(gw_uri);
            # and the Q value
            $msg.branch.q = $avp(gw_prio);
            
            # delete the last used values
            $avp(gw_uri) = NULL;
            $avp(gw_prio) = NULL;
            
        }
        
        # send the reply now
        t_reply( 302,"LCR Redirect");
    } else {
        send_reply( 403, "You are not allowed here!" );
    }
    exit;
}
```
