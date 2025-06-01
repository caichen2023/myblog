---
title: "OSPF Neighbor Adjacency Failures"
subtitle: ""
date: 2025-05-29T13:16:34-08:00
draft: false
author: "Chen Cai"
authorLink: ""
description: ""
license: ""
images: []
featuredImage: "https://i.imgur.com/8EPboBE.jpeg"
featuredImagePreview: "https://i.imgur.com/8EPboBE.jpeg"
tags: [OSPF]
categories: [Notes]
---

## OSPF Neighbor States
- Down: No Hello packets received.
- Attempt (Only in NBMA networks): waiting for Hello reply after actively sending.
- Init: Hello received, but own Router ID not seen by neighbor yet.
- 2-Way: Hello exchanged, bidirectional communication confirmed.
- ExStart/Exchange: Master/slave negotiation, start of database exchange.
- Loading/Full: LSAs exchanged; Full means neighbor is fully established.

---

The Down state means no Hello packets are received. This state does not appear in `show ip ospf neighbor` output.

## 1. Router ID conflict
```
R1#
*May 28 22:37:20.707: %OSPF-4-DUPRTRIDNBR: OSPF detected duplicate router-id 1.1.1.1 from 12.1.1.2 on interface GigabitEthernet0/0
```

```
R2#
*May 28 22:37:21.379: %OSPF-4-DUPRTRIDNBR: OSPF detected duplicate router-id 1.1.1.1 from 12.1.1.1 on interface GigabitEthernet0/0
```

## 2. Area ID mismatch
```
R1#debug ip ospf adj
May 28 22:33:36.823: OSPF-110 ADJ   Gi0/0: Rcv pkt from 12.1.1.2, area 0.0.0.0, mismatched area 0.0.0.1 in the header
R1#
May 28 22:33:46.383: OSPF-110 ADJ   Gi0/0: Rcv pkt from 12.1.1.2, area 0.0.0.0, mismatched area 0.0.0.1 in the header
R1#
*May 28 22:33:56.119: OSPF-110 ADJ   Gi0/0: Rcv pkt from 12.1.1.2, area 0.0.0.0, mismatched area 0.0.0.1 in the header
```

```
May 28 22:33:34.491: %OSPF-4-ERRRCV: Received invalid packet: mismatched area ID, from backbone area must be virtual-link but not found from 12.1.1.1, GigabitEthernet0/0
R2#
May 28 22:33:43.527: %OSPF-4-ERRRCV: Received invalid packet: mismatched area ID, from backbone area must be virtual-link but not found from 12.1.1.1, GigabitEthernet0/0
R2#
*May 28 22:33:52.651: %OSPF-4-ERRRCV: Received invalid packet: mismatched area ID, from backbone area must be virtual-link but not found from 12.1.1.1, GigabitEthernet0/0
```

## 3. OSPF authentication mismatch
- Different authentication types:

R1 is configured with OSPF authentication, whereas R2 is not.
```
R1#sh run int g0/0 | in ospf
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 chen
R1#sh ip os int g0/0
Message digest authentication enabled
Youngest key id is 1
R1#sh ip os nei
R1#
```

```
R1#debug ip os adj
OSPF adjacency debugging is on
R1#
May 28 22:54:20.947: OSPF-110 ADJ   Gi0/0: Send with youngest Key 1
R1#
May 28 22:54:29.199: OSPF-110 ADJ   Gi0/0: Rcv pkt from 12.1.1.2 : Mismatched Authentication type. Input packet specified type 0, we use type 2
```

```
R2#sh run int g0/0 | i ospf
R2#sh ip os ne
R2#
```

- Different authentication keys:

R1 and R2 have the same authentication enabled, but with different keys.
```
R2#sh run int g0/0 | in ospf
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 cai
R2#sh ip os int g0/0
Message digest authentication enabled
Youngest key id is 1
R2#sh ip os nei
R2#
```

```
May 28 23:02:22.319: OSPF-110 ADJ   Gi0/0: Rcv pkt from 12.1.1.2 : Mismatched Authentication Key - Message Digest Key 1
R1#
May 28 23:02:23.783: OSPF-110 ADJ   Gi0/0: Send with youngest Key 1
R1#
*May 28 23:02:31.815: OSPF-110 ADJ   Gi0/0: Rcv pkt from 12.1.1.2 : Mismatched Authentication Key - Message Digest Key 1
```

## 4. Subnet Mask Mismatch
R1: 12.1.1.1/24
```
R1#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.1/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
Transmit Delay is 1 sec, State DR, Priority 1
Designated Router (ID) 1.1.1.1, Interface address 12.1.1.1
No backup designated router on this network
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

R1#sh ip os nei
R1#
```

