# CDS-4 Instructions

My Contact information:
> Email:    michael.zsiga@gmail.com\
> Twitter:  https://twitter.com/zig_zsiga \
> LinkedIn: https://www.linkedin.com/in/zigzag \
> Website:  https://zigbits.tech

This is Common Deployment Scenario (CDS) # 4 from the Cisco Live presentation BRKRST-2044 - Enterprise Multi-Homed Internet Edge Architectures. CDS-4 highlights the multiple routers, multiple ISP connections, and multiple sites deployment example. Within this page are the steps to properly configure BGP Active / Active routing connectivity to the Internet (INET). We are going to handle IPv4 and IPv6 separately here, so they will be discussed in their own sections.

NOTE: For all the Common Deployment Scenarios (CDS) you can load the initial configurations for BB1, BB2, ISP-A, and ISP-B once. We are not making a lot of changes to these devices, if any.

# CDS-4 Reference topology
Here is the CDS-4 Reference topology

![CDS-3 Reference topology](CDS-4_Topology.jpg)

# CDS-4 Section 1: IPv4 Solution

For our IPv4 policy we have to keep in mind that we will be utilizing network address translation on FW4 and FW5, and the state is not shared between the two sites.  Now from a symmetry of routes perspective NAT is a solution we can work with. For our Ingress policy we need to split up our network into different slices so that each firewall can own its own slide.  For IPv4, our range was 128.4.0.0/16 and we are going to break this up into two /17 slices, 128.4.0.0/17 and 128.4.128.0/17.  The West data center will own 128.4.0.0/17 and the east data center will own 128.4.128.0/17.  For our Egress policy we are going to install the default routes from each provider and propagate them down into our Campus (R7 and R8) in each location.  Once again for this CDS, we do not have administrative control over the firewalls, FW4 and FW5, so we are going to have to find a way around this to propagate the default down to the Campus routers (R7 and R8).

Make sure you have the initial configurations loaded for R5, R6, FW4, FW5, R7, and R8. Our starting configurations for our edge routers (R5 and R6) already have BGP configured to the provider without any policies applied. The campus routers (R7 and R8) have EIGRP configured to simulate a large campus network.

## Egress Policy

For us to propagate any routes between our edge routers (R5 and R6) and our campus routers (R7 and R8), we need to form some sort of routing neighborship through the firewalls (FW4 and FW5).  We could use an IGP or BGP to accomplish this. Because we have better policy control with BGP we are going to leverage eBGP neighborships between R5 and R7, and R6 and R8. On the campus routers (R7 and R8) we will be leveraging provide ASNs (65535 and 65534) for our BGP configuration.  We will want to remove these private ASNs before the networks are advertised to our providers. If you happened to notice that FW4 and FW5 are layer 3 Firewalls, we will need to leverage multi-hop BGP configurations to form these adjacencies.

eBGP Multi-hop
```
router bgp 64494
 neighbor 128.4.127.254 remote-as 65535
 neighbor 128.4.127.254 description IPv4_eBGP_PEER_TO_R7
 neighbor 128.4.127.254 ebgp-multihop 255
 address-family ipv4
  neighbor 128.4.127.254 activate
  neighbor 128.4.127.254 prefix-list v4Default-Only out
```

Remove Private ASNs from AS-PATH\
Egress Policy: Only allow default\
Ingress Policy: Advertise our Local subnets
```
router bgp 64494
 address-family ipv4
  neighbor 51.51.5.1 remove-private-as all
  neighbor 51.51.5.1 prefix-list v4Default-Only in
  neighbor 51.51.5.1 prefix-list IPv4_LOCAL_ROUTES out
```

Having the BGP configuration isn't going to be enough because the Firewalls will be utilizing static tragic routes, thus we must also use some static tragic routes ourselves.

Putting all of this together for R5 and R6 we get the following:

