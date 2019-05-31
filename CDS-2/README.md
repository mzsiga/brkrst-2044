# CDS-2 Instructions

## My Contact information:
> Email:    michael.zsiga@gmail.com\
> Twitter:  https://twitter.com/michael_zsiga \
> LinkedIn: https://www.linkedin.com/in/zigzag \
> Website:  https://zigbits.tech

This is Common Deployment Scenario (CDS) # 2 from the Cisco Live presentation BRKRST-2044 - Enterprise Multi-Homed Internet Edge Architectures. CDS-2 highlights the single router, dual ISP connections deployment example. Within this page are the steps to properly configure BGP Active / Standby (Section 1) and BGP Active / Active (Section 2) routing connectivity to the Internet (INET).

NOTE: For all the Common Deployment Scenarios (CDS) you can load the initial configurations for BB1, BB2, ISP-A, and ISP-B once. We are not making a lot of changes to these devices, if any.

# CDS-2 Reference topology
Here is the CDS-2 Reference topology

![CDS-2 Reference topology](CDS-2_Topology.jpg)

When we have dual IPS links we really need to know what we are using our links for and what we are honestly trying to solve.  We can solve for high-availability (HA) which is Active / Standby or we can solve for congestion and HA which is Active / Active.  In addition to this, we also need address how we handle equal and unequal bandwidths from an ISP perspective.

# CDS-2 Section 1: BGP Active / Standby

Here we are going to show how to properly create a BGP Active / Standby policy.  The first task we need to complete is to identify what our policies are going to be.  You can feel free to tweak these policies but for this section we are going to do the following:

Ingress Policy: We are going to use ISP A for all incoming traffic. If ISP-A fails, we will leverage ISP-B for full failover. Now what tool do you think we are going to use to help us achieve this policy?? We are going to use BGP AS-Path Prepend

Egress Policy: We are going to use ISP A for all outgoing traffic.  If ISP-A fails, we will leverage ISP-B for full failover. From a BGP Tool perspective, we are going to use Local-Preference to make this policy happen.

NOTE: Before moving forward, make sure you have the initial configurations loaded for FW2, R2, BB1, BB2, ISP-A, and ISP-B. Remember, we are not making a ton of changes on BB1, BB2, ISP-A, or ISP-B, so if you have already loaded them you should be good to go.

The initial configurations of FW2 and R2 include the IGP configuration between the two devices along with the basic BGP configuration on R2 to bring up its BGP Neighbors ISP-A and ISP-B.  No BGP policies have been applied, though there are some pre-configured route-maps and prefix-lists that we will be using.

Below is the starting BGP configuration on R2. You can check the running configuration of the devices to view the starting EIGRP configuration, we are not going to show that here to reduce the length of this section.

```
router bgp 64492
 no bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 2100:5100:51:2::1 remote-as 64501
 neighbor 2100:5100:51:2::1 description IPv6_eBGP_PEER_TO_ISP-A
 neighbor 2100:5200:52:2::1 remote-as 64502
 neighbor 2100:5200:52:2::1 description IPv6_eBGP_PEER_TO_ISP-B
 neighbor 51.51.2.1 remote-as 64501
 neighbor 51.51.2.1 description IPv4_eBGP_PEER_TO_ISP-A
 neighbor 52.52.2.1 remote-as 64502
 neighbor 52.52.2.1 description IPv4_eBGP_PEER_TO_ISP-B
 !
 address-family ipv4
  network 51.51.2.0 mask 255.255.255.252
  network 52.52.2.0 mask 255.255.255.252
  network 128.2.0.0
  neighbor 51.51.2.1 activate
  neighbor 51.51.2.1 soft-reconfiguration inbound
  neighbor 52.52.2.1 activate
  neighbor 52.52.2.1 soft-reconfiguration inbound
 exit-address-family
 !
 address-family ipv6
  network 2001:1282::/44
  network 2100:5100:51:2::/64
  network 2100:5200:52:2::/64
  neighbor 2100:5100:51:2::1 activate
  neighbor 2100:5100:51:2::1 soft-reconfiguration inbound
  neighbor 2100:5200:52:2::1 activate
  neighbor 2100:5200:52:2::1 soft-reconfiguration inbound
 exit-address-family
```

## Safety First