R2: 12.1.1.2/25
```
R2#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.2/25, Area 0, Attached via Network Statement
Process ID 110, Router ID 2.2.2.2, Network Type BROADCAST, Cost: 1
Transmit Delay is 1 sec, State DR, Priority 1
Designated Router (ID) 2.2.2.2, Interface address 12.1.1.2
No backup designated router on this network
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

R2#sh ip os nei
R2#
```

Generally, OSPF neighbors must be in the same IP subnet to form adjacency.
Even if devices can ping each other, different subnet masks (e.g., /24 vs /25) cause OSPF to treat the interfaces as not on the same network. Hello packets are ignored, and neighbors are not formed.

### Exception: point-to-point/multipoint
On `point-to-point` and `point-to-multipoint` network types, OSPF does not require the interfaces to have matching subnet masks.

R1: 12.1.1.1/24, Network Type POINTTOPOINT
```
R1#sh ip ospf interface g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.1/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 1.1.1.1, Network Type POINTTOPOINT, Cost: 1
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
Neighbor Count is 1, Adjacent neighbor count is 1
Adjacent with neighbor 2.2.2.2

R1#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           0   FULL/  -        00:00:33    12.1.1.2        GigabitEthernet0/0
R1#
```

R2: 12.1.1.2/25, Network Type POINTTOPOINT
```
R2#sh ip os interface g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.2/25, Area 0, Attached via Network Statement
Process ID 110, Router ID 2.2.2.2, Network Type POINTTOPOINT, Cost: 1
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
Neighbor Count is 1, Adjacent neighbor count is 1
Adjacent with neighbor 1.1.1.1

R2#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           0   FULL/  -        00:00:36    12.1.1.1        GigabitEthernet0/0
R2#
```

```
R1#sh ip os int g0/0
Internet Address 12.1.1.1/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 1.1.1.1, Network Type POINTTOMULTIPOINT, Cost: 1
Timer intervals configured, Hello 30, Dead 120, Wait 120, Retransmit 5

R1#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           0   FULL/  -        00:01:58    12.1.1.2        GigabitEthernet0/0
R1#
```

```
R2#sh ip os interface g0/0
Internet Address 12.1.1.2/25, Area 0, Attached via Network Statement
Process ID 110, Router ID 2.2.2.2, Network Type POINTTOMULTIPOINT, Cost: 1
Timer intervals configured, Hello 30, Dead 120, Wait 120, Retransmit 5

R2#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           0   FULL/  -        00:01:53    12.1.1.1        GigabitEthernet0/0
R2#
```

## 5. OSPF Network Type Mismatch
R1: Network Type POINTTOPOINT
```
R1#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.1/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 1.1.1.1, Network Type POINTTOPOINT, Cost: 1
Transmit Delay is 1 sec, State POINTTOPOINT
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

R1#sh ip os nei
R1#
```

R2: Network Type POINTTOMULTIPOINT
```
R2#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.2/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 2.2.2.2, Network Type POINTTOMULTIPOINT, Cost: 1
Transmit Delay is 1 sec, State POINTTOMULTIPOINT
Timer intervals configured, Hello 30, Dead 120, Wait 120, Retransmit 5

R2#sh ip os nei
R2#
```

### Exception: broadcast & point-to-point
By default, OSPF adjacency fails between `point-to-point` and `point-to-multipoint`, but can form between `point-to-point` and `broadcast` because they share the same default Hello and Dead intervals.

R1: Network Type POINTTOPOINT, Hello 10, Dead 40
```
R1#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.1/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 1.1.1.1, Network Type POINTTOPOINT, Cost: 1
Transmit Delay is 1 sec, State POINTTOPOINT
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

R1#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           0   FULL/  -        00:00:39    12.1.1.2        GigabitEtherne t0/0
R1#
```

R2: Network Type BROADCAST, Hello 10, Dead 40
```
R2#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.2/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 2.2.2.2, Network Type BROADCAST, Cost: 1
Transmit Delay is 1 sec, State DR, Priority 1
Designated Router (ID) 2.2.2.2, Interface address 12.1.1.2
Backup Designated router (ID) 1.1.1.1, Interface address 12.1.1.1
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

R2#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   FULL/BDR        00:00:36    12.1.1.1        GigabitEtherne        t0/0
R2#
```

### Exception: manually adjust the Hello/Dead intervals
R1:Network Type POINTTOPOINT; using default Hello/Dead intervals
```
R1#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.1/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 1.1.1.1, Network Type POINTTOPOINT, Cost: 1
Transmit Delay is 1 sec, State POINTTOPOINT
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

R1#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           0   FULL/  -        00:00:35    12.1.1.2        GigabitEthernet0/0
R1#
```