```
R5:
router bgp 64494
 neighbor 128.4.127.254 remote-as 65535
 neighbor 128.4.127.254 description IPv4_eBGP_PEER_TO_R7
 neighbor 128.4.127.254 ebgp-multihop 255
 address-family ipv4
  neighbor 51.51.5.1 remove-private-as all
  neighbor 51.51.5.1 prefix-list v4Default-Only in
  neighbor 51.51.5.1 prefix-list IPv4_LOCAL_ROUTES out
  neighbor 128.4.127.254 activate
  neighbor 128.4.127.254 prefix-list v4Default-Only out
 exit-address-family

ip route 128.4.0.0 255.255.0.0 128.4.0.2
ip route 128.4.127.254 255.255.255.255 128.4.0.2

R6:
router bgp 64494
 neighbor 128.4.255.254 remote-as 65534
 neighbor 128.4.255.254 description IPv4_eBGP_PEER_TO_R8
 neighbor 128.4.255.254 ebgp-multihop 255
 !
 address-family ipv4
  neighbor 52.52.6.1 remove-private-as all
  neighbor 52.52.6.1 prefix-list v4Default-Only in
  neighbor 52.52.6.1 prefix-list IPv4_LOCAL_ROUTES out
  neighbor 128.4.255.254 activate
  neighbor 128.4.255.254 prefix-list v4Default-Only out
 exit-address-family

ip route 128.4.0.0 255.255.0.0 128.4.128.2
ip route 128.4.255.254 255.255.255.255 128.4.128.2
```

Here are screenshots of eBGP Multi-hop being configured on R5 and R6:

![CDS-4 Section 1: eBGP Multihop Config R5](CDS-4_Section_1-01.png)

![CDS-4 Section 1: eBGP Multihop Config R6](CDS-4_Section_1-02.png)

Now that we have finished up our eBGP Multi-hop configuration on R5 and R6, we need to do the same eBGP Multi-hop configuration on R7 and R8.  We also cannot forget our lovely tragic routes! :)

```
R7:
router bgp 65535
 bgp router-id 128.4.127.254
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 128.4.0.1 remote-as 64494
 neighbor 128.4.0.1 description IPv4_eBGP_PEER_TO_R5
 neighbor 128.4.0.1 ebgp-multihop 255
 neighbor 128.4.0.1 update-source Loopback0
 !
 address-family ipv4
  network 128.4.0.0 mask 255.255.128.0
  neighbor 128.4.0.1 activate
 exit-address-family
 !

ip route 128.4.0.0 255.255.255.252 10.4.0.1

R8:
router bgp 65534
 bgp router-id 128.4.255.254
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 128.4.128.1 remote-as 64494
 neighbor 128.4.128.1 description IPv4_eBGP_PEER_TO_R6
 neighbor 128.4.128.1 ebgp-multihop 255
 neighbor 128.4.128.1 update-source Loopback0
 !
 address-family ipv4
  network 128.4.128.0 mask 255.255.128.0
  neighbor 128.4.128.1 activate
 exit-address-family
 !

ip route 128.4.128.0 255.255.255.252 10.4.128.1
```

Here are screenshots of eBGP Multi-hop being configured on R7 and R8:

![CDS-4 Section 1: eBGP Multihop Config R7](CDS-4_Section_1-03.png)

![CDS-4 Section 1: eBGP Multihop Config R8](CDS-4_Section_1-04.png)

Now that we have our full eBGP configuration done, we should expect to see our neighborships come up but they dont...because we need to have the security team add those static tragic routes on FW4 and FW5, so this is what they would need to add.

```
FW4:
ip route 0.0.0.0 0.0.0.0 128.4.0.1
ip route 10.4.0.0 255.255.128.0 10.4.0.2
ip route 128.4.127.254 255.255.255.255 10.4.0.2

FW5:
ip route 0.0.0.0 0.0.0.0 128.4.128.1
ip route 10.4.128.0 255.255.128.0 10.4.128.2
ip route 128.4.255.254 255.255.255.255 10.4.128.2

```

Here are screenshots showing these static routes being configured.

![CDS-4 Section 1: FW4 Static Tragic Routes](CDS-4_Section_1-05.png)

![CDS-4 Section 1: FW5 Static Tragic Routes](CDS-4_Section_1-06.png)

NOTE:You would also need to allow BGP through the Firewall ACL configuration, we are not running a full firewall configuration on FW4 and FW5 to save on resources.

Now we should do a quick verification that our eBGP neighborships are up and that we are just receiving the default route.  We do this with the following commands on R7 and R8:

```
show bgp ipv4 unicast summary
show bgp ipv4 unicast
```

Here are screenshots validating our eBGP configurations on R7 and R8.

![CDS-4 Section 1: R7 eBGP Validation](CDS-4_Section_1-07.png)

![CDS-4 Section 1: R8 eBGP Validation](CDS-4_Section_1-08.png)

Now all we need to do for our Egress policy is to get these default routes into the local IGP in the campus, so we will be redistributing BGP into EIGRP.  When we do this redistribution we need to make sure we increase the metric considerably as we do not want campus traffic to take a suboptimal route to the Internet.  Here are the configurations and verification commands:

```
ip prefix-list v4Default-Only description ALLOW_ONLY_v4DEFAULT_ROUTE
ip prefix-list v4Default-Only seq 5 permit 0.0.0.0/0

route-map IPv4_BGP->EIGRP permit 10
 match ip address prefix-list v4Default-Only

router eigrp 10
 redistribute bgp 65535 metric 10000 0 255 1 1500 route-map IPv4_BGP->EIGRP

show ip eigrp topology
```

Here is a screenshot showing the configuration being applied on R7, the same configuration is applied to R8 but the BGP ASN is 65534.

![CDS-4 Section 1: EIGRP Default Validation](CDS-4_Section_1-09.png)

## Ingress Policy

We can leverage the same eBGP Multihop configuration to instantiate our Ingress policy. We need to determine where we should originate our new /17 networks (128.4.0.0/17 and 128.4.128.0/17).  We could originate them on the edge routers or on the campus routers. Because the edge routers do not have visibility of the links between the Firewalls and the campus routers, there is a failure situation where we would end up dropping traffic.  This makes the campus routers the only viable place to originate the advertisement of the /17s.

To accomplish this advertisement we can add the corresponding network statements in BGP but then we actually need to get them in the RIB on R7 and R8. We are going to do this via static tragic routes to Null 0.

Here is what this looks like.

```
R7:
router bgp 65535
 address-family ipv4
  network 128.4.0.0 mask 255.255.128.0
 exit-address-family
 !

ip route 128.4.0.0 255.255.128.0 Null0

R8:
router bgp 65534
 address-family ipv4
  network 128.4.128.0 mask 255.255.128.0
 exit-address-family
 !

ip route 128.4.128.0 255.255.128.0 Null0

```
Here are screenshots showing these configurations on R7 and R8.

![CDS-4 Section 1: R7 Null Route](CDS-4_Section_1-10.png)

![CDS-4 Section 1: R8 Null Route](CDS-4_Section_1-11.png)


## Verification

Now its time to verify everything!! We are going to utilize ping and traceroute to assist us in our verification.

West Cost Server: 128.4.44.44 (10.4.44.44)\
East Cost Server: 128.4.128.44 (10.4.128.44)

Internet Web Server: 16.16.16.16

```
ping 16.16.16.16 so lo44
traceroute 16.16.16.16 so lo44

ping 128.4.44.44
ping 128.4.128.44
traceroute 128.4.44.44
traceroute 128.4.128.44

```

Here are the associated screenshots of our policy validation.

![CDS-4 Section 1: R7 Egress Validation](CDS-4_Section_1-12.png)

![CDS-4 Section 1: R8 Egress Validation](CDS-4_Section_1-13.png)

![CDS-4 Section 1: Ingress Validation](CDS-4_Section_1-14.png)


# CDS-4 Section 2: IPv6 Solution

The big difference between IPv4 above and IPv6 is that there is no NAT on the firewalls for IPv6. The question I ask is what problems does this bring to us from an Egress and Ingress perspective.

Egress we can really just do the same exact thing we did for IPv4.  Accept the corresponding default routes from our providers and bring that default route into the campus routers IGP.  The issue is Ingress and how traffic comes back to us from the internet and then how do the Edge routers make the correct decisions.  So lets get through the Engress policy configuration and validation real quick, then we will discuss the concern about the Ingress policy.

## Egress Policy

Here we are going to configure the following items on our edge routers (R5 and R6), similar to what we did in Section 1 above for IPv4:

- Filter provider routes learned to only the default route (::/0)
- Filter provider routes advertised to just our Local Subnets
- Remove Provide ASNs in AS-PATH
- Enable IPv6 eBGP Multi-hop from edge routers (R5 and R6) to the campus routers (R7 and R8).
- Add required static tragic routes

These bullets are shown in the below configurations for R5 and R6:

```
#R5:
router bgp 64494
 neighbor 2001:1284:4:127::254 remote-as 65535
 neighbor 2001:1284:4:127::254 description IPv6_eBGP_PEER_TO_R7
 neighbor 2001:1284:4:127::254 ebgp-multihop 255
 !
 address-family ipv6
  neighbor 2001:1284:4:127::254 activate
  neighbor 2001:1284:4:127::254 prefix-list v6Default-Only out
  neighbor 2100:5100:51:5::1 remove-private-as all
  neighbor 2100:5100:51:5::1 prefix-list v6Default-Only in
  neighbor 2100:5100:51:5::1 prefix-list IPv6_LOCAL_ROUTES out
 exit-address-family
!

ipv6 route 2001:1284:4:127::254/128 2001:1284:4:51::1

#R6:
router bgp 64494
 neighbor 2001:1284:4:255::254 remote-as 65534
 neighbor 2001:1284:4:255::254 description IPv6_eBGP_PEER_TO_R8
 neighbor 2001:1284:4:255::254 ebgp-multihop 255
 !
 address-family ipv6
  neighbor 2001:1284:4:255::254 activate
  neighbor 2001:1284:4:255::254 prefix-list v6Default-Only out
  neighbor 2100:5200:52:6::1 remove-private-as all
  neighbor 2100:5200:52:6::1 prefix-list v6Default-Only in
  neighbor 2100:5200:52:6::1 prefix-list IPv6_LOCAL_ROUTES out
 exit-address-family
!

ipv6 route 2001:1284:4:255::254/128 2001:1284:4:62::2
```

Here are the screenshots of these configurations being applied on R5 and R6 respectively.

![CDS-4 Section 2: R5 Egress Configurations](CDS-4_Section_2-01.png)

![CDS-4 Section 2: R6 Egress Configurations](CDS-4_Section_2-02.png)

Now we need to do the IPv6 eBGP Multi-hop configuration on the campus routers (R7 and R8). We will also redistribute the default route into EIGRP  Here is what it should look like.

```
#R7:
router bgp 65535
 neighbor 2001:1284:4:51::5 remote-as 64494
 neighbor 2001:1284:4:51::5 description IPv6_eBGP_PEER_TO_R5
 neighbor 2001:1284:4:51::5 ebgp-multihop 255
 neighbor 2001:1284:4:51::5 update-source Loopback0
 !
 address-family ipv6
  network 2001:1284::/44
  network 2001:1284:4:17::/64
  neighbor 2001:1284:4:51::5 activate
 exit-address-family
!

ipv6 router eigrp 100
 redistribute bgp 65535 metric 10000 0 255 1 1500 route-map IPv6_BGP->EIGRP

ipv6 route 2001:1284:4:51::/64 2001:1284:4:17::1
ipv6 route 2001:1284::/44 Null0

#R8:
router bgp 65534
 neighbor 2001:1284:4:62::6 remote-as 64494
 neighbor 2001:1284:4:62::6 description IPv6_eBGP_PEER_TO_R6
 neighbor 2001:1284:4:62::6 ebgp-multihop 255
 neighbor 2001:1284:4:62::6 update-source Loopback0
 !
 address-family ipv6
  network 2001:1284::/44
  network 2001:1284:4:28::/64
  neighbor 2001:1284:4:62::6 activate
 exit-address-family
!

ipv6 router eigrp 100
 redistribute bgp 65534 metric 10000 0 255 1 1500 route-map IPv6_BGP->EIGRP

ipv6 route 2001:1284:4:62::/64 2001:1284:4:28::2
ipv6 route 2001:1284::/44 Null0
```
And here are screenshots showing these configurations being applied on R7 and R8 respectively.

![CDS-4 Section 2: R7 Egress Configurations](CDS-4_Section_2-03.png)

![CDS-4 Section 2: R8 Egress Configurations](CDS-4_Section_2-04.png)


Now we have to do one more thing before our IPv6 eBGP Multi-hop neighborship will be up.  We need to add static tragic routes to the Firewalls because they are not participating in BGP but they will need to allow the BGP traffic to route through. No screenshots for these configurations as they are pretty straight forward.

```
#FW4:
ipv6 route 2001:1284::/44 2001:1284:4:17::7
ipv6 route ::/0 2001:1284:4:51::5

#FW5:
ipv6 route 2001:1284::/44 2001:1284:4:28::8
ipv6 route ::/0 2001:1284:4:62::6
```

Now lets just verify IPv6 BGP Neighborship, that we have the default route, and that we are properly redistributing it into EIGRP.  The commands we are going to use to accomplish this are listed below, and we are going run them on R7 and R8.

```
show bgp ipv6 unicast summary
show bgp ipv6 unicast | inc ::/0
show ipv6 eigrp topology | inc ::/0
```

Here are the screenshots of these validations:

![CDS-4 Section 2: R7 Egress Validation](CDS-4_Section_2-05.png)

![CDS-4 Section 2: R8 Egress Validation](CDS-4_Section_2-06.png)


## Ingress Policy

