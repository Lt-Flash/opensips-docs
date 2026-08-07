---
title: "Event Interface"
description: "The OpenSIPS Event Interface provides a common way to notify external applications whenever internal events occur."
---

## Event Interface

The OpenSIPS Event Interface provides a common way to notify external applications or OpenSIPS script logic when certain events occur. It has two sides: an internal interface used by OpenSIPS core, modules and script logic to advertise and raise events, and a subscriber interface used to register interest in those events and receive notifications when they are triggered.

---

## Overview

OpenSIPS can notify external applications or internal script logic using event modules and core route hooks:

* [event_datagram](/manual/devel/modules/event_datagram) - sends event notifications over UDP or UNIX datagram sockets.
* [event_flatstore](/manual/devel/modules/event_flatstore) - writes event notifications to flat files.
* [event_kafka](/manual/devel/modules/event_kafka) - sends event notifications to Apache Kafka.
* [event_rabbitmq](/manual/devel/modules/event_rabbitmq) - sends event notifications to a RabbitMQ server.
* [event_routing](/manual/devel/modules/event_routing) - uses Event Interface notifications for internal event-based routing.
* [event_sqs](/manual/devel/modules/event_sqs) - sends event notifications to Amazon SQS.
* [event_stream](/manual/devel/modules/event_stream) - exchanges event notifications over TCP streams.
* [event_virtual](/manual/devel/modules/event_virtual) - groups multiple event backends behind a virtual subscriber.
* [event_xmlrpc](/manual/devel/modules/event_xmlrpc) - issues XML-RPC commands to an XML-RPC server.
* route - runs `event_route[...]` blocks directly in the core script engine.

In order to receive an event notification, an external application subscribes to a specific event. This can be done either using a Management Interface (MI) command, or directly from the OpenSIPS script. The subscription specifies the event name and a destination where notifications have to be sent. The destination format depends on the event module used, so check the module documentation for the exact syntax.

When an event is triggered, the Event Interface sends a notification to each subscriber registered for that event, using the selected event module or route hook. The notification body and delivery semantics are module-specific.

---

## Event Subscription

An external application can subscribe to an event using MI commands or directly from the OpenSIPS script. MI subscriptions are useful for dynamic subscribers, while script subscriptions are useful for subscribers that should be registered by OpenSIPS itself.

### MI Subscription

An external application should subscribe for an event using MI commands when the subscription is dynamic: it subscribes for a short period, receives notifications, then renews or removes the subscription. The current Core MI command used to subscribe for an event is [evi:subscribe](/manual/devel/interface-coremi).

For example, to subscribe for the `E_PIKE_BLOCKED` event for 1200 seconds, an external application that listens on localhost UDP port 8888 issues the following MI command:

```bash
$ opensips-mi evi:subscribe E_PIKE_BLOCKED udp:127.0.0.1:8888 1200
```

The first parameter is the event name, the second parameter is the subscriber destination and the last parameter is the expiration period expressed in seconds. If the last parameter is missing, a default subscription period of 3600 seconds is assumed.

In order to unsubscribe from an event, the external application uses the same event and destination used at subscription time, but specifies an expiration time of `0`:

```bash
$ opensips-mi evi:subscribe E_PIKE_BLOCKED udp:127.0.0.1:8888 0
```

The [evi:subscribers](/manual/devel/interface-coremi) MI command lists the active subscribers. It can also filter subscribers by event name and destination:

```bash
$ opensips-mi evi:subscribers
$ opensips-mi evi:subscribers E_PIKE_BLOCKED
```

### Script Subscription

The Event Interface also allows the script writer to subscribe to an event directly from the OpenSIPS configuration script. This is usually used when the subscription is managed by OpenSIPS, for example when a server continuously listens for OpenSIPS events and processes them externally. The script function used for this is [subscribe_event](/manual/devel/script-corefunctions).

Permanent subscriptions are usually registered from `startup_route`, but this is not a requirement, as the function can be used from any route.

Here is an example that subscribes to `E_PIKE_BLOCKED` at OpenSIPS startup, without an expiration time:

```opensips
startup_route() {
   subscribe_event("E_PIKE_BLOCKED", "udp:127.0.0.1:8888");
}
```

If the subscription should expire after 1200 seconds, specify the third parameter of [subscribe_event](/manual/devel/script-corefunctions):

```opensips
timer_route[refresh_pike_subscription, 3600] {
   subscribe_event("E_PIKE_BLOCKED", "udp:127.0.0.1:8888", 1200);
}
```

