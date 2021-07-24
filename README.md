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