When we have dual links like we do in this CDS, and all of the rest of them, we need to make sure we are not re-advertising networks that we do not own out to our other ISP Peer.  If we were to do this, we could end up becoming an unintentional transport provider. This honestly happens a lot so do not skip this step.

The easiest way to make sure this doesn't happen to you is to use an as-path access-list to filter the outbound advertisements so that you are only advertising your Enterprises Prefixes.  Whats cool about an as-path access list is that you can use it for both IPv4 and IPv6.  To create the as-path access list we need to understand BGP regular expressions a little.  We also need to understand when and how our own ASN is added to the AS-PATH of our Enterprise routes.

Our local AS number is added after processing the AS-PATH list, so we can match on an empty AS PATH.

Now from a regular expression to match an empty path we need to match the beginning of the path, the end of the path, and nothing else. The (^) matches the beginning of the path. The ($) matches the end of the path.  When we put it together this is what we get.

```
ip as-path access-list 1 permit ^$
```

To apply this as-path access-list we just add it to the BGP neighbor configuration with the filter-list option, and now we are safe!

```
router bgp 64492
 address-family ipv4
  neighbor 51.51.2.1 filter-list 1 out
  neighbor 52.52.2.1 filter-list 1 out
 exit-address-family
 !
 address-family ipv6
  neighbor 2100:5100:51:2::1 filter-list 1 out
  neighbor 2100:5200:52:2::1 filter-list 1 out
 exit-address-family
```

Here is a screenshot of this being configured on R2:

![CDS-2 Section 1: Safety First](CDS-2_Section_1-01.png)

## Ingress policy

To enforce Ingress traffic over ISP-A, we need to prepend our AS-PATH out to ISP-B.  Remember BGP prefers the shortest AS-PATH. How many times should you prepend your ASN? Its usually good to prepend 4 or less times.  When you start to prepend more than that it can be obnoxious. If you have business reason to prepend more than do it. There are some designs out there in the wild that have a higher number of prepends.

To achieve this AS-PATH prepend we can create a simple route-map on R2 and apply it to our ISP-A BGP Neighbor.

```
route-map OUT-IspB permit 10
 set as-path prepend 64492 64492 64492 64492

router bgp 64492
 address-family ipv4
  neighbor 52.52.2.1 route-map OUT-IspB out
 exit-address-family
 !
 address-family ipv6
  neighbor 2100:5200:52:2::1 route-map OUT-IspB out
 exit-address-family
```

Here is a screenshot of the ingress policy being created and applied.

![CDS-2 Section 1: Ingress Policy](CDS-2_Section_1-02.png)

## Egress policy

For our egress policy, we do not need the full internet table, so we can start by filtering the routes coming from both ISP-A and ISP-B to just the defaults.  Then to enforce Egress traffic over ISP-A, we need to prefer the default routes being used from ISP-A over the ISP-B default routes. To do this we use local-preference.  Here is how this looks:

```
ip prefix-list v4Default-Only description ALLOW_ONLY_v4DEFAULT_ROUTE
ip prefix-list v4Default-Only seq 5 permit 0.0.0.0/0

ipv6 prefix-list v6Default-Only description ALLOW_ONLY_v6DEFAULT_ROUTE
ipv6 prefix-list v6Default-Only seq 5 permit ::/0

route-map IN-IspA permit 10
 set local-preference 200

router bgp 64492
 address-family ipv4
  neighbor 51.51.2.1 prefix-list v4Default-Only in
  neighbor 52.52.2.1 prefix-list v4Default-Only in
  neighbor 51.51.2.1 route-map IN-IspA in
 exit-address-family
 !
 address-family ipv6
  neighbor 2100:5100:51:2::1 prefix-list v6Default-Only in
  neighbor 2100:5200:52:2::1 prefix-list v6Default-Only in
  neighbor 2100:5100:51:2::1 route-map IN-IspA in
 exit-address-family
```

Here is a screenshot of the egress policy being created and applied.

![CDS-2 Section 1: Egress Policy](CDS-2_Section_1-03.png)

## Verification

To verify our ingress policy we need to check a remote server such as a BGP looking glass or route server on the internet. In this lab environment this role is performed by BB1 and/or BB2.

On BB1 we will check to see how the internet see's CDS-2's prefixes 128.2.0.0/16 and 2001:1282::/44 and make sure its being preferred through ISP-A's ASN 64501.

```
show bgp ipv4 unicast 128.2.0.0/16
show bgp ipv6 unicast 2001:1282::/44
```