---

## Events

There are two types of events: static or predefined events exported by OpenSIPS core and modules, and dynamic events raised directly from the OpenSIPS script. A few examples of predefined events are:

* [E_PIKE_BLOCKED](/manual/devel/modules/pike#id250185) - raised when the **pike** module decides an IP should be blocked.
* [E_RTPPROXY_STATUS](/manual/devel/modules/rtpproxy#id293509) - raised when a RTPProxy connects or disconnects.
* [E_DISPATCHER_STATUS](/manual/devel/modules/dispatcher#id293894) - raised by the **dispatcher** module when a destination address becomes active or inactive.
* [E_CORE_THRESHOLD](/manual/devel/interface-coreevents) - raised when debugging bottleneck detection is enabled and the configured limit is exceeded.
* [E_CORE_PKG_THRESHOLD](/manual/devel/interface-coreevents) - raised when private memory usage exceeds a configured limit.
* [E_CORE_SHM_THRESHOLD](/manual/devel/interface-coreevents) - raised when shared memory usage exceeds a configured limit.

### Event Names

Although there is no strict event naming enforced by the Event Interface, it is good practice for an event name to follow the `E_MODULE_NAME` format, where `MODULE` is the name of the module that exports the event, `CORE` if the event is exported by OpenSIPS core, or `SCRIPT` if it is a dynamic event raised from the script.

### Raising an Event

In order to raise an event from the script, use the [raise_event](/manual/devel/script-corefunctions) function. The function receives one, two or three parameters, depending on the script writer's needs. If the event has no parameters, only the event name is needed.

If the script writer wants to attach extra information to the event, the second and optionally the third parameter have to be used. The values are stored in an AVP, for example:

```opensips
$avp(values) = "value1";
$avp(values) = "value2";
...
```

If the parameters also have names, another AVP has to store the attribute names. The number of values in the values AVP has to be the same as the number of attributes:

```opensips
$avp(attributes) = "param1";
$avp(values) = "value1";
$avp(attributes) = "param2";
$avp(values) = "value2";
...
```

The following examples show how to raise the event from the script:

```opensips
raise_event("E_SCRIPT_EVENT");

raise_event("E_SCRIPT_EVENT", $avp(values));

raise_event("E_SCRIPT_EVENT", $avp(attributes), $avp(values));
```

The [evi:raise](/manual/devel/interface-coremi) MI command can also raise an event through the Event Interface:

```bash
$ opensips-mi evi:raise -j '{"event":"E_SCRIPT_EVENT","params":{"source":"mi"}}'
```

### Events List

The [evi:list](/manual/devel/interface-coremi) MI command lists all available events:

```bash
$ opensips-mi evi:list
```

### Examples

A practical use case of dynamic events is raising an event from the script when the TCP load is greater than 80%:

```opensips
route[TCP_LOAD_CHECK] {
   if ($stat(tcp-load) > 80) {
      # raise the event, specifying the limit and the real load
      $avp(attrs) = "limit";
      $avp(vals) = 80;
      $avp(attrs) = "load";
      $avp(vals) = $stat(tcp-load);

      raise_event("E_SCRIPT_TCP_LOAD", $avp(attrs), $avp(vals));
   }
}
```

Dynamic events can also be used for accounting purposes. For example, you could use three types of dynamic events to notify an external application about a call status:

```opensips
# this route initializes the accounting variables
route[ACC_VARS] {
   # first cleanup whatever it was there before this
   $avp(attrs) := NULL;
   $avp(vals) := NULL;
   $avp(attrs) = "from";
   $avp(vals) = $fU;
   if (has_totag()) {
      $avp(attrs) = "to";
      $avp(vals) = $rU;
   }
   ...
   $avp(attrs) = "callid";
   $avp(vals) = $ci;
   # populate attrs and vals with whatever extra info you need
}

route {
   ...
   if (!has_totag() && is_method("INVITE")) {
      route(ACC_VARS);
      raise_event("E_SCRIPT_CALL_START", $avp(attrs), $avp(vals));
      t_on_reply("acc");
   } else if (is_method("CANCEL|BYE")) {
      route(ACC_VARS);
      raise_event("E_SCRIPT_CALL_STOP", $avp(attrs), $avp(vals));
   }
   ...
}

onreply_route[acc] {
   ...
   if (t_check_status("200")) {
      route(ACC_VARS);
      raise_event("E_SCRIPT_CALL_ESTABLISHED", $avp(attrs), $avp(vals));
   }
   ...
}
```

---

## Event Modules and Core Routes

There are several event modules that can be used to notify an external application or to wire Event Interface notifications into other OpenSIPS subsystems.

### Datagram Messages

External applications can be notified when OpenSIPS internal events are triggered using either UDP or UNIX datagrams. This allows programmers to write a simple UDP or UNIX datagram server that receives the notification and processes it in the preferred programming language.

In order to support this type of communication, load the [event_datagram](/manual/devel/modules/event_datagram) module. Consult the module documentation page for more information about the subscription socket and encapsulation format.

### Flatstore Logs

The [event_flatstore](/manual/devel/modules/event_flatstore) module writes event notifications and their parameters to text files. It is useful when events have to be consumed later by log processors or other local tooling.

### Kafka Messages

The [event_kafka](/manual/devel/modules/event_kafka) module sends event notifications to Apache Kafka. It can be used when OpenSIPS events have to be integrated with a Kafka-based processing pipeline.

### RabbitMQ Messages

OpenSIPS can communicate with a [RabbitMQ](http://www.rabbitmq.com/) server using AMQP messages. This type of communication leverages the queuing features provided by the server.

The [event_rabbitmq](/manual/devel/modules/event_rabbitmq) module provides this functionality. In this case, the RabbitMQ server is the event backend, so subscriptions are usually managed from the OpenSIPS script or by a separate control application. For more information, consult the module documentation page.

### Event Routing

The [event_routing](/manual/devel/modules/event_routing) module provides event-based routing capabilities. It is useful when SIP processing needs to wait for, react to or trigger logic based on Event Interface notifications.

Do not confuse this module with core `event_route[...]` handlers. The old route capabilities are now part of OpenSIPS core and are described below.

### SQS Messages

The [event_sqs](/manual/devel/modules/event_sqs) module sends event notifications to Amazon SQS. It can be used when OpenSIPS events have to be consumed by AWS-hosted workers or queues.

### Stream Messages

The [event_stream](/manual/devel/modules/event_stream) module exchanges event notifications over TCP streams. It is useful for applications that keep a persistent TCP connection to OpenSIPS.

### Virtual Subscribers

The [event_virtual](/manual/devel/modules/event_virtual) module groups multiple event backends behind a virtual subscriber. It can be used to fan out, load-balance or provide failover between real event subscribers.

### XML-RPC Commands

OpenSIPS can also execute a remote XML-RPC command when an event is triggered. The module that handles the communication between the Event Interface and the XML-RPC server is [event_xmlrpc](/manual/devel/modules/event_xmlrpc).

### Core Event Routes

OpenSIPS can handle events directly from the script using [event_route](/manual/devel/script-routes) blocks. This allows the script writer to react to Event Interface notifications without using an external transport module. The route name specifies the event handled by that route.

Older examples may use the `fetch_event_params()` function to retrieve event parameters. That function was deprecated and later removed. In current OpenSIPS versions, read event parameters from the `$params` variable inside the event route.

For example, in order to handle the `E_PIKE_BLOCKED` event and print the blocked IP, the OpenSIPS script can contain the following code:

```opensips
event_route[E_PIKE_BLOCKED] {
   xlog("IP $params(ip) has been blocked\n");
}
```

Defining the `event_route` automatically subscribes the route to the event, so no explicit external or script subscription is needed for that route.

---

## Older Version Notes

### MI Command Names

> [!IMPORTANT]
> The `evi:*` MI namespace was introduced in OpenSIPS 4.0. Older versions use the previous flat MI command names.

| 4.0+ MI command | Older MI command |
| --- | --- |
| `evi:subscribe` | `event_subscribe` |
| `evi:list` | `events_list` |
| `evi:subscribers` | `subscribers_list` |
| `evi:raise` | `raise_event` |

### Event Route Parameters

The `fetch_event_params()` function was removed in OpenSIPS 3.0. Older `event_route[...]` examples may use it to copy event parameters into script variables. Current versions should read event parameters directly from `$params(name)`.

### OpenSIPS 1.8

OpenSIPS 1.8 uses the same Event Interface concept, but with the event modules and MI command names available in that branch. For a 1.8 deployment, load the supported event transport module, subscribe to the event either through MI or with `subscribe_event()` from script, and raise script events with `raise_event()`.

For 1.8 MI subscriptions, use `event_subscribe <event> <destination> [expires]`; setting `expires` to `0` removes the subscription. Do not confuse the older MI `raise_event` command with the script function `raise_event()`, which is still the script-side API shown above.
