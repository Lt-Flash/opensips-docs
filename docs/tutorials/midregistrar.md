---
title: "Mid-Registrar"
author: "by Liviu Chircu"
description: "Configure a mid-registrar in front of an existing SIP PBX or registrar."
---

## Tutorial Overview

This tutorial will teach you how to configure an OpenSIPS mid-registrar in front of an existing SIP PBX or registrar. We will configure this new component to take over most of the SIP registrations and all call forking for the existing server. This will allow you to leverage existing infrastructure in order to continue to grow (as subscribers and as registration traffic) while keeping the current low-resources registrar server.

### How does the mid-registrar work?

The OpenSIPS mid-registrar is capable of throttling the amount of SIP registrations flowing through it, on the way to the main registrar. This is possible by configuring the mid-registrar to extend the lifetime of SIP Contacts on the way out.  The mid-registrar will then internally keep track of both ingress and egress expiration values, in order to automatically De-REGISTER the downstream, duration-extended contact when needed, ensuring a correct SIP signaling on both sides.


The module may also handle *SIP call forking* on behalf of the downstream SIP element.  Both serial/parallel forking can be achieved with the mid-registrar, potentially freeing up even more resources for the existing registrar server. This, in turn, will allow you to expand your user base with minimal software costs, extra hardware or configuration of the current components.

#### Scenario

![mid registrar platform before](/public/uploads/mid-registrar-platform-before.png)



This is our initial example VoIP platform. Its services are provided by a layer of multi-purpose servers (represented as the **main registrar** box). To name a few services of this current platform: conferencing, call pickup and internet messaging (IM). Currently, the platform servers seem to reach peak usage when handling around 50,000 mobile device subscribers.



Our task is to find a solution to scale the platform to 500,000 subscribers. We first do an assessment of the types of VoIP traffic being handled by the platform, only to reach an interesting conclusion: **80%+** of the traffic that is hitting our servers is made up of REGISTER requests! Is this correct? Well, it might be, as mobile devices have short registration lifetimes (60 seconds). With this conclusion, we have spotted a potential bottleneck: registration traffic handling!



After several phases of testing, we later find out that registration processing is rather expensive on the current system, which immediately turns it into a bottleneck. A cheap solution to this problem is to integrate an OpenSIPS mid-registrar into our platform, since it is capable of converting incoming high-rate registration traffic (mobile users to mid-registrar) into a low-rate variant (mid-registrar to main registrar). Its main jobs will include soaking up most of this registration traffic, as well as handling parallel forking.



By adding this inexpensive mid-registrar traffic conversion front-end for our servers, we will reduce their incoming rate of registrations. This, in turn, will free up a lot of platform resources, allowing it to redirect its resources to handling more subscribers. The resulting architecture will resemble the following:



![mid registrar platform after](/public/uploads/mid-registrar-platform-after.png)

#### OpenSIPS script

In the following, we will extend the default OpenSIPS script with mid-registrar capabilities in two steps. By doing this, we will actually go through two working modes of the module: *Contact Throttling* and *AOR Throttling*.

##### Throttling registrations

First, we will configure the mid-registrar to greatly increase the outgoing Contact expiration values. After doing so, it will internally keep track of the initial expirations as well as the extended ones, in order to decide if a REGISTER should be forwarded to the main registrar or not.

##### Module configuration

```opensips

loadmodule "mid_registrar.so"
modparam("mid_registrar", "mode", 1) /* 0 = mirror / 1 = ct / 2 = AoR */
modparam("mid_registrar", "outgoing_expires", 7200)
modparam("mid_registrar", "contact_id_param", "ctid")

```



We first load the *mid_registrar* module and set its parameters. Going through each parameter:

* *"mode"* - the working mode of the module. In this case, *"1"* means "Contact throttling", where the module will reduce the amount of registration traffic sent to the main registrar
* *"outgoing_expires"* - the default value for the expirations of the contacts that the mid-registrar relays to the main registrar. The *mid_registrar_save()* function also has a similar optional parameter, for more fine grained control.
* *"contact_id_param"* - only relevant in a "mirroring" or "contact throttling" mode. Specifies the name of the Contact URI parameter which is used by the module in order to match contacts and route SIP requests.

##### REGISTER handling

```opensips

    if (is_method("REGISTER")) {
        mid_registrar_save("location");
        switch ($retcode) {
        case 1:
            xlog("forwarding REGISTER to main registrar ($$ci=$ci)\n");
            $ru = "sip:10.0.0.3:5070";
            t_relay();
            break;
        case 2:
            xlog("absorbing REGISTER! ($$ci=$ci)\n");
            break;
        default:
            xlog("failed to save registration! ($$ci=$ci)\n");
        }

        exit;
    }

```



