---
title: Presence User Agent
description: OpenSIPS can act as a SIP User Agent from presence point of view -
  this means it can publish various types of presence information, like the
  status from Register Gateway to xmpp, presence Sending Publish a...
---
OpenSIPS can act as a **SIP User Agent from presence point of view**. This means it can publish, via the **PUBLISH** request,  various types of presence information to a configured SIP Presence Server. 

Architecturally speaking, the core functionality of the Presence User Agent is provided by the [pua](modules/devel/pua/) module. Note that the module does NOT provide any functionality to be used directly from the script, but it is providing this PUA client support (via an internal API) for other event-specific modules to do PUA client operations.

Depending on the type of information to be published, OpenSIPS has some presence extensions that offer  very interesting features:

* Dialog info / Busy Lamp Field (BLF, event:dialog), via the module [pua_dialoginfo](#pua_dialoginfo)
* Sending Publish and Subscribe messages using the MI interface, via the module [pua_mi](#pua_mi)
* Registration status sharing (RFC 3680), via the module [pua_reginfo](#pua_reginfo)
* Get presence status from Register, via the module [pua_usrloc](#pua_usrloc)
* Bridge Line Appearance (BLA, event: dialog;sla), via the module [pua_bla](#pua_bla)
* Gateway to xmpp presence, via the module [pua_xmpp](#pua_xmpp)

- - -

## pua_dialoginfo

This module enables **BLF feature** in the server by monitoring and publishing dialog information to a presence server. Basically this module uses the [dialog](/modules/devel/dialog) module (which is a tracker for INVITE based dialog). Callbacks are registered to be called whenever a SIP calls starts / fails / establishes / terminates / times out . These callbacks translates the new state of the dialog into an *application/dialog-info+xml* data format text which is sent to the presence server as a body for a PUBLISH message.

Then, the rest of the job is done by the presence server. On the presence server side, you need to enable the handling for this event by loading the [presence_dialoginfo](/modules/devel/presence_dialoginfo) corresponding module -  that will register the 'dialog-info' event to the presence server. The presence server will receive **SUBSCRIBE** messages from the UA's and generate **NOTIFY**'s with the information received in **PUBLISH** messages sent by the pua_dialoginfo module. 

See here a [configuration file example](https://github.com/OpenSIPS/opensips/tree/master/modules/pua_dialoginfo/samples/opensips-pua_dialoginfo.cfg).

You can read the [modules readme here](/modules/devel/pua_dialoginfo).

- - -

## pua_mi

OpenSIPS offers the possibility to publish any kind of information to the Presence Server or subscribe to the presence state of a resource through the MI interface. To use this module, you should load at least one [MI backend module](/manual/devel/interface-mi).

The available commands are:

1. **[publish](modules/devel/pua_mi/#mi_publish)**

Example:

```text
opensips-cli -x mi pua_mi:publish sip:system@opensips.org 3600 presence application/pidf+xml <?xml version='1.0'?><presence xmlns='urn:ietf:params:xml:ns:pidf' xmlns:dm='urn:ietf:params:xml:ns:pidf:data-model' xmlns:rpid='urn:ietf:params:xml:ns:pidf:rpid' xmlns:c='urn:ietf:params:xml:ns:pidf:cipid' entity='system@opensips.org'><tuple id='0x81475a0'><status><basic>open</basic></status></tuple><dm:person id='pdd748945'><rpid:activities><rpid:away/>away</rpid:activities><dm:note>CPU:16 MEM:476</dm:note></dm:person></presence>
```

This example publishes the state of a system( CPU/MEM). This feature can be used in many applications, one example is advertising and publishing news and offers through the presence status of a dummy SIP account.

2. **[subscribe](modules/devel/pua_mi/#mi_subscribe)**

```text
opensips-cli -x mi pua_mi:subscribe sip:system@opensips.org sip:400@opensips.org presence 3600
```

Here, the *sip:400@opensips.org* entity subscribes to entity *sip:system@opensips.org* for the next 3600 seconds in order to receive *presence* events.

You can read the [modules readme here](/modules/devel/pua_mi).

- - -

## pua_reginfo

This module publishes information about *reg* events according to to RFC 3680. This can be used distribute the registration-info status to the subscribed watchers.

This module **PUBLISH-**es information when a new user registers at this server (e.g. when "save()" is called) to users, which have subscribed for the reg-info for this user.

This module can **SUBSCRIBE** for information at another server, so it will receive **NOTIFY** requests, when the information about a user changes.

And finally, it can process received  **NOTIFY** requests and it will update the local registry accordingly.

Use cases for this might be:

* Keeping different Servers in Sync regarding the location database
* Get notified, when a user registers: A presence-server, which handles offline message storage for an account, would get notified, when the user comes online.
* A client could subscribe to its own registration-status, so he would get notified as soon as his account gets administratively unregistered.

You can read the [modules readme here](/modules/devel/pua_reginfo).

- - -

## pua_usrloc

There are still many SIP phones( especially hard phones) that do not implement presence support. Even so, the server is aware if the phone is online or not from whether the phone is registered or not. This information is used by this module to send basic presence information to the Presence Server about any phone.

The [pua_usrloc](/modules/devel/pua_usrloc) module hooks on top of the *usrloc* module in order to "see" when a user registers a devices (in order to publish him as *online*) or when all his registrations are removed/expired (in order to publish him as *offline*)

See here a [configuration file example](https://github.com/OpenSIPS/opensips/tree/master/modules/pua_usrloc/samples/opensips-pua_usrloc.cfg).

NOTE: It is not a must for the presence server to run on the same machine as the **pua_usrloc** extension, as the communication is through PUBLISH SIP messages. 

You can read the [modules readme here](/modules/devel/pua_usrloc).

- - -

## pua_bla

This module implements *BLA (Bridge Line Appearance)* according to draft [draft-anil-sipping-bla-03.txt](http://tools.ietf.org/draft/draft-anil-sipping-bla/draft-anil-sipping-bla-03.txt).

It also requires to monitor **REGISTER** messages which means that, similar to [pua_usrloc](#pua_usrloc), it has to be collocated with the registrar or **REGISTER** messages has to be forwarded to the server.
The BLA server works as a central entity that gathers bridge line information from the phones and then distributes it to all the phones registered to the line. 
The gathering part is done by the [pua_bla](/modules/devel/pua_bla) module, with functionalities provided by the **pua** module. It is achieved by sending **SUBSCRIBE** messages to the phones when registering and then catching the **NOTIFY's** from the phones. The **NOTIFY** is then transformed into a **PUBLISH** message and sent to the presence server.
The presence server is the one that will deal with the distributing part. It will receive the subscribe messages from the phones and using the published information sent by the [pua_bla](/modules/devel/pua_bla) module, it will send **NOTIFY's** with the state of the line.

See a [configuration example here](https://github.com/OpenSIPS/opensips/tree/master/modules/pua_bla/samples/opensips-pua_bla.cfg).

You can read the [module readme here](/modules/devel/pua_bla).

- - -

## pua_xmpp

This module implements a presence gateway between XMPP and SIP. It requires **[pua](/modules/devel/pua)** and **[xmpp](/modules/devel/xmpp)** modules.

The operations performed by **pua_xmpp** module consist in translating the presence information format from XMPP to SIP and back and also dealing with logic incompatibilities (SIP is time based while XMPP is not).
In the SIP part, the module sends **SUBSCRIBE** messages on behalf of the xmpp users and catches **NOTIFY's** in the script.
Also it detects **SUBSCRIBE's** to users in XMPP domains and sends **SUBSCRIBE** with event presence.winfo to on behalf of those users to be notified when others **SUBSCRIBE** to its presence.

This is achieved by calling the following functions in the script:

* **pua_xmpp_notify()**
* **pua_xmpp_req_winfo(char *request_uri, char* expires)**

See a [configuration example here](https://github.com/OpenSIPS/opensips/tree/master/modules/pua_xmpp/samples/opensips-pua_xmpp.cfg).

You can read the [module readme here](/modules/devel/pua_xmpp).

- - -
