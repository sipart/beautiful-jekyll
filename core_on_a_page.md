---
layout: page
title: Core On A Page
subtitle: Creating a Juniper MPLS core using RSVP as the LDP and ISIS as the IGP. The brief assumes no current need for Traffic Engineering
image: /img/mpls.jpg
bigimg: /img/bigimg/horizon-1.jpg
---
The below is the basics for a MPLS core to use in a lab. It only scratches the surface, but a good start if you want a MPLS core network to start swapping out the IGP or the LDP, adding CoS, VRFs, RSVP-TE, peering’s or inter-AS services and VPNs (option [A]( https://www.juniper.net/documentation/en_US/junose16.1/topics/concept/mbgp-inter-as-option-a-overview.html), [B]( https://www.juniper.net/documentation/en_US/junose16.1/topics/concept/mbgp-inter-as-option-b-overview.html) or [C]( https://www.juniper.net/documentation/en_US/junose16.1/topics/concept/mbgp-inter-as-option-c-overview.html))

![core_mpls](/img/core.png)

EVE-NG lab - sample configs in [this GitHUB repo](https://github.com/sipart/core_mpls). The brief below assumes your familar with initial Junos setup (root authentication, hostname etc.)

# Autonomous System
* Configure a suitable `autonomous-system` number under `routing-options`

# Configure MPLS
* Configure all the core link `interfaces` with a suitable IP `address` under `family inet`
* Configure all the core link `interfaces` with `family mpls` under the `unit`
* Set a suitable MTU size for the core trunk `interfaces`
* Configure all the core link interfaces under `protocols mpls`
* Set `no-propogate-ttl` at `protocols mpls` stanza level to disable normal time-to-live (TTL) decrementing (sets a MPLS header with a TTL value of 255)
* Add the `label-switched-path` (s) - used for path signalling - under `protocols mpls`. For pre signalling with set paths then add a primary (strict), secondary (strict) and tertiary (loose) path. The primary, secondary and tertiary paths are set under `protocols mpls` path with IP of trunk hops and destination. Set `no-cspf` on each `label-switched-path` to disable constrained-path LSP computation ([CSPF](https://netflask.net/jnpr-constrained-shortest-path-first/) is used if the core network is setup to use Traffic Engineering).
* LSPs are unidirectional so need to be defined on all the ingress routers for all primary and secondary paths.
* Optional: (`fast-reroute` provides a mechanism for automatically rerouting traffic on an LSP if a node or link in an LSP fails, thus reducing the loss of packets traveling over the LSP. Configure under the defined LSPs. Check pre-requisites for `fast-reroute` before configuring.

# Configure the Label Distribution Protocol using RSVP (or LDP)
* This brief assumes the use of RSVP in its simplest form – not being used with traffic engineering but note that RSVP has this capability for future growth or TE requirements
* RSVP uses unidirectional and simplex flows through the network to perform its function. The inbound router initiates an RSVP path message and sends it downstream to the outbound router. The path message contains information about the resources needed for the connection. Each router along the path begins to maintain information about a reservation
* RSVP is not a routing protocol – it needs an IGP (ISIS or OSPF) to determine paths
* Add core trunk interfaces to `protocols rsvp`

# Configure an IGP to determine best paths for LSPs using ISIS (or OSPF)
* ISIS is an IGP to determine best paths
* One flat Level 2 area can be used
* Set level 2 auth’ (md5) and `wide-metrics-only` ([generate metric values greater than 63 on a per ISIS level basis](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/wide-metrics-only-edit-protocols-isis.html)) at `protocols isis` stanza level
* Set interfaces `level 1 disable` as appropriate – set `level 2 passive` on management and loopback
* [Setup BFD](https://www.juniper.net/documentation/en_US/junos/topics/concept/configuring-bfd-isis.html) by setting `bfd-liveness-detection` on interfaces to core routers to configure bidirectional failure detection timers for ISIS
* Add `family iso` on core trunk interfaces and loopback
* Add a NET (Network Entity Title) `address` under `family iso` on the loopback `unit`
* For example, the NET address `49.0001.1921.6800.1001.00` consists of the following parts:

|AFI|Area ID|System Identifier|Selector|
|:---:|:---:|:---:|:---:|
|49|0001|1921.6800.1001|00|

# Configure the internal BGP
* iBGP is used to exchange label routes
* iBGP full mesh or route reflection – two route reflectors is ideal
* For full mesh set `type` and `local-address` and `family` and all `neighbor`(s) + optionally (recommended) an `authentication-key`
* For a route reflector set all the above BUT for RR clients with just the one (or more) route reflectors as neighbors and then on the reflectors set the `cluster` (which is the same as the local address) with all the other non-route reflector neighbors.

# Configure the Class of Service
* Just a summary - too much to cover here check out the Juniper [Deploying Basic QoS Day One book](https://www.juniper.net/uk/en/training/jnbooks/day-one/fundamentals-series/deploying-basic-qos/)
* Firewall filters
* Policers
* Classifiers
* Code point aliases
* Drop profiles
* Forwarding classes
* Traffic control profiles
* Rewrite rules
* Interfaces
* Scheduler maps
* Schedulers
* Note: for M and T series routers with IQ2 cards. If more than 128 units on an interface ensure that the scheduler mapping is configured to use more than 128 schedulers. To check:
`request pfe execute pic-slot <#> target fpc<#> command "show tmdrv scheduler-partition"`

# Other settings
* Radius, [banner](https://www.juniper.net/documentation/en_US/junos/topics/task/configuration/authentication-router-login-message-qfx-series.html), [users](https://www.juniper.net/documentation/en_US/junos/topics/concept/authentication-user-accounts-overview.html), ssh, [syslog](https://www.juniper.net/documentation/en_US/junos/topics/example/syslog-single-chassis-system-configuring.html), ntp, [RPM](https://www.juniper.net/documentation/en_US/junos/topics/concept/security-rpm-overview.html)
* Protect RE firewall filter on the loopback (made of associated prefix lists, policers, addresses, protocols and ports). [Securing the Routing Engine PDF](http://www.hiphop-resistance.com/juniperdayone/Securing_RouteEngine2.pdf)
* Backup-router settings (for dual RE) - [great post here](https://jncie.eu/how-to-deploy-vmx-with-multiple-res-and-multiple-fpcs-in-eve-ng-kvm/) on getting dual routing engines in EVE-NG courtesy of [Christian Scholz](https://twitter.com/chsjuniper)
* It is recommended to filter out unnecessary log messages that can cause log bloat or unnecessary load on syslog or monitoring systems: [Juniper KB Article](https://kb.juniper.net/InfoCenter/index?page=content&id=KB9382)
* Example below that was used on devices that performed RPM was used to just filter RPM test completion messages at very short intervals filling up logs very quickly: To set use:
`set system syslog file messages match "!(.*DAEMON-6-PING_TEST_COMPLETED:.*)"`

# VRF note
The difference between `route-distinguisher` and route target (`vrf-target`)
* The `route-distinguisher` and `vrf-target` values perform two completely separate functions, and although most press publications/websites show the values as the same (which they can be) it is confusing to someone learning MPLS for the first time as they assume they do the same thing. The `route-distinguisher` makes a route unique across the MPLS network. The `vrf-target` defines which prefixes get imported and exported on the PE routers VRFs. Having different route distinguishers for each VRF on each PE means that you can quickly id which PE the prefix originated from

## Ideal Further Reading and Reference
* Firstly if you want a guide on doing something similar but with Cisco, LDP, OSPF and a bit of Susan Sarandon (yes really!) then check this post over at [networkfuntimes](http://www.networkfuntimes.com/configure-a-basic-service-provider-mpls-vpn-free-gns3-topology-to-download/)
* DAY ONE - [MPLS UP AND RUNNING ON JUNOS](https://www.juniper.net/documentation/en_US/day-one-books/MPLS_UR.pdf)
* [Deploying Basic QoS Day One book](https://www.juniper.net/uk/en/training/jnbooks/day-one/fundamentals-series/deploying-basic-qos/)
* [Hardening Junos Devices Day One Book and Handy Checklist](https://www.juniper.net/uk/en/training/jnbooks/day-one/fundamentals-series/hardening-junos-devices-checklist/)
* [Good overview on Route Target and Distinguisher on networkfuntimes](http://www.networkfuntimes.com/route-distinguishers-vs-route-targets-what-are-they-why-do-we-need-them-both/)
* [ibgp family route-target](https://www.juniper.net/documentation/en_US/junos/topics/usage-guidelines/vpns-configuring-bgp-route-target-filtering-in-vpns.html) overview
* [Juniper CSPF netflask Post](https://netflask.net/jnpr-constrained-shortest-path-first/)
* [ISSU Overview](https://www.juniper.net/documentation/en_US/junos/topics/concept/issu-oveview.html)
* How to deploy vMX with [multiple RE’s and multiple FPC’s](https://jncie.eu/how-to-deploy-vmx-with-multiple-res-and-multiple-fpcs-in-eve-ng-kvm/) in EVE-NG (KVM)
* Configuring Junos OS for the First Time on a Device with [Dual Routing Engines](https://www.juniper.net/documentation/en_US/junos/topics/task/configuration/routing-engine-dual-initial-configuration.html)
* Understanding [High Availability Features](https://www.juniper.net/documentation/en_US/junos/information-products/pathway-pages/config-guide-high-availability/high-availability.html) - i.e. Graceful Routing Engine Switchover (GRES) and NonStop Routing (NSR)
