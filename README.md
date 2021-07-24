# LDP-Tweaks

Let us continue with the same topology that we used in 'Basic-MPLS'


Let us determine what would happen if the LDP neighborship on the link PE1----P3 were to go down. 

We create an access-list on P3 router to block only LDP packets

```
P3#show ip access-lists 101
Extended IP access list 101
    10 deny tcp any eq 646 any
    20 deny tcp any any eq 646
    30 permit ip any any
```

We apply it on the interface facing the PE1 router. 
```
interface GigabitEthernet1
 ip address 13.1.1.3 255.255.255.0
 ip router isis test
 ip access-group 101 in
 negotiation auto
 no mop enabled
 no mop sysid
end
```
Now, there is no labelled path from PE1 to reach PE2 even though there is a label received from P4, we will not use it because it is not the IGP shortest path to reach 2.2.2.2
```
PE1#show mpls ldp bindings 
...
 lib entry: 2.2.2.2/32, rev 18
        local binding:  label: 20
        remote binding: lsr: 4.4.4.4:0, label: 17


PE1#show ip cef 2.2.2.2
2.2.2.2/32
  nexthop 13.1.1.3 GigabitEthernet1   >> uses the unlabelled path towards P3
```

This will cause disruptions in the service as P3 will receive a naked IP packet and will not have the route to reach the Dest IP. 

Notice that this situation arises if the IGP on the link is working properly and only the LDP neighborship goes down. If the whole link were to go down, even the IGP route would change and PE1 would have accepted the label from P4 (since it is now the only path to reach 2.2.2.2)

Nevertheless, we would still need to consider the case where only LDP goes down as it can very well happen in the network if there is some congestion in the network or CPU of the router. 

This is where LDP sync comes into play. We enable it using the command on all XE Router- 
```
PE1(config-router)#mpls ldp sync
```

On XR, it is slightly different 
```
RP/0/0/CPU0:PE2(config-isis)#interface gigabitEthernet 0/0/0/0
RP/0/0/CPU0:PE2(config-isis-if)#address-family ipv4 unicast 
RP/0/0/CPU0:PE2(config-isis-if-af)#mpls ldp sync 
RP/0/0/CPU0:PE2(config-isis-if-af)#exit
RP/0/0/CPU0:PE2(config-isis-if)#exit 
RP/0/0/CPU0:PE2(config-isis)#interface gigabitEthernet 0/0/0/1
RP/0/0/CPU0:PE2(config-isis-if)#address-family ipv4 unicast 
RP/0/0/CPU0:PE2(config-isis-if-af)#mpls ldp sync 
```

The below show command can be used to verify the status - 
```
PE1#show mpls ldp igp sync 
    GigabitEthernet1:
        LDP configured; LDP-IGP Synchronization enabled.
        Sync status: sync achieved; peer reachable.
        Sync delay time: 0 seconds (0 seconds left)
        IGP holddown time: infinite.
        Peer LDP Ident: 3.3.3.3:0
        IGP enabled: ISIS test
    GigabitEthernet2:
        LDP configured; LDP-IGP Synchronization enabled.
        Sync status: sync achieved; peer reachable.
        Sync delay time: 0 seconds (0 seconds left)
        IGP holddown time: infinite.
        Peer LDP Ident: 4.4.4.4:0
        IGP enabled: ISIS test
```




<font size="30">LDP session Protection </font>

What normally happens when a LDP peer goes down is that the labels associated with the peer are removed from the router. 

However, consider the scenario where the LDP peer goes down for a short period of time and comes back up within a second. We would need to relearn all the ldp label bindings from the peer again which would add to the convergence time. 

Instead, what we could do is to build an LDP targeted TCP session to the peer’s loopback IP. This targeted session uses a different or alternate path to reach the peer’s loopback and if successful, it means that the peer is still alive and we need not flush out the labels. 

Let us enable on all routers using the command - 
```
PE1(config)#mpls ldp session protection 

IOS XR - 

RP/0/0/CPU0:PE2(config)#mpls ldp 
RP/0/0/CPU0:PE2(config-ldp)#session protection 
RP/0/0/CPU0:PE2(config-ldp)#commit
Sat Jul 24 19:58:00.930 UTC
```
Let us now shut down the link between PE1 and P4
```
PE1(config)#int g2
PE1(config-if)#shutdown 
```
The targeted hello to 4.4.4.4 becomes active
```
PE1#show mpls ldp neighbor 
   ...    
Peer LDP Ident: 4.4.4.4:0; Local LDP Ident 1.1.1.1:0
        TCP connection: 4.4.4.4.16364 - 1.1.1.1.646
        State: Oper; Msgs sent/rcvd: 17/14; Downstream
        Up time: 00:00:48
        LDP discovery sources:
          Targeted Hello 1.1.1.1 -> 4.4.4.4, active, passive
        Addresses bound to peer LDP Ident:
          14.1.1.4        24.1.1.4        4.4.4.4   
```
We also see that the labels from the peer 4.4.4.4 are retained
```
PE1#show mpls ldp bindings 
  lib entry: 1.1.1.1/32, rev 2
        local binding:  label: imp-null
        remote binding: lsr: 3.3.3.3:0, label: 20
        remote binding: lsr: 4.4.4.4:0, label: 16
  lib entry: 2.2.2.2/32, rev 18
        local binding:  label: 20
        remote binding: lsr: 3.3.3.3:0, label: 19
        remote binding: lsr: 4.4.4.4:0, label: 17
```

These are some of the useful tweaks that we can do to LDP to make our network resillient during events of failure. 