R2: Network type POINTTOMULTIPOINT; Hello interval changed to 10
```
R2#sh run int g0/0 | in ip ospf
ip ospf network point-to-multipoint
ip ospf hello-interval 10
R2#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.2/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 2.2.2.2, Network Type POINTTOMULTIPOINT, Cost: 1
Transmit Delay is 1 sec, State POINTTOMULTIPOINT
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

R2#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           0   FULL/  -        00:00:37    12.1.1.1        GigabitEthernet0/0
R2#
```

OSPF adjacencies can form between different network types **if Hello and Dead intervals match**, but this may lead to LSA flooding issues.

## 6. Hello/Dead interval mismatch
When `hello-interval` is modified, `dead-interval` automatically adjusts to 4 times the hello value.
But when `dead-interval` is modified, `hello-interval` does not change.

R1: Dead interval changed to 50
```
R1#sh run int g0/0 | in ip ospf
ip ospf dead-interval 50
R1#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.1/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
Transmit Delay is 1 sec, State DR, Priority 1
Designated Router (ID) 1.1.1.1, Interface address 12.1.1.1
No backup designated router on this network
Timer intervals configured, Hello 10, Dead 50, Wait 50, Retransmit 5

R1#sh ip os nei
R1#
```

```
R2#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.2/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 2.2.2.2, Network Type BROADCAST, Cost: 1
Transmit Delay is 1 sec, State DR, Priority 1
Designated Router (ID) 2.2.2.2, Interface address 12.1.1.2
No backup designated router on this network
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

R2#sh ip os nei
R2#
```

### Exception: hello-interval < 1 Seconds
R1: hello-interval 333 msec
```
R1#sh run int g0/0 | in ospf
ip ospf dead-interval minimal hello-multiplier 3
R1#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.1/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
Transmit Delay is 1 sec, State BDR, Priority 1
Designated Router (ID) 2.2.2.2, Interface address 12.1.1.2
Backup Designated router (ID) 1.1.1.1, Interface address 12.1.1.1
Timer intervals configured, Hello 333 msec, Dead 1, Wait 1, Retransmit 5

R1#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           1   FULL/DR         896 msec    12.1.1.2        GigabitEthernet0/0
R1#
```

R2: hello-interval 250 msec
```
R2#sh run int g0/0 | in ospf
ip ospf dead-interval minimal hello-multiplier 4
R2#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.2/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 2.2.2.2, Network Type BROADCAST, Cost: 1
Transmit Delay is 1 sec, State DR, Priority 1
Designated Router (ID) 2.2.2.2, Interface address 12.1.1.2
Backup Designated router (ID) 1.1.1.1, Interface address 12.1.1.1
Timer intervals configured, Hello 250 msec, Dead 1, Wait 1, Retransmit 5

R2#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   FULL/BDR        880 msec    12.1.1.1        GigabitEthernet0/0
R2#
```

When `hello-interval` is configured to less than 1 second, OSPF can form adjacencies even if the Hello/Dead timers are not exactly the same on both sides.

### Exception: hello-interval > dead-interval
When the `hello-interval` is set longer than the `dead-interval`, OSPF neighbors will form adjacency briefly, but then continuously flap.

```
R1#sh run int g0/0 | in ospf
ip ospf dead-interval 8
R1#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.1/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
Transmit Delay is 1 sec, State BDR, Priority 1
Designated Router (ID) 2.2.2.2, Interface address 12.1.1.2
Backup Designated router (ID) 1.1.1.1, Interface address 12.1.1.1
Flush timer for old DR LSA due in 00:02:46
Timer intervals configured, Hello 10, Dead 8, Wait 8, Retransmit 5

R1#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           1   EXSTART/DR      00:00:05    12.1.1.2        GigabitEthernet0/0
R1#
```

```
R2#sh run int g0/0 | in ospf
ip ospf dead-interval 8
R2#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.2/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 2.2.2.2, Network Type BROADCAST, Cost: 1
Transmit Delay is 1 sec, State DR, Priority 1
Designated Router (ID) 2.2.2.2, Interface address 12.1.1.2
No backup designated router on this network
Timer intervals configured, Hello 10, Dead 8, Wait 8, Retransmit 5

R2#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   FULL/BDR        00:00:07    12.1.1.1        GigabitEthernet0/0
R2#
```

