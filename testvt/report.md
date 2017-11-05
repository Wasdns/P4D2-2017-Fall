# Different behaviours when running vt

## BehaviourA: Droping

1.change the program of [p4app.json#L2](testvt/p4app.json#L2) to `testvt1.p4`.

2.start mininet to simulate the p4 switch.

3.configure the ARP caches:

```
mininet> xterm h1 h4

h1> ./setARP.sh
h4> ./setARP.sh
```

4.generate a iperf UDP traffic from h1 to h4:

```
h4> iperf -s -u -i 1
h1> iperf -c 10.0.0.4 -u -b 1G -t 60 -i 1
```

5.start wireshark to capture packets at `s1-eth4`, no packets have been forwarded by the switch.

## BehaviourB: Normal forwarding

1.change the program of [p4app.json#L2](testvt/p4app.json#L2) to `testvt2.p4`.

2.start mininet to simulate the p4 switch.

3.configure the ARP caches:

```
mininet> xterm h1 h4

h1> ./setARP.sh
h4> ./setARP.sh
```

4.generate a iperf UDP traffic from h1 to h4:

```
h4> iperf -s -u -i 1
h1> iperf -c 10.0.0.4 -u -b 1G -t 60 -i 1
```

5.start wireshark to capture packets at `s1-eth4`, you could see a lot of packets from h1 to h4.

## Analyze

The difference between `testvt1.p4` and testvt2.p4 is the action `myTunnel_ingress`.

In this experiment scenario, the packets which has dstIP `10.0.0.4` would be applied with the action `myTunnel_ingress`:

`testvt1.p4`:

```
    action myTunnel_ingress(bit<16> dst_id, egressSpec_t port) {
        hdr.myTunnel.setValid();
        hdr.myTunnel.dst_id = dst_id;
        hdr.myTunnel.proto_id = hdr.ethernet.etherType;
        hdr.ethernet.etherType = TYPE_MYTUNNEL;
        // unknown bug here
        standard_metadata.egress_spec = port;
    }
```

`testvt2.p4`:

```
    action myTunnel_ingress(bit<16> dst_id, egressSpec_t port) {
        // hdr.myTunnel.setValid();
        // hdr.myTunnel.dst_id = dst_id;
        // hdr.myTunnel.proto_id = hdr.ethernet.etherType;
        // hdr.ethernet.etherType = TYPE_MYTUNNEL;
        // unknown bug here
        standard_metadata.egress_spec = port;
    }
```

The common runtime command is in `s1-commands.txt`:

```
table_add ipv4_lpm myTunnel_ingress 10.0.0.4/32 => 4 4
```

So one thing could be concluded is that the packets are droped by the switch as soon as they have been encapsulated with the tunnel.