Here is a screenshot of the Ingress policy being verified.

![CDS-2 Section 1: Ingress Policy Verified](CDS-2_Section_1-04.png)

![CDS-2 Section 1: Ingress Policy Verified with Traceroute](CDS-2_Section_1-05.png)

As you can see in the screenshots above, ISP-A is being selected to CDS-2's networks as we have configured.  

Now lets verify our egress policy.  To do this we just check the local BGP table on R2 with the following commands.

```
show bgp ipv4 unicast
show bgp ipv6 unicast
```

Here is a screenshot to show the output of these commands on R2.

![CDS-2 Section 1: Egress Policy Verified](CDS-2_Section_1-06.png)


# CDS-2 Section 2: BGP Active / Active

Here we are going to show how to properly create a BGP Active / Active policy.

Ingress Policy: Our goal for our ingress policy is to distribute the load evenly across both links.  To do this we are going to use more specific advertisements. For our example we have a /16 for IPv4 and a /44 for IPv6.  We are going to break these up into two smaller ranges. For IPv4 it will be two /17s. For IPv6 it will be two /45s.  Here are the ranges once divided up.

IPv4 Summary Range: 128.2.0.0/16\
IPv4 First Range:   128.2.0.0/17\
IPv4 Second Range:  128.2.128.0/17

IPv6 Summary Range: 2001:1282::/44\
IPv6 First Range:   2001:1282::/45\
IPv6 Second Range:  2001:1282:8::/45

Now that we have the split ranges, we will advertise the first range (128.2.0.0/17 and 2001:1282::/45) out of ISP-A and then advertise the second range (128.2.128.0/17 and 2001:1282:8::/45) out of ISP-B to distribute our ingress load evenly. We also will want to make sure we still continue to advertise the summary ranges out of both providers incase of a provider failure.

If you cannot split your addresses up, your options are limited. You can anycast advertise your network out to both providers but keep in mind this has unpredictable results. Finally, you can tweak with the AS-PATH configuration for your advertisement but then you are really doing an Active / standby deployment and not an Active / Active deployment.

Egress Policy: Our goal for our egress policy is the same goal we had for our ingress policy, we want to distribute the load evenly across both links. We are going to use local-preference and more specifics to achieve this policy for us.

From ISP-A, we are going to filter every other IPv4 /4 prefixes and install them.  For IPv6 we are going to install select prefixes only. We will also accept the default routes from ISP-A.

From ISP-B, we are going to accept the default routes and increase their local-preference.

NOTE: Before moving forward, make sure you have the initial configurations loaded for FW2, R2, BB1, BB2, ISP-A, and ISP-B. Remember, we are not making a ton of changes on BB1, BB2, ISP-A, or ISP-B, so if you have already loaded them you should be good to go.

## Ingress Policy

We should have the initial configurations loaded on R2 and FW2. On R2, we already have our basic BGP configuration running without any policies being applied.

For our split ranges to work, we will need to add BGP network statements in the respective address families. For these network statements to work, there must be a route in the RIB for it. In some cases you may need to add a null0 static tragic route. Instead of doing that, we are going to add EIGRP summaries on FW2 that will be redistributed into BGP on R2.  We also will be filtering what prefixes we are sending to which ISP.

Here is what we are adding on R2 for networks in BGP:

```
ip prefix-list IPv4_OUT_ISP-A description IPv4_OUT_ISP-A
ip prefix-list IPv4_OUT_ISP-A seq 5 permit 128.2.0.0/16
ip prefix-list IPv4_OUT_ISP-A seq 10 permit 128.2.0.0/17

ip prefix-list IPv4_OUT_ISP-B description IPv4_OUT_ISP-B
ip prefix-list IPv4_OUT_ISP-B seq 5 permit 128.2.0.0/16
ip prefix-list IPv4_OUT_ISP-B seq 10 permit 128.2.128.0/17

ipv6 prefix-list IPv6_OUT_ISP-A description IPv6_OUT_ISP-A
ipv6 prefix-list IPv6_OUT_ISP-A seq 5 permit 2001:1282::/44
ipv6 prefix-list IPv6_OUT_ISP-A seq 10 permit 2001:1282::/45

ipv6 prefix-list IPv6_OUT_ISP-B description IPv6_OUT_ISP-B
ipv6 prefix-list IPv6_OUT_ISP-B seq 5 permit 2001:1282::/44
ipv6 prefix-list IPv6_OUT_ISP-B seq 10 permit 2001:1282:8::/45

router bgp 64492
 address-family ipv4
  neighbor 51.51.2.1 prefix-list IPv4_OUT_ISP-A out
  neighbor 52.52.2.1 prefix-list IPv4_OUT_ISP-B out
  network 128.2.0.0 mask 255.255.128.0
  network 128.2.128.0 mask 255.255.128.0
 exit
 address-family ipv6
  neighbor 2100:5100:51:2::1 prefix-list IPv6_OUT_ISP-A out
  neighbor 2100:5200:52:2::1 prefix-list IPv6_OUT_ISP-B out
  network 2001:1282::/45
  network 2001:1282:8::/45
 exit
```