```
May 28 22:19:45.775: %OSPF-5-ADJCHG: Process 110, Nbr 1.1.1.1 on GigabitEthernet0/0 from FULL to DOWN, Neighbor Down: Dead timer expired
R2#
May 28 22:19:51.947: %OSPF-5-ADJCHG: Process 110, Nbr 1.1.1.1 on GigabitEthernet0/0 from LOADING to FULL, Loading Done
R2#
May 28 22:20:04.491: %OSPF-5-ADJCHG: Process 110, Nbr 1.1.1.1 on GigabitEthernet0/0 from FULL to DOWN, Neighbor Down: Dead timer expired
R2#
*May 28 22:20:11.403: %OSPF-5-ADJCHG: Process 110, Nbr 1.1.1.1 on GigabitEthernet0/0 from LOADING to FULL, Loading Done
```

## 7. Stub Area Flag Mismatch
```
R2#debug ip ospf hello
OSPF hello debugging is on
R2#
May 28 23:29:43.095: OSPF-110 HELLO Gi0/0: Send hello to 224.0.0.5 area 1 from 12.1.1.2
R2#
May 28 23:29:45.075: OSPF-110 HELLO Gi0/0: Rcv hello from 1.1.1.1 area 1 12.1.1.1
*May 28 23:29:45.075: OSPF-110 HELLO Gi0/0: Hello from 12.1.1.1 with mismatched Stub/Transit area option bit
R2#
```

## 8. No `neighbor` command on non-broadcast network
On `non-broadcast` networks like NBMA and `Point-to-Multipoint NBMA`, OSPF does not send Hello packets automatically and requires manual neighbor configuration with the `neighbor` command to form adjacencies.

## 9. Passive interface
A passive interface does not send or process Hello packets, so no OSPF neighbor adjacency is formed on it.

## 10. Interface is down or cable disconnected

## 11. OSPF process not running

## 12. Hello packets blocked by ACL or firewall

## 13. MTU mismatch (Stuck in **ExStart/Exchange** state)
R1: MTU 2000
```
R1#sh run int g0/0 | in mtu
mtu 2000
R1#sh int g0/0 | in MTU
MTU 2000 bytes, BW 1000000 Kbit/sec, DLY 10 usec,
R1#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           1   EXCHANGE/BDR    00:00:39    12.1.1.2        GigabitEthernet0/0
R1#
```

R2: MTU 1500
```
R2#sh int g0/0 | in MTU
MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec,
R2#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   EXSTART/DR      00:00:39    12.1.1.1        GigabitEthernet0/0
R2#
```

```
R1#debug ip ospf adj
OSPF adjacency debugging is on
R1#
May 28 23:38:26.239: OSPF-110 ADJ   Gi0/0: Rcv DBD from 2.2.2.2 seq 0x1250 opt 0x52 flag 0x7 len 32  mtu 1500 state EXCHANGE
May 28 23:38:26.239: OSPF-110 ADJ   Gi0/0: Nbr 2.2.2.2 has smaller interface MTU
*May 28 23:38:26.239: OSPF-110 ADJ   Gi0/0: Send DBD to 2.2.2.2 seq 0x1250 opt 0x52 flag 0x2 len 52
```

### Exception: ip ospf mtu-ignore
The command **`ip ospf mtu-ignore`** allows OSPF to ignore MTU checks, enabling neighbor adjacency even when MTU values are unequal.  
It is sufficient to apply this command on the side with the **smaller MTU**.

```
R2#sh run int g0/0 | in ospf
 ip ospf mtu-ignore
R2#sh ip os nei
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   FULL/DR         00:00:39    12.1.1.1        GigabitEthernet0/0
R2#
```

## 14. MA Network: Priority = 0 (Stuck in **2-Way** state)
```
R1#sh run int g0/0 | in ospf
ip ospf priority 0
R1#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.1/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
Transmit Delay is 1 sec, State DROTHER, Priority 0
No designated router on this network
No backup designated router on this network
Flush timer for old DR LSA due in 00:01:59
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
Neighbor Count is 1, Adjacent neighbor count is 0
Suppress hello for 0 neighbor(s)
R1#sh ip os nei

Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           0   2WAY/DROTHER    00:00:36    12.1.1.2        GigabitEthernet0/0
R1#
```

```
R2#sh run int g0/0 | in ospf
ip ospf priority 0
R2#sh ip os int g0/0
GigabitEthernet0/0 is up, line protocol is up
Internet Address 12.1.1.2/24, Area 0, Attached via Network Statement
Process ID 110, Router ID 2.2.2.2, Network Type BROADCAST, Cost: 1
Transmit Delay is 1 sec, State DROTHER, Priority 0
No designated router on this network
No backup designated router on this network
Flush timer for old DR LSA due in 00:01:59
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
Neighbor Count is 1, Adjacent neighbor count is 0
Suppress hello for 0 neighbor(s)
R2#sh ip os nei

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           0   2WAY/DROTHER    00:00:36    12.1.1.1        GigabitEthernet0/0
R2#
```

