---
title: "WebSocket Transport using OpenSIPS"
subtitle: "WebSocket Transport"
subtitleHref: "/modules/devel/proto_ws"
description: "This document describes how to use OpenSIPS as the core component of a SIP platform that connects both SIP clients (over UDP, TCP or TLS) as well as browser..."
---

> [!NOTE]
> Other versions: [OpenSIPS 2.1 version](/tutorials-websocket-2-1), [older than OpenSIPS 2.1](/tutorials-websocket-older).

## Tutorial Overview

[WebSocket](https://datatracker.ietf.org/doc/html/rfc6455) is a protocol that provides full-duplex communication between web clients and servers over TCP connections. Using the WebSocket protocol, browsers can connect to web servers and exchange data, regardless the type or nature of the application protocol. [RFC 7118](https://datatracker.ietf.org/doc/html/rfc7118) leveraged this protocol in order to allow browsers to make VoIP calls using the SIP protocol. [WebSocketSecure](/modules/devel/proto_wss) (WSS) overlays TLS onto the WebSocket protocol, making the connection secure, a requirement for many browsers if you want to do WebRTC.

This document describes how to use **OpenSIPS** as the core component of a SIP platform that connects both SIP clients (over UDP, TCP or TLS) as well as browser based clients using SIP over WebSockets and WebSocketsSecure. While OpenSIPS handles the SIP signalling part, media is relayed through [RTPengine](https://github.com/sipwise/rtpengine), with OpenSIPS using the [rtp_relay](/modules/devel/rtp_relay) module to keep the media handling tied to the call context.

## Setup

### RTPengine

#### Installation

The RTPengine consists of two main components: a kernel module used to efficiently route the RTP packets directly in kernel, and a daemon used to communicate with OpenSIPS. You can find more details [here](https://github.com/sipwise/rtpengine#overview). Both components can be installed from debs (on Debian based systems) or directly from sources. Simply follow the [official documentation](https://github.com/sipwise/rtpengine#compiling-and-installing) to install RTPengine.

You must generate certificates to use with TLS and WSS. For this example we are generating certificates using [Let's Encrypt](https://letsencrypt.org/).

Certificate management is handled by the [TLS_MGM](/modules/devel/tls_mgm) module. Setting the appropriate `tls_mgm` module parameters is how we will manage our certificates for both WSS and TLS.

#### Usage

After installing the kernel module and the additional libraries, the RTPengine daemon has to be configured. Modern RTPengine deployments should keep the daemon options in an INI-style configuration file, normally `/etc/rtpengine/rtpengine.conf`. The RTPengine daemon documentation describes the [`--config-file`](https://rtpengine.readthedocs.io/en/latest/rtpengine.html#options) option, the default config file path and the available options.

The interesting settings we are using are as follows:

* `interface`: the RTP/SRTP listening interface and, optionally, the advertised address
* `listen-ng`: the IP and port used by OpenSIPS to communicate with RTPengine using the NG control protocol
* `listen-cli`: the IP and port of the CLI interface, used to gather statistics for RTP/SRTP sessions
* `port-min` and `port-max`: the local UDP port range RTPengine allocates from for media relaying
* `log-level`: the syslog verbosity level

You can find all the parameters in the [RTPengine daemon manual](https://rtpengine.readthedocs.io/en/latest/rtpengine.html).

Here is a minimal `/etc/rtpengine/rtpengine.conf` example that talks with OpenSIPS over localhost and relays RTP using the `1.1.1.1` IP:

```ini
[rtpengine]
table = 0
interface = external/1.1.1.1
listen-ng = 127.0.0.1:60000
listen-cli = 127.0.0.1:60001
port-min = 50000
port-max = 55000
log-level = 6
```

#### Troubleshoot

First make sure the `rtpengine` daemon is started:
```bash

ps -ef | grep rtpengine

```

If the `rtpengine` daemon does not start, make sure the `xt_RTPENGINE` kernel module is loaded:
```bash

lsmod | grep xt_RTPENGINE

```

If the module is not loaded, make sure the `ip_tables` and `x_tables` kernel modules are loaded. Also, check the logs for the last errors of the system
```bash

dmesg

```

### OpenSIPS

In order to use WebSocket and WebSocketSecure in OpenSIPS, one has to load the [proto_ws](/modules/devel/proto_ws) and [proto_wss](/modules/devel/proto_wss) modules into its configuration file and define a listener for the WebSocket and WebSocketSecure protocol. We also must load one TLS backend module, such as `tls_wolfssl` or `tls_openssl`, together with the [tls_mgm](/modules/devel/tls_mgm) module in order to manage our certificates.

```opensips

# set listeners for all protocols
socket=ws:127.0.0.1:8080
socket=wss:127.0.0.1:443
socket=tls:127.0.0.1:5061
socket=udp:127.0.0.1:5060

# load all protocol modules
loadmodule "proto_udp.so"
loadmodule "proto_ws.so"

# load TLS support
loadmodule "tls_wolfssl.so"
loadmodule "tls_mgm.so"
loadmodule "proto_tls.so"
loadmodule "proto_wss.so"

# modparam our certificate information
modparam("tls_mgm", "server_domain", "default")
modparam("tls_mgm", "match_ip_address", "[default]*")
modparam("tls_mgm", "certificate", "[default]/etc/letsencrypt/live/acme.com/fullchain.pem")
modparam("tls_mgm", "private_key", "[default]/etc/letsencrypt/live/acme.com/privkey.pem")
modparam("tls_mgm", "verify_cert", "[default]0")
modparam("tls_mgm", "require_cert", "[default]0")

```

Next, load the [rtp_relay](/modules/devel/rtp_relay) module and the [rtpengine](/modules/devel/rtpengine) backend. The `rtp_relay` module keeps the media context for the call and automatically drives the offer, answer and delete operations through the selected backend.

```opensips

loadmodule "rtp_relay.so"
loadmodule "rtpengine.so"
modparam("rtpengine", "rtpengine_sock", "udp:127.0.0.1:60000")

```

Note that the [rtpengine_sock](/modules/devel/rtpengine#param_rtpengine_sock) parameter should match the `listen-ng` setting from the `rtpengine` daemon configuration, and OpenSIPS should have IP connectivity to that socket.

Next, the routing logic has to be changed in order to treat clients that use DTLS-SRTP differently from clients that use plain RTP and enable bridging if necessary. To do that, one can check if the request message was received over the WebSocket protocol. This can be achieved using the following code:

```opensips

if ($socket_in(proto) == "WS" || $socket_in(proto) == "WSS")
    setflag("SRC_WS");

```

In case the request is a REGISTER, we want to store this information in the *location* table, so that we know the transport type when the user is called. To do that, we can set a branch flag before calling the [save()](/modules/devel/registrar#func_save) function. This way, when the [lookup()](/modules/devel/registrar#func_lookup) method returns, we will be able to determine whether the client uses WebSocket or not.

```opensips

    if (is_method("REGISTER")) {
        if (isflagset("SRC_WS"))
            setbflag("DST_WS");

        fix_nated_register();
        if (!save("location"))
            send_reply(500, "Registration failed");

        exit;
    }

```

When a call is placed, based on the two flags (`SRC_WS` and `DST_WS`) we can determine what our caller and callee can "speak" (either RTP or DTLS-SRTP). The current script provisions the media type for each side through the [`$rtp_relay`](/modules/devel/rtp_relay#pv_rtp_relay) and [`$rtp_relay_peer`](/modules/devel/rtp_relay#pv_rtp_relay_peer) variables, then engages RTPengine through [rtp_relay_engage()](/modules/devel/rtp_relay#func_rtp_relay_engage).

```opensips

    if (isbflagset("DST_WS"))
        $rtp_relay(type) = "UDP/TLS/RTP/SAVPF ICE=force";
    else
        $rtp_relay(type) = "RTP/AVP ICE=remove";

    if (isflagset("SRC_WS"))
        $rtp_relay_peer(type) = "UDP/TLS/RTP/SAVPF ICE=force";
    else
        $rtp_relay_peer(type) = "RTP/AVP ICE=remove";

    if (!has_totag())
        rtp_relay_engage("rtpengine");

```

Once `rtp_relay_engage()` is called for the initial INVITE, `rtp_relay` registers the hooks needed to handle the rest of the RTPengine interaction for the call, including in-dialog offer/answer updates and media teardown.

Having done all these settings should provide a full setup for interconnecting SIP clients over both UDP, TCP, etc. protocols, as well as browser based SIP clients over WebSocket.

## Configuration file

### Normal SDP negotiation

A full sample configuration is available in the PROTO_WSS module examples: [opensips.cfg](https://github.com/OpenSIPS/opensips/blob/master/modules/proto_wss/samples/opensips.cfg), which is a minimal working example of a residential script that can handle client connections over both UDP and WebSocket transports. This example assumes that the SDP offer is present in the INVITE from the UAC and the SDP answer is in the 200 OK from the UAS.

> [!NOTE]
> The default port for WSS (443) is privileged, so if you are running this script, you should start OpenSIPS with super-user rights (as user root).

### Late SDP negotiation

For late SDP negotiation, where SDP is exchanged between the 200 OK and ACK, use the same transport flagging and RTP relay provisioning. Since `rtp_relay` owns the call media context after `rtp_relay_engage()`, the script only has to make sure SDP-bearing in-dialog messages continue through the normal relay path.

## Older Version Notes

The devel configuration above is the reference version. Older tutorials can be merged into this document as short version notes, because the deployment model changed only in a few important ways.

### OpenSIPS 2.2

The 2.2 tutorial already used native WebSocket and Secure WebSocket support in OpenSIPS. When updating it, start from the current sample configuration, refresh the general script syntax and replace the direct RTPengine handling with the `rtp_relay` call context.

### OpenSIPS 2.1

The 2.1 tutorial covered native plain WebSocket support. Secure WebSocket support should follow the current OpenSIPS flow, using the WSS transport and TLS modules, while media handling should move to `rtp_relay` when updating to devel.

### Older Than OpenSIPS 2.1

Older deployments used OverSIP as a WebSocket-to-SIP gateway, so OpenSIPS itself did not terminate WebSocket traffic. When updating to the current version, remove that gateway role from the tutorial flow and let OpenSIPS handle WebSocket and Secure WebSocket traffic directly.
