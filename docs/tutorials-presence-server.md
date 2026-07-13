---
title: OpenSIPS Presence Server
description: You can see a configuration file example for this case here.
---
OpenSIPS is able to act as full SIP **Presence Server** - to share presence information between SIP parties. This means to **receive presence information** from different sources (typically via **PUBLISH** requests) , to potentially **process this information** and finally **distributed it to other parties** interested in such information - these parties are called watchers or subscribers and this is done via **SUBSCRIBE**/**NOTIFY** mechanism.  

The presence information may be of multiple types, like status info (if online, offline, buys, etc), calling info (if in a call or not), feature info (if certain features are enabled or not) .

## Presence Server Key Features

* **extensible** - new events to be handled by the server can be added easily due to the layered architecture
* **fast caching** with periodic update in database - a fallback to database mode can be configured useful when more than one server is used for a domain
* **privacy rules with XCAP storage** - a faster integrated xcap mode that uses a database as an interface with the XCAP server ( works with OpenXCAP )

- - -

## Presence Server Architecture

![presence schema](/public/uploads/presence-schema.jpg "Presence architecture")

- - -

## OpenSIPS modules

### The PRESENCE module

The [PRESENCE module](/modules/devel/presence) - this is the core component of the Presence Service engine. The module is a general, event package independent Subscribe, Publish handler – Notify generator according to RFCs 3265 and 3903.

The module maintains all the active subscriptions (SIP entities interested in receiving presence information), by  processing the **SUBSCRIBE** requests via the [handle_subscribe()](modules/devel/presence/#func_handle_subscribe) function.

When new presence information is received via **PUBLISH** requests, via the [handle_publish()](modules/devel/presence/#func_handle_publish) function, the module will match the interested subscribers and update them via the **NOTIFY** request. 

So, the PRESENCE module offers the way to sharing / exchanging the presence information (in a relative agnostic way to the actual payload)  between the SIP entities.

### The EVENT modules

Between receiving and sharing the presence information, the OpenSIPS Presence Server is also able to handle (manipulate) the presence information. In order to understand various types of presence information, OpenSIPS provides events oriented modules - modules that are able to read, merge or change the exchange presence information:

* [PRESENCE_XML](/modules/devel/presence_xml)  - works with xml bodies for events like *presence* (RFC 3856), *presence.winfo* (RFC 3857), *dialog;sla*. 
* [PRESENCE_MWI](/modules/devel/presence_mwi) - specific handling for notify-subscribe *message-summary* (message waiting indication) (RFC 3842)
* [PRESENCE_DIALOGINFO](/modules/devel/presence_dialoginfo) - handling of event  *dialog* (as defined in RFC 4235)
* [PRESENCE_CALLINFO](/modules/devel/presence_callinfo) - handling of shared call appearances (SCA) as defined by BroadWorks SIP Access Side Extensions Interface specifications.
* [PRESENCE_DFKS](/modules/devel/presence_DFKS) - handling of the *as-feature-event* event package (as defined by Broadsoft’s Device Feature Key Synchronization protocol)
* [PRESENCE_REGINFO](/modules/devel/presence_reginfo) - handling of the event *reg* (as defined in RFC 3680)
* [PRESENCE_XCAPDIFF](/modules/devel/presence_xcapdiff) - handling of the *xcap-diff* event

### The CONTROL modules

The [XCAP module](/modules/devel/xcap) is responsible for storing and applying/checking the XCAP rules - rules telling who should be allowed to see someone's presence info.

The [XCAP_CLIENT module](/modules/devel/xcap_client) implements an XCAP client for OpenSIPS - a way to retrieve XCAP rules from outside of OpenSIPS . It fetches XCAP elements, either documents or part of them, by sending HTTP GET requests.

- - -

## Configuration

The above architecture diagram shows a full/maximal configuration/setup, where (a) no restrictions are imposed, (b) with privacy permission rules handling enabled and (c) the use of a general XCAP server. 
There is the possibility to simplify and addapt the setup according to your custom needs. This can be done by configuring the modules so that some connections are removed or others appear.

### Basic Presence Server

The most simple configuration is a presence server without privacy rules, when anybody is allowed to see the presence status of anybody and there is no need for an xcap server. In this case the xcap part disappears from the scheme.. This mode of operation is configured by setting the flag **force_active** which is a module parameter for **presence_xml** module.  

You can see a [configuration file example for this case here](/tutorials-presence-simplepresconfig).

### Presence Server with integrated XCAP

If you want presence privacy rules enabled and you use a XCAP server, but one that communicates with the OpenSIPS Presence Server through a database table ( like OpenXCAP) than you can configure the presence server to work in an integrated xcap sever mode. For this you have to set the **integrated_xcap_server** parameter from **presence_xml** module. In this case the scheme simplifies by removing the xcap_client module and having the presence_xml module communicates only with the database.  

You will notice in the configuration file that the mi_xmlrpc module is also loaded. The MI interface needs to be activated, so that OpenXCAP can signal to OpenSIPS when a change occurs in the presence rules documents. In this way the changes be effective immediately.The MI command that is sent in this case is 'refreshWatchers'.  

See [configuration file example](/tutorials-presence-openxcappresconfig) here.

### Presence Server with external XCAP

If you decide to have a presence server with privacy authorization and not use an integrated xcap server, an improvement can be made in this case also. For a non integrated XCAp server, xcap_client module has to be used and the address of the server( or servers) must be set using the **presence_xml** modules parameter - **xcap_server**. The xcap_client module has the task to retrieve XCAP documents from XCAP servers and store them in database. Also the module can be asked to synchronize documents saved in database with those on the server. For this a mean of detecting an update is required. The general mean, that apply to every XCAP server is periodical query. There is however the possibility for the server to signal an update to the module. For this the module has to be able to send **refreshXcapDoc** MI command when an update occurs. If you are using a XCAP server able to send this command then you can configure the server to work in no periodical query mode. For this you have to unset the **xcap_client** module parameter **periodical_query** ( set it to 0). 

### Presence server with shared DB

Another configuration option refers to the way the presence module stores information. The default method is caching with periodical update in database for fast processing. However, if you have more presence servers serving the same domain and sharing the same database, this mode will not work. In this case you need to ensure that the servers will be able to communicate through the database. This can be achieved by setting the **presence** module parameter **fallback2db**. 

- - -

## Database

Presence Server requires 4 tables in database:

* active_watchers : stores the active Subscribe dialogs
* presentity: stores the published information
* watchers: stores the subscription status of the watchers
* xcap: stores the XCAP documents

The first three tables are only used internally by the server. The **xcap** table is either used as a communication interface between the XCAP server (if the server is working in 'integrated_xcap' mode) or as a communication means between the **xcap_client** module which populates the table and **presence_xml** module which uses that information.