![CDS-2 Section 2: Ingress Policy Implementation on R2](CDS-2_Section_2-01.png)

Here is what we are adding on FW2 for our summary routes in EIGRP.


```
router eigrp DUAL-STACK
 !
 address-family ipv4 unicast autonomous-system 10
  !
  af-interface GigabitEthernet0/1
   summary-address 128.2.0.0 255.255.128.0
   summary-address 128.2.128.0 255.255.128.0
   no passive-interface
  exit-af-interface
 exit-address-family
 !
 address-family ipv6 unicast autonomous-system 10
  !
  af-interface GigabitEthernet0/1
   summary-address 2001:1282::/45
   summary-address 2001:1282:8::/45
   no passive-interface
  exit-af-interface
  !
 exit-address-family
```

![CDS-2 Section 2: Ingress Policy Implementation on FW2](CDS-2_Section_2-02.png)


## Egress Policy

For our IPv4 egress distribution to work we will need to receive the full internet BGP table from ISP-A on R2. From ISP-B we can filter down to the IPv4 default route.  On ISP-A we will create a prefix-list matching every other /4 and the default route, then inject these prefixes into BGP.  On ISP-B we will use a route-map to match the IPv4 default route, via a prefix-list, and then increase the local preference to 200.

```
ip prefix-list IPv4_IN_ISP-A description EVERY_OTHER_SLASH4
ip prefix-list IPv4_IN_ISP-A seq 5 permit 0.0.0.0/0
ip prefix-list IPv4_IN_ISP-A seq 10 permit 0.0.0.0/4 le 24
ip prefix-list IPv4_IN_ISP-A seq 15 permit 32.0.0.0/4 le 24
ip prefix-list IPv4_IN_ISP-A seq 20 permit 64.0.0.0/4 le 24
ip prefix-list IPv4_IN_ISP-A seq 25 permit 96.0.0.0/4 le 24
ip prefix-list IPv4_IN_ISP-A seq 30 permit 128.0.0.0/4 le 24
ip prefix-list IPv4_IN_ISP-A seq 35 permit 160.0.0.0/4 le 24
ip prefix-list IPv4_IN_ISP-A seq 40 permit 192.0.0.0/4 le 24

ip prefix-list IPv4_IN_ISP-B description DEFAULT_IN_B
ip prefix-list IPv4_IN_ISP-B seq 5 permit 0.0.0.0/0

route-map IPv4_IN_ISP-A permit 10
 description APPLY_TO_INBOUND_PREFIXES_FROM ISP-A
 match ip address prefix-list IPv4_IN_ISP-A

route-map IPv4_IN_ISP-B permit 10
 description APPLY_TO_INBOUND_PREFIXES_FROM ISP-B
 match ip address prefix-list IPv4_IN_ISP-B
 set local-preference 200

router bgp 64492
 address-family ipv4
  neighbor 51.51.2.1 route-map IPv4_IN_ISP-A in
  neighbor 52.52.2.1 route-map IPv4_IN_ISP-B in
 exit
```

Here is a screenshot showing the configuration of the IPv4 egress policy.

![CDS-2 Section 2: IPv4 Egress Policy Implementation on R2](CDS-2_Section_2-03.png)

Our IPv6 policy is similar to the IPv4 policy, the only difference is that we cannot easily breakup the IPv6 Internet ranges into neat /4s like we can with IPv4.  So we will have to create different IPv6 prefix-lists to match the first half and the second half of the IPv6 internet table.  Other than that, the policies are the same!  

Here goes nothing! :)