For all registrations, we make sure to call *mid_registrar_save()*. As stated in its [documentation](/modules/4-1/mid_registrar), it will not process the contacts immediately, but rather do the necessary changes to the REGISTER request before it is relayed to the main registrar, while also preparing to transparently handle its replies. The above code stays the same regardless of the module's working mode or insertion mode. We are now done with REGISTER request handling!

##### INVITE handling

Next, we proceed with contact lookup during calls or instant messages:



```opensips

    # initial requests from main registrar, need to look them up!
    if (is_method("INVITE|MESSAGE") && $si == "10.0.0.3" && $sp == 5070) {
        xlog("looking up $ru!\n");
        if (!mid_registrar_lookup("location")) {
            t_reply("404", "Not Found");
            exit;
        }

        t_relay();

        exit;
    }

```



Contact lookup is almost identical to the stock OpenSIPS script. The only big difference is that we are using *mid_registrar_lookup()* instead of *lookup()*. Note that *mid_registrar_lookup()* will behave according to the insertion and working modes. In our case, it will only populate the Request-URI variable (see [`$ru`](/manual/4-1/script-corevar)) since the module is in working *"mode = 1"*.

##### Call flow (throttling registrations by Contact)

The diagram below shows two devices of the same user (**A** and **B**) periodically registering to the platform. In *"Contact throttling"* mode, the main registrar will be notified about the presence of each SIP contact on the mid-registrar. When receiving a call, the mid-registrar will simply relay it to the user.



![mid registrar contact throttling](/public/uploads/mid-registrar-contact-throttling.png)



Notice how each of Alice's devices first registers with both registrars, after which subsequent registrations are absorbed at mid-registrar level.

##### Parallel forking support

The mid-registrar can additionally take over all call forking responsibilities. In order to enable this, we only need to switch it to *"mode = 2"* (AOR throttling).

##### Module configuration

```opensips

loadmodule "mid_registrar.so"
modparam("mid_registrar", "mode", 2) /* 0 = mirror / 1 = ct / 2 = AoR */
modparam("mid_registrar", "outgoing_expires", 7200)
modparam("mid_registrar", "at_escape_str", "%40")

```



Additional explanations:

* *"mode"* - the working mode of the module. In this case, *"2"* means "AOR throttling", where the module will both reduce the amount of registration traffic sent to the main registrar and also aggregate contacts into a single one when forwarding registrations to it. The latter will allow it to do parallel forking.
* *"at_escape_str"* - only relevant when in "AoR throttling" mode and with the usrloc use_domain setting enabled. This string represents the escape sequence for the "@" character which must be included, in one way or another, in mid-registrar's generated Contact URI usernames.



Although the rest of the script stays the same, the *mid_registrar_save()* and *mid_registrar_lookup()* functions will behave differently. For example, when saving  contacts, all Contact header fields of REGISTER requests will be aggregated into a single one. When looking up contacts, the entire branch set will be populated (see [`$branch`](/manual/4-1/script-corevar)), thus preparing to fork calls in parallel.

##### Call flow (throttling registrations by AOR)

![mid registrar aor throttling](/public/uploads/mid-registrar-aor-throttling.png)



In *AOR throttling* mode, the mid-registrar is aggregating both Alice's contacts (**A** and **B**) into a single contact (**alice@mid-registrar**) when forwarding registrations to the main registrar.

##### Call flow (parallel forking)

![mid registrar parallel forking](/public/uploads/mid-registrar-parallel-forking.png)



This call flow shows how the mid-registrar's *"AOR throttling"* mode allows it to take over all call forking responsibilities, freeing up even more resources for the main registrar.

#### SIP Authentication

User authentication is a mandatory requirement for any real-world VoIP service, so we assume it is enabled for our platform as well. However, doing so will require additional configuration in order for the mid-registrar to properly work. We will have to *whitelist* the mid-registrar's IP address on the main registrar, thus skipping SIP authentication for all requests it may generate.



The explanation for this is that in working modes *"1"* and *"2"*, the mid-registrar may *generate* De-REGISTER requests in order to avoid stale registrations on the main registrar. The problem is that by default, it will not be able to authenticate itself to the main registrar since it is not provisioned with any credentials, so the mid-registrar IP address should simply be whitelisted on the main registrar in order for these de-registrations to properly go through.

#### Downloads

The full script used in this tutorial is available [here](https://topaz.github.io/paste/#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).
