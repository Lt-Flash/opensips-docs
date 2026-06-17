---
title: Tips and FAQ
description: How to do certain things.
---
## Tips

How to do certain things.

### How to recalculate ha1 and ha1b

When you change the domain column in the subscriber table, you have to recalculate ha1 and ha1b fields. In order to do that you must have the password of each subscriber. It is stored in the 'password' column if you have set STORE_PLAINTEXT_PW=1 in opensipsctlrc (default).

HA1 is a MD5 hash of "username:domain:password". For example, if you have created a SIP account [1000@mydomain.com](mailto:1000@mydomain.com) using password 123456, then HA1 is the MD5 hash of "1000:mydomain.com:123456" (without quotes). On the other hand HA1B is the MD5 hash of "username@domain:domain:password"; so using the same example above, HA1B would be the MD5 hash of "[1000@mydomain.com](mailto:1000@mydomain.com):mydomain.com:123456" (without quotes).

To recalculate and update ha1 and ha1b columns in the subscriber table, just execute the following sql statement in mysql:

```sql
update subscriber
set ha1 = md5(concat(username, ':', domain, ':', password)),
ha1b = md5(concat(username, '@', domain, ':', domain, ':', password))
```

> \[!NOTE]
> the above is only true if you have "use_domain" enabled for auth_db *and* you do not use a static challenge parameter for www_authorize.

If you use a static challenge for www_authorize (i.e. the first parameter of www_authorize is not the empty string), then HA1 is MD5("username:challenge:password") and HA1B is MD5("username@challenge:challenge:password"). If the challenge parameter of www_authorize is empty, OpenSIPS automatically selects the domain as the challenge value, which gives the solution presented above.

If "use_domain" is false, then the HA1B field must be computed based on "username@:domain:password" or "username@:challenge:password", depending on whether challenge is empty or defined, respectively.

### PHP example to communicate via mi_xmlrpc

```php
#!/usr/bin/php
<?php
$params[] = "dialog:";
$params[] = "tm:";
$method = "get_statistics";
$request = xmlrpc_encode_request($method,$params);

$context = stream_context_create(array('http' => array(
   'method' => "POST",
   'header' => "Content-Type: text/xml",
   'content' => $request
)));
$file = file_get_contents("http://example.opensips.org:8080/RPC2", false, $context);
$response = xmlrpc_decode($file);
if (is_array($response)) {
   trigger_error("xmlrpc: $response[faultString] ($response[faultCode])");
} else {
   print_r($response);
}
?>
```

## Capturing SIP traffic with ngrep

Quickly sniff SIP signalling on the wire — useful for confirming what's actually being sent/received without a full capture tool.

```bash
### capture all SIP packages on 5060 on all interfaces
ngrep -W byline -td any . port 5060

### capture all SIP packages containing 'username' on port 5060 on all interfaces
ngrep -W byline -tqd any username port 5060
```

## Capturing/analysing RTP & RTCP with tshark

Use tshark (the CLI side of Wireshark) for media-plane inspection and for piping a remote capture into a local Wireshark.

```bash
### RTCP packets reporting any packet loss, or jitter above 30 (RTP timestamp units)
tshark -i eth0 -o "rtcp.heuristic_rtcp: TRUE" \
  -Y 'rtcp.ssrc.fraction >= 1 or rtcp.ssrc.jitter >= 30' -V

### watch a remote capture live in a local Wireshark
wireshark -k -i <(ssh -l root 192.168.10.98 tshark -w - not tcp port 22)
```

- - -

## FAQS

Answers to the most frequent questions

### Changing FROM header

Q) How does one change the FROM Header in OpenSIPS ?

A) Use the **uac_replace_from()** function in the **uac** module

### Recalculating HA1 and HA1B hashes when adding a subscriber

Q) How do I re-calculate the ha1 and halb values in the subscriber table for a new subscriber?

A)  Run the following mysql operation:

update subscriber set ha1 = md5(concat(username, ':', domain, ':', password)), ha1b = md5(concat(username, '@', domain, ':', domain, ':', password))

This will automatically update the ha1 and ha1b fields for all subscribers.

### Obsolete modules

Q) Some of the modules I was using prior to **OpenSIPS 2.1** no longer exist. What happened to them?

A) Starting from **OpenSIPS 2.1** we obsoleted some of the old modules (i.e mi_xmlrpc, closeddial, auth_diameter). These modules were not completely removed, but only moved in the `modules_obsolete` directory and are no longer compiled by default. If you **really** need these modules, you can manually move them to the `modules` directory.

### Migrate to MI_XMLRPC_NG module

Q) How can I migrate from the MI_XMLRPC module to MI_XMLRPC_NG

A) The first step is to remove the `loadmodule "mi_xmlrpc.so"` directive from your script and load the `httpd` and `mi_xmlrpc_ng` modules:

```c
loadmodule "httpd.so"
loadmodule "mi_xmlrpc_ng.so"
```

Next, you have to change the default path of the module to `RPC2`, the port the server is listening on to `8080` and the buffer size if specified.

```c
modparam("mi_xmlrpc_ng", "mi_xmlrpc_ng_root", "RPC2")
modparam("httpd", "port", 8080)
modparam("httpd", "buf_size", 524288)
```

> \[!Note]
> The `reply_option` and `log_file` parameters have no equivalent for the new module, so if you were using these options, you have to revise your scripts logic.



These are only some roughly steps that you have to follow. A full migration guide is highly dependent on your usage.