```
ipv6 prefix-list IPv6_IN_ISP-A description FIRST_HALF_V6
ipv6 prefix-list IPv6_IN_ISP-A seq 5 permit ::/0
ipv6 prefix-list IPv6_IN_ISP-A seq 10 permit 2001::/18 le 32

ipv6 prefix-list IPv6_IN_ISP-B description SECOND_HALF_V6
ipv6 prefix-list IPv6_IN_ISP-B seq 5 permit ::/0
ipv6 prefix-list IPv6_IN_ISP-B seq 10 permit 2001:4000::/20 le 32
ipv6 prefix-list IPv6_IN_ISP-B seq 15 permit 2001:8000::/22 le 32
ipv6 prefix-list IPv6_IN_ISP-B seq 20 permit 2002::/15 le 32
ipv6 prefix-list IPv6_IN_ISP-B seq 25 permit 2001:5000::/20 le 32
ipv6 prefix-list IPv6_IN_ISP-B seq 30 permit 2400::/6 le 32
ipv6 prefix-list IPv6_IN_ISP-B seq 35 permit 2800::/5 le 32

route-map IPv6_IN_ISP-A permit 10
 description APPLY_TO_INBOUND_PREFIXES_FROM ISP-A
 match ipv6 address prefix-list IPv6_IN_ISP-A

route-map IPv6_IN_ISP-B permit 10
 description APPLY_TO_INBOUND_PREFIXES_FROM ISP-B
 match ipv6 address prefix-list IPv6_IN_ISP-B
 set local-preference 200

router bgp 64492
 address-family ipv6
  neighbor 2100:5100:51:2::1 route-map IPv6_IN_ISP-A in
  neighbor 2100:5200:52:2::1 route-map IPv6_IN_ISP-B in
 exit
```

Here is a screenshot showing the configuration of the IPv6 egress policy.

![CDS-2 Section 2: IPv6 Egress Policy Implementation on R2](CDS-2_Section_2-04.png)

## Verification Time

Now that we have our policy implemented, we need to verify it actually works as expected.

For our ingress policy we need to validate the internet is receiving CDS-2's networks with the correct next-hop and AS-PATH information. To do this we will leverage BB1 as our Router Server / Looking Glass. Below are the commands we will utilize to check our networks:

```
show bgp ipv4 unicast regexp 64492$
show bgp ipv6 unicast regexp 64492$
```

Here is a screenshot showing our networks are in fact being learned like we wanted them too.

![CDS-2 Section 2: Ingress Policy Verification](CDS-2_Section_2-05.png)

Now we will use ping and traceroute to verify connectivity from the internet to our 'hosted' content.

```
ping 128.2.22.22
ping 128.2.128.22
ping 2001:1282:0:22::22
ping 2001:1282:8:128::22

traceroute 128.2.22.22
traceroute 128.2.128.22
traceroute 2001:1282:0:22::22
traceroute 2001:1282:8:128::22

```

Here are screenshots showing connectivity and that our ingress traffic is traversing the correct path we defined in our policy.

![CDS-2 Section 2: Ingress Policy Verification ping](CDS-2_Section_2-06.png)

![CDS-2 Section 2: Ingress Policy Verification traceroute](CDS-2_Section_2-07.png)


Now to verify our egress path, we need to verify the BGP table on R2.  Then we can check connectivty with ping and the correct path with traceroute.

```
show bgp ipv4 unicast

ping 16.16.16.16
ping 32.32.32.32

traceroute 16.16.16.16
traceroute 32.32.32.32

show bgp ipv6 unicast

ping 2001:1::1
ping 2001:5000::1

traceroute 2001:1::1
traceroute 2001:5000::1
```

And here are the screenshots for proof!! :)

![CDS-2 Section 2: Egress Policy Verification IPv4 BGP](CDS-2_Section_2-08.png)

![CDS-2 Section 2: Egress Policy Verification IPv4 ping](CDS-2_Section_2-09.png)

![CDS-2 Section 2: Egress Policy Verification IPv6 BGP](CDS-2_Section_2-10.png)

![CDS-2 Section 2: Egress Policy Verification IPv6 ping](CDS-2_Section_2-11.png)

# Thats a wrap for CDS-2

The final configurations files for CDS-2 Section 1 are located under /final-configs/CDS-2_Section_1

The final configurations files for CDS-2 Section 2 are located under /final-configs/CDS-2_Section_2