So Egress is done, YAY!  Now we need to figure out a way for our ingress traffic to pass through the correct Firewall it left out of, with out NAT. Remember that Firewalls are still state devices even if they are not running NAT. I know its a shock but its true my friends. :) So what does this mean, it means that traffic that leaves one firewall must return back through that same firewall because it owns the state of that connection. Because of this requirement, we need our edge routers to know about our campus networks so they can send the correct traffic back to the correct firewall. To do this, we are going to advertise our IGP routes from the campus into BGP, and then on up to the edge routers. Now when we do this redistribution, we want to ensure we include the IGP metric in the BGP MED field.  Then we also need to enable BGP always compared med on R5 and R6 for this to work. Lets go a head and make these changes and see all of this in action!!

```
#R7:
route-map IPv6_EIGRP->BGP permit 10
 set metric-type internal

router bgp 65535
 address-family ipv6
  redistribute eigrp 100 route-map IPv6_EIGRP->BGP include-connected

#R8:
route-map IPv6_EIGRP->BGP permit 10
 set metric-type internal

router bgp 65534
 address-family ipv6
  redistribute eigrp 100 route-map IPv6_EIGRP->BGP include-connected
```

Screenshots for proof!

![CDS-4 Section 2: R7 Ingress IGP Routes](CDS-4_Section_2-07.png)

![CDS-4 Section 2: R8 Ingress IGP Routes](CDS-4_Section_2-08.png)

Lets not forget to enable BGP always compare med on our edge routers (R5 and R6) now!

```
#R5 and R6:
router bgp 64494
 bgp always-compare-med
```

Screenshots or it didn't happen! :)

![CDS-4 Section 2: R5 Ingress compare med](CDS-4_Section_2-09.png)

![CDS-4 Section 2: R6 Ingress compare med](CDS-4_Section_2-10.png)


## Verification

And we are finally here!  Its end to end, egress and ingress, verification time!  We did already verify that we are receving our ISP default routes through into our campus router's EIGRP topology table.  So we know from an Egress perspective whats configured is working. Lets verify the IGP routes we injected into BGP are showing up on our Edge routers with the include MED (IGP metric) values.

On R5 and R6 we are going to issue the following command, and when we do we are looking for specific routes that are owned by devices in the other stack.  For example, when we look at R5, we are going to focus on the route 2001:1284:0:128::44/128 (R8's Loopback44). When we look at R6, we are going to focus on the route 2001:1284:0:44::44/128 (R7's Loopback44). It might be easier to use the specific network commands as well, so I will show them below with screenshots.

```
#R5 and R6:
show bgp ipv6 unicast

#R5:
show bgp ipv6 unicast 2001:1284:0:128::44/128

#R6:
show bgp ipv6 unicast 2001:1284:0:44::44/128

```
What we should see here is that we have two paths to reach the network in question and one of the paths is selected.  The selected path as a metric value of zero while the unselected path has much higher metric.  This is because we added the IGP metric in the BGP Med field when we redistributed from EIGRP into BGP on R7 and R8.

Here are those screenshots showing these details:

![CDS-4 Section 2: R5 Ingress med validation](CDS-4_Section_2-11.png)

![CDS-4 Section 2: R6 Ingress med validation](CDS-4_Section_2-12.png)

Now that we have shown the intended policies, lets test them with ping and traceroute.

R7 2001:1284:0:44::44
R8 2001:1284:0:128::44
Internet Server 1: 2000:16:16:16::16
Internet Server 2: 2001:5000::1

```
#R7 and R8:
ping 2000:16:16:16::16
ping 2001:5000::1
traceroute 2000:16:16:16::16
traceroute 2001:5000::1

#BB1:
ping 2001:1284:0:44::44
ping 2001:1284:0:128::44
traceroute 2001:1284:0:44::44
traceroute 2001:1284:0:128::44
```

Here are the screenshots!!

![CDS-4 Section 2: R7 Final Validation](CDS-4_Section_2-13.png)

![CDS-4 Section 2: R8 Final Validation](CDS-4_Section_2-14.png)

![CDS-4 Section 2: BB1 Final Validation](CDS-4_Section_2-15.png)

If you take a look at this last screenshot above, you will see that the second traceroute actually crosses the iBGP link between R5 and R6 to go down the correct stack. This is accomplished because of the IGP redistribution we did on R7 and R8 with the inclusion of the metric in the BGP MED field.

# Thats a wrap for CDS-3

The final configurations files for CDS-4 Section 1 are located under /final-configs/CDS-4_Section_1

The final configurations files for CDS-4 Section 2 are located under /final-configs/CDS-4_Section_2
