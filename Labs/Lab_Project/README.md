### Проектная работа

## Тема: Модернизация геораспределённой сети датацентров с использованием EVPN/VXLAN

### Цели:
- Модернизация текущей сетевой инфраструктуры с поддержкой возможностей использования VXLAN/EVPN
- Отказ от кривой legacy сети в ЦОДах


### Задачи:

- Разработка отказоустойчивой и масштабируемой топологии для существующих ЦОДов. 
- Проектирование DCI между ЦОД.
- Проектирование адресного пространства.
- Проектирование Underlay и Overlay сетей.
- Обеспечение маршрутизации трафика между VRF через маршрутизатор.

### Старая схема сети

![Схема1](old_map.png)

### Планируемая новая схема сети:
![Схема2](new_map.png)


#### <u>Планирование адресного пространства:</u>

|Network |Description|
|:-:|:-:|
|10.0.0.0/24| p2p/31 DCI|
|10.DCn.0.0/24| Mgmt/24 Network|
|10.DCn.1.0/24| p2p/31 Spine1|
|10.DCn.2.0/24| p2p/31 Spine2|
|10.DCn.3-250.0| Зарезервированно|
|10.DCn.251.0/24| p2p/31 iBGP MLAG|
|10.DCn.254.0/24| Lo/32 EVPN|
|10.DCn.255.0/24| p2p/31 MLAG|

DCn - номер ЦОДа (1 или 2)


#### <u>Таблица Loopback0/Router-ID и ASn, настроенных на устройствах:</u>

Pod-1:

|Device|IP Lo/Router-ID |ASn||
|:-:|:-:|:-:|:-:|
|Spine1|10.1.254.101|65100|
|Spine2|10.1.254.102|65100|
|Leaf1|10.1.254.1|65001|
|Leaf2|10.1.254.2|65001|
|Leaf3|10.1.254.3|65002|
|Leaf4|10.1.254.4|65002|
|Leaf5|10.1.254.5|65003|
|Leaf6|10.1.254.6|65004|
|Router|172.16.0.0|65099|

Pod-2:

|Device|IP Lo/Router-ID |ASn||
|:-:|:-:|:-:|:-:|
|Spine3|10.2.254.101|65200|
|Leaf21|10.2.254.1|65201|
|Leaf22|10.2.254.2|65202|
|Leaf23|10.2.254.3|65203|


#### <u>Таблица VTEP IP устройствах:</u>

|DC|Device|VTEP IP|
||:-:|:-:|:-:|
|1|Leaf1/2|10.1.254.201|
|1|Leaf3/4|10.1.254.202|
|1|Leaf5|10.1.254.5|
|1|Leaf6|10.1.254.6|
|2|Leaf21|10.2.254.1|
|2|Leaf22|10.2.254.2|
|1|Leaf23|10.2.254.3|

#### <u>Таблица VLAN/VRF/Network:</u>

|VLAN|Network|VRF|
|:-:|:-:|:-:|:-:|
|10|172.16.10.0/24|ADM|
|20|172.16.20.0/24|ADM|
|30|172.16.30.0/24|DEV|
|30|172.16.40.0/24|DEV|
|50|172.16.50.0/24|Stor|
|100|100.0.0.0/24|EXT|
|200|100.0.0.0/24|EXT|



#### <u>Таблица VLAN/VRF/VNI/RT :</u>

|VLAN|VRF|VNI|RT|
|:-:|:-:|:-:|:-:|:-:|
|10||100010|10:100010|
|20||100020|20:100020|
|30||100030|30:100030|
|40||100040|40:100040|
|50||100050|50:100050|
|100||100100|100:100100|
|200||100200|200:100200|
||ADM|111|111:111|
||DEV|222|222:222|
||Stor|333|333:333|
||EXT|444|444:444|


### Диагности BGP/EVPN RouteType-2/3/5 /MLAG/VRF Routing :


```
#### <u>Проверка на LEAF1:</u>
```
Leaf-1#show bgp summary
BGP summary information for VRF default
Router identifier 10.1.254.1, local AS number 65001
Neighbor                 AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.1.1.0              65100 Established   IPv4 Unicast            Negotiated             11         11
10.1.2.0              65100 Established   IPv4 Unicast            Negotiated             11         11
10.1.251.1            65001 Established   IPv4 Unicast            Negotiated             13         13
10.1.254.101          65100 Established   L2VPN EVPN              Negotiated             54         54
10.1.254.102          65100 Established   L2VPN EVPN              Negotiated             54         54
Leaf-1#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.254.1, local AS number 65001
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec   RD: 65203:30 mac-ip 504c.8961.0a0f
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:30 mac-ip 504c.8961.0a0f
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65203:30 mac-ip 504c.8961.0a0f 172.16.30.211
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:30 mac-ip 504c.8961.0a0f 172.16.30.211
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >     RD: 65001:10 imet 10.1.254.201
                                 -                     -       -       0       i
 * >     RD: 65001:20 imet 10.1.254.201
                                 -                     -       -       0       i
 * >     RD: 65001:30 imet 10.1.254.201
                                 -                     -       -       0       i
 * >     RD: 65001:40 imet 10.1.254.201
                                 -                     -       -       0       i
 * >     RD: 65001:50 imet 10.1.254.201
                                 -                     -       -       0       i
 * >     RD: 65001:100 imet 10.1.254.201
                                 -                     -       -       0       i
 * >Ec   RD: 65002:10 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:10 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65002:20 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:20 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65002:30 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:30 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65002:40 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:40 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65002:50 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:50 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65002:100 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:100 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65203:10 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:10 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65203:30 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:30 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65203:50 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:50 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65203:200 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:200 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65003:444 ip-prefix 0.0.0.0/0
                                 10.1.254.5            -       100     0       65100 65003 65099 ?
 *  ec   RD: 65003:444 ip-prefix 0.0.0.0/0
                                 10.1.254.5            -       100     0       65100 65003 65099 ?
 * >Ec   RD: 65004:444 ip-prefix 0.0.0.0/0
                                 10.1.254.6            -       100     0       65100 65004 65099 ?
 *  ec   RD: 65004:444 ip-prefix 0.0.0.0/0
                                 10.1.254.6            -       100     0       65100 65004 65099 ?
 * >Ec   RD: 65003:444 ip-prefix 8.8.8.8/32
                                 10.1.254.5            -       100     0       65100 65003 65099 i
 *  ec   RD: 65003:444 ip-prefix 8.8.8.8/32
                                 10.1.254.5            -       100     0       65100 65003 65099 i
 * >Ec   RD: 65004:444 ip-prefix 8.8.8.8/32
                                 10.1.254.6            -       100     0       65100 65004 65099 i
 *  ec   RD: 65004:444 ip-prefix 8.8.8.8/32
                                 10.1.254.6            -       100     0       65100 65004 65099 i
 * >     RD: 65001:444 ip-prefix 100.0.0.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65002:444 ip-prefix 100.0.0.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 *  ec   RD: 65002:444 ip-prefix 100.0.0.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 * >     RD: 65001:111 ip-prefix 172.16.10.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65002:111 ip-prefix 172.16.10.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 *  ec   RD: 65002:111 ip-prefix 172.16.10.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 * >Ec   RD: 65003:222 ip-prefix 172.16.10.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65002 ?
 *  ec   RD: 65003:222 ip-prefix 172.16.10.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:111 ip-prefix 172.16.10.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:111 ip-prefix 172.16.10.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:222 ip-prefix 172.16.10.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:222 ip-prefix 172.16.10.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:444 ip-prefix 172.16.10.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:444 ip-prefix 172.16.10.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65203:111 ip-prefix 172.16.10.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 *  ec   RD: 65203:111 ip-prefix 172.16.10.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 * >     RD: 65001:111 ip-prefix 172.16.20.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65002:111 ip-prefix 172.16.20.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 *  ec   RD: 65002:111 ip-prefix 172.16.20.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 * >     RD: 65001:222 ip-prefix 172.16.30.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65002:222 ip-prefix 172.16.30.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 *  ec   RD: 65002:222 ip-prefix 172.16.30.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 * >Ec   RD: 65003:111 ip-prefix 172.16.30.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65002 ?
 *  ec   RD: 65003:111 ip-prefix 172.16.30.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:111 ip-prefix 172.16.30.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:111 ip-prefix 172.16.30.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:222 ip-prefix 172.16.30.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:222 ip-prefix 172.16.30.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:444 ip-prefix 172.16.30.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:444 ip-prefix 172.16.30.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65203:222 ip-prefix 172.16.30.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 *  ec   RD: 65203:222 ip-prefix 172.16.30.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 * >Ec   RD: 65003:111 ip-prefix 172.16.30.211/32
                                 10.1.254.5            -       100     0       65100 65003 65099 65004 65202 65200 65203 i
 *  ec   RD: 65003:111 ip-prefix 172.16.30.211/32
                                 10.1.254.5            -       100     0       65100 65003 65099 65004 65202 65200 65203 i
 * >Ec   RD: 65003:222 ip-prefix 172.16.30.211/32
                                 10.1.254.5            -       100     0       65100 65003 65099 65004 65202 65200 65203 i
 *  ec   RD: 65003:222 ip-prefix 172.16.30.211/32
                                 10.1.254.5            -       100     0       65100 65003 65099 65004 65202 65200 65203 i
 * >Ec   RD: 65003:444 ip-prefix 172.16.30.211/32
                                 10.1.254.5            -       100     0       65100 65003 65099 65004 65202 65200 65203 i
 *  ec   RD: 65003:444 ip-prefix 172.16.30.211/32
                                 10.1.254.5            -       100     0       65100 65003 65099 65004 65202 65200 65203 i
 * >Ec   RD: 65004:111 ip-prefix 172.16.30.211/32
                                 10.1.254.6            -       100     0       65100 65004 65099 65004 65202 65200 65203 i
 *  ec   RD: 65004:111 ip-prefix 172.16.30.211/32
                                 10.1.254.6            -       100     0       65100 65004 65099 65004 65202 65200 65203 i
 * >Ec   RD: 65004:444 ip-prefix 172.16.30.211/32
                                 10.1.254.6            -       100     0       65100 65004 65099 65004 65202 65200 65203 i
 *  ec   RD: 65004:444 ip-prefix 172.16.30.211/32
                                 10.1.254.6            -       100     0       65100 65004 65099 65004 65202 65200 65203 i
 * >     RD: 65001:222 ip-prefix 172.16.40.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65002:222 ip-prefix 172.16.40.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 *  ec   RD: 65002:222 ip-prefix 172.16.40.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 * >Ec   RD: 65003:111 ip-prefix 172.16.40.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65002 ?
 *  ec   RD: 65003:111 ip-prefix 172.16.40.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:111 ip-prefix 172.16.40.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:111 ip-prefix 172.16.40.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:222 ip-prefix 172.16.40.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:222 ip-prefix 172.16.40.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:444 ip-prefix 172.16.40.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:444 ip-prefix 172.16.40.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >     RD: 65001:333 ip-prefix 172.16.50.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65002:333 ip-prefix 172.16.50.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 *  ec   RD: 65002:333 ip-prefix 172.16.50.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 * >Ec   RD: 65203:333 ip-prefix 172.16.50.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 *  ec   RD: 65203:333 ip-prefix 172.16.50.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 * >Ec   RD: 65003:111 ip-prefix 172.16.101.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 *  ec   RD: 65003:111 ip-prefix 172.16.101.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 * >Ec   RD: 65004:111 ip-prefix 172.16.101.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 *  ec   RD: 65004:111 ip-prefix 172.16.101.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 * >Ec   RD: 65003:222 ip-prefix 172.16.103.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 *  ec   RD: 65003:222 ip-prefix 172.16.103.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 * >Ec   RD: 65004:222 ip-prefix 172.16.103.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 *  ec   RD: 65004:222 ip-prefix 172.16.103.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 * >Ec   RD: 65003:444 ip-prefix 172.16.110.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 *  ec   RD: 65003:444 ip-prefix 172.16.110.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 * >Ec   RD: 65004:444 ip-prefix 172.16.110.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 *  ec   RD: 65004:444 ip-prefix 172.16.110.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 * >Ec   RD: 65003:111 ip-prefix 200.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65003:111 ip-prefix 200.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65003:222 ip-prefix 200.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65003:222 ip-prefix 200.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65004:111 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65004:111 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65004:222 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65004:222 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65004:444 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65004:444 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65203:444 ip-prefix 200.0.0.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 *  ec   RD: 65203:444 ip-prefix 200.0.0.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
Leaf-1#show ip route

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        10.1.1.0/31 is directly connected, Ethernet1
 C        10.1.2.0/31 is directly connected, Ethernet2
 C        10.1.251.0/31 is directly connected, Vlan4091
 C        10.1.254.1/32 is directly connected, Loopback0
 B I      10.1.254.2/32 [200/0] via 10.1.251.1, Vlan4091
 B E      10.1.254.3/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.1.254.4/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.1.254.5/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.1.254.6/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.1.254.101/32 [200/0] via 10.1.1.0, Ethernet1
 B E      10.1.254.102/32 [200/0] via 10.1.2.0, Ethernet2
 C        10.1.254.201/32 is directly connected, Loopback1
 B E      10.1.254.202/32 [200/0] via 10.1.1.0, Ethernet1
                                  via 10.1.2.0, Ethernet2
 C        10.1.255.0/31 is directly connected, Vlan4090
 B E      10.2.254.1/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.2.254.2/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.2.254.3/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.2.254.101/32 [200/0] via 10.1.1.0, Ethernet1
                                  via 10.1.2.0, Ethernet2

Leaf-1#
Leaf-1#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP               Tunnel Type(s)
------------------ --------------
10.1.254.5         unicast
10.1.254.6         unicast
10.1.254.202       unicast, flood
10.2.254.3         unicast, flood

Total number of remote VTEPS:  4
Leaf-1#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI          VLAN       Source       Interface            802.1Q Tag
------------ ---------- ------------ -------------------- ----------
100010       10         static       Port-Channel10       10
                                     Vxlan1               10
100020       20         static       Port-Channel10       20
                                     Vxlan1               20
100030       30         static       Port-Channel10       30
                                     Vxlan1               30
100040       40         static       Port-Channel10       40
                                     Vxlan1               40
100050       50         static       Port-Channel10       50
                                     Vxlan1               50
100100       100        static       Port-Channel10       100
                                     Vxlan1               100

VNI to dynamic VLAN Mapping for Vxlan1
VNI       VLAN       VRF        Source
--------- ---------- ---------- ------------
111       4085       ADM        evpn
222       4088       DEV        evpn
333       4086       STOR       evpn
444       4087       EXT        evpn

Leaf-1#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  30  504c.8961.0a0f  EVPN      Vx1  10.2.254.3       1       0:04:18 ago
4085  504b.d202.caff  EVPN      Vx1  10.2.254.3       1       12:36:15 ago
4085  5053.c96a.80ff  EVPN      Vx1  10.1.254.6       1       9:18:59 ago
4085  5057.ebb5.cd8b  EVPN      Vx1  10.1.254.202     1       12:37:34 ago
4085  50b6.a924.91d7  EVPN      Vx1  10.1.254.5       1       12:36:11 ago
4085  50d7.8378.a060  EVPN      Vx1  10.1.254.202     1       12:37:34 ago
4086  504b.d202.caff  EVPN      Vx1  10.2.254.3       1       12:36:15 ago
4086  5057.ebb5.cd8b  EVPN      Vx1  10.1.254.202     1       12:37:34 ago
4086  50d7.8378.a060  EVPN      Vx1  10.1.254.202     1       12:37:34 ago
4087  504b.d202.caff  EVPN      Vx1  10.2.254.3       1       12:36:15 ago
4087  5053.c96a.80ff  EVPN      Vx1  10.1.254.6       1       9:18:58 ago
4087  5057.ebb5.cd8b  EVPN      Vx1  10.1.254.202     1       12:37:34 ago
4087  50b6.a924.91d7  EVPN      Vx1  10.1.254.5       1       12:36:11 ago
4087  50d7.8378.a060  EVPN      Vx1  10.1.254.202     1       12:37:34 ago
4088  504b.d202.caff  EVPN      Vx1  10.2.254.3       1       12:36:15 ago
4088  5053.c96a.80ff  EVPN      Vx1  10.1.254.6       1       9:18:59 ago
4088  5057.ebb5.cd8b  EVPN      Vx1  10.1.254.202     1       12:37:34 ago
4088  50b6.a924.91d7  EVPN      Vx1  10.1.254.5       1       12:36:10 ago
4088  50d7.8378.a060  EVPN      Vx1  10.1.254.202     1       12:37:34 ago
Total Remote Mac Addresses for this criterion: 19
Leaf-1#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    5069.a447.888f    STATIC      Po100
  20    5069.a447.888f    STATIC      Po100
  30    504c.8961.0a0f    DYNAMIC     Vx1        1       0:04:18 ago
  30    5069.a447.888f    STATIC      Po100
  40    5069.a447.888f    STATIC      Po100
  50    5069.a447.888f    STATIC      Po100
 100    5069.a447.888f    STATIC      Po100
4085    504b.d202.caff    DYNAMIC     Vx1        1       12:36:15 ago
4085    5053.c96a.80ff    DYNAMIC     Vx1        1       9:18:59 ago
4085    5057.ebb5.cd8b    DYNAMIC     Vx1        1       12:37:34 ago
4085    5069.a447.888f    STATIC      Po100
4085    50b6.a924.91d7    DYNAMIC     Vx1        1       12:36:11 ago
4085    50d7.8378.a060    DYNAMIC     Vx1        1       12:37:34 ago
4086    504b.d202.caff    DYNAMIC     Vx1        1       12:36:15 ago
4086    5057.ebb5.cd8b    DYNAMIC     Vx1        1       12:37:34 ago
4086    5069.a447.888f    STATIC      Po100
4086    50d7.8378.a060    DYNAMIC     Vx1        1       12:37:34 ago
4087    504b.d202.caff    DYNAMIC     Vx1        1       12:36:15 ago
4087    5053.c96a.80ff    DYNAMIC     Vx1        1       9:18:58 ago
4087    5057.ebb5.cd8b    DYNAMIC     Vx1        1       12:37:34 ago
4087    5069.a447.888f    STATIC      Po100
4087    50b6.a924.91d7    DYNAMIC     Vx1        1       12:36:11 ago
4087    50d7.8378.a060    DYNAMIC     Vx1        1       12:37:34 ago
4088    504b.d202.caff    DYNAMIC     Vx1        1       12:36:15 ago
4088    5053.c96a.80ff    DYNAMIC     Vx1        1       9:18:59 ago
4088    5057.ebb5.cd8b    DYNAMIC     Vx1        1       12:37:34 ago
4088    5069.a447.888f    STATIC      Po100
4088    50b6.a924.91d7    DYNAMIC     Vx1        1       12:36:10 ago
4088    50d7.8378.a060    DYNAMIC     Vx1        1       12:37:34 ago
4090    5069.a447.888f    STATIC      Po100
4091    5069.a447.888f    STATIC      Po100
Total Mac Addresses for this criterion: 31

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf-1#
Leaf-1#show mlag
MLAG Configuration:
domain-id                          :                 100
local-interface                    :            Vlan4090
peer-address                       :          10.1.255.1
peer-link                          :     Port-Channel100
peer-config                        :          consistent

MLAG Status:
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:2f:9e:5a:b0:e7
dual-primary detection             :            Disabled
dual-primary interface errdisabled :               False

MLAG Ports:
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   0
Active-full                        :                   1

Leaf-1#
Leaf-1#show ip route vrf all

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        10.1.1.0/31 is directly connected, Ethernet1
 C        10.1.2.0/31 is directly connected, Ethernet2
 C        10.1.251.0/31 is directly connected, Vlan4091
 C        10.1.254.1/32 is directly connected, Loopback0
 B I      10.1.254.2/32 [200/0] via 10.1.251.1, Vlan4091
 B E      10.1.254.3/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.1.254.4/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.1.254.5/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.1.254.6/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.1.254.101/32 [200/0] via 10.1.1.0, Ethernet1
 B E      10.1.254.102/32 [200/0] via 10.1.2.0, Ethernet2
 C        10.1.254.201/32 is directly connected, Loopback1
 B E      10.1.254.202/32 [200/0] via 10.1.1.0, Ethernet1
                                  via 10.1.2.0, Ethernet2
 C        10.1.255.0/31 is directly connected, Vlan4090
 B E      10.2.254.1/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.2.254.2/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.2.254.3/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.2.254.101/32 [200/0] via 10.1.1.0, Ethernet1
                                  via 10.1.2.0, Ethernet2


VRF: ADM
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 S        172.16.10.0/24 is directly connected, Vlan10
 S        172.16.20.0/24 is directly connected, Vlan20
 B E      172.16.30.211/32 [200/0] via VTEP 10.1.254.5 VNI 111 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                                   via VTEP 10.1.254.6 VNI 111 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.30.0/24 [200/0] via VTEP 10.1.254.5 VNI 111 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                                 via VTEP 10.1.254.6 VNI 111 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.40.0/24 [200/0] via VTEP 10.1.254.5 VNI 111 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                                 via VTEP 10.1.254.6 VNI 111 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.101.0/31 [200/0] via VTEP 10.1.254.5 VNI 111 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
 B E      172.16.101.2/31 [200/0] via VTEP 10.1.254.6 VNI 111 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 C        192.1.10.0/24 is directly connected, Vlan10
 C        192.1.20.0/24 is directly connected, Vlan20
 B E      200.0.0.0/24 [200/0] via VTEP 10.1.254.5 VNI 111 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                               via VTEP 10.1.254.6 VNI 111 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1


VRF: DEV
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      172.16.10.0/24 [200/0] via VTEP 10.1.254.5 VNI 222 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                                 via VTEP 10.1.254.6 VNI 222 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.30.211/32 [200/0] via VTEP 10.2.254.3 VNI 222 router-mac 50:4b:d2:02:ca:ff local-interface Vxlan1
 S        172.16.30.0/24 is directly connected, Vlan30
 S        172.16.40.0/24 is directly connected, Vlan40
 B E      172.16.103.0/31 [200/0] via VTEP 10.1.254.5 VNI 222 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
 B E      172.16.103.2/31 [200/0] via VTEP 10.1.254.6 VNI 222 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 C        192.1.30.0/24 is directly connected, Vlan30
 C        192.1.40.0/24 is directly connected, Vlan40
 B E      200.0.0.0/24 [200/0] via VTEP 10.1.254.5 VNI 222 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                               via VTEP 10.1.254.6 VNI 222 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1


VRF: EXT
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via VTEP 10.1.254.5 VNI 444 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                            via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1

 B E      8.8.8.8/32 [200/0] via VTEP 10.1.254.5 VNI 444 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                             via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 S        100.0.0.0/24 is directly connected, Vlan100
 B E      172.16.10.0/24 [200/0] via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.30.211/32 [200/0] via VTEP 10.1.254.5 VNI 444 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                                   via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.30.0/24 [200/0] via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.40.0/24 [200/0] via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.110.0/31 [200/0] via VTEP 10.1.254.5 VNI 444 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
 B E      172.16.110.2/31 [200/0] via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 C        192.1.100.0/24 is directly connected, Vlan100
 B E      200.0.0.0/24 [200/0] via VTEP 10.2.254.3 VNI 444 router-mac 50:4b:d2:02:ca:ff local-interface Vxlan1


VRF: STOR
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 S        172.16.50.0/24 is directly connected, Vlan50
 C        192.1.50.0/24 is directly connected, Vlan50

Leaf-1#show ip arp vrf all

VRF: ADM
Address         Age (sec)  Hardware Addr   Interface
172.16.10.111     1:46:02  509c.edd9.92a1  Vlan10, not learned
172.16.20.111     1:53:10  509c.edd9.92a1  Vlan20, not learned

VRF: DEV
Address         Age (sec)  Hardware Addr   Interface
172.16.30.111     2:12:06  509c.edd9.92a1  Vlan30, not learned
172.16.30.211           -  504c.8961.0a0f  Vlan30, Vxlan1
172.16.40.111     1:43:34  509c.edd9.92a1  Vlan40, not learned

VRF: EXT
Address         Age (sec)  Hardware Addr   Interface
100.0.0.111       2:12:19  509c.edd9.92a1  Vlan100, not learned

VRF: STOR
Address         Age (sec)  Hardware Addr   Interface
172.16.50.111     1:57:47  509c.edd9.92a1  Vlan50, not learned

VRF: default
Address         Age (sec)  Hardware Addr   Interface
10.1.1.0          0:00:00  50ef.19d8.dff9  Ethernet1
10.1.2.0          0:00:00  50a5.3448.d4f7  Ethernet2
10.1.255.1        0:00:02  5069.a447.888f  Vlan4090, Port-Channel100
10.1.251.1        0:00:00  5069.a447.888f  Vlan4091, Port-Channel100

```
#### <u>LEAF2:</u>
```
Leaf-2#show bgp summary
BGP summary information for VRF default
Router identifier 10.1.254.2, local AS number 65001
Neighbor                 AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.1.1.2              65100 Established   IPv4 Unicast            Negotiated             11         11
10.1.2.2              65100 Established   IPv4 Unicast            Negotiated             11         11
10.1.251.0            65001 Established   IPv4 Unicast            Negotiated             13         13
10.1.254.101          65100 Established   L2VPN EVPN              Negotiated             51         51
10.1.254.102          65100 Established   L2VPN EVPN              Negotiated             51         51
Leaf-2#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.254.2, local AS number 65001
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec   RD: 65203:50 mac-ip 504c.8961.0a0f
                                 10.2.254.3            -       100     0       65100 65003 65201 65200 65203 i
 *  ec   RD: 65203:50 mac-ip 504c.8961.0a0f
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65203:50 mac-ip 504c.8961.0a0f 172.16.50.211
                                 10.2.254.3            -       100     0       65100 65003 65201 65200 65203 i
 *  ec   RD: 65203:50 mac-ip 504c.8961.0a0f 172.16.50.211
                                 10.2.254.3            -       100     0       65100 65003 65201 65200 65203 i
 * >Ec   RD: 65002:50 mac-ip 50f5.6711.ea60
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:50 mac-ip 50f5.6711.ea60
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65002:50 mac-ip 50f5.6711.ea60 172.16.50.122
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:50 mac-ip 50f5.6711.ea60 172.16.50.122
                                 10.1.254.202          -       100     0       65100 65002 i
 * >     RD: 65001:10 imet 10.1.254.201
                                 -                     -       -       0       i
 * >     RD: 65001:20 imet 10.1.254.201
                                 -                     -       -       0       i
 * >     RD: 65001:30 imet 10.1.254.201
                                 -                     -       -       0       i
 * >     RD: 65001:40 imet 10.1.254.201
                                 -                     -       -       0       i
 * >     RD: 65001:50 imet 10.1.254.201
                                 -                     -       -       0       i
 * >     RD: 65001:100 imet 10.1.254.201
                                 -                     -       -       0       i
 * >Ec   RD: 65002:10 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:10 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65002:20 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:20 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65002:30 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:30 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65002:40 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:40 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65002:50 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:50 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65002:100 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 *  ec   RD: 65002:100 imet 10.1.254.202
                                 10.1.254.202          -       100     0       65100 65002 i
 * >Ec   RD: 65203:10 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:10 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65203:30 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:30 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65203:50 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:50 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65203:200 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:200 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65003:444 ip-prefix 0.0.0.0/0
                                 10.1.254.5            -       100     0       65100 65003 65099 ?
 *  ec   RD: 65003:444 ip-prefix 0.0.0.0/0
                                 10.1.254.5            -       100     0       65100 65003 65099 ?
 * >Ec   RD: 65004:444 ip-prefix 0.0.0.0/0
                                 10.1.254.6            -       100     0       65100 65004 65099 ?
 *  ec   RD: 65004:444 ip-prefix 0.0.0.0/0
                                 10.1.254.6            -       100     0       65100 65004 65099 ?
 * >Ec   RD: 65003:444 ip-prefix 8.8.8.8/32
                                 10.1.254.5            -       100     0       65100 65003 65099 i
 *  ec   RD: 65003:444 ip-prefix 8.8.8.8/32
                                 10.1.254.5            -       100     0       65100 65003 65099 i
 * >Ec   RD: 65004:444 ip-prefix 8.8.8.8/32
                                 10.1.254.6            -       100     0       65100 65004 65099 i
 *  ec   RD: 65004:444 ip-prefix 8.8.8.8/32
                                 10.1.254.6            -       100     0       65100 65004 65099 i
 * >     RD: 65001:444 ip-prefix 100.0.0.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65002:444 ip-prefix 100.0.0.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 *  ec   RD: 65002:444 ip-prefix 100.0.0.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 * >     RD: 65001:111 ip-prefix 172.16.10.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65002:111 ip-prefix 172.16.10.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 *  ec   RD: 65002:111 ip-prefix 172.16.10.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 * >Ec   RD: 65003:222 ip-prefix 172.16.10.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65002 ?
 *  ec   RD: 65003:222 ip-prefix 172.16.10.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:111 ip-prefix 172.16.10.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:111 ip-prefix 172.16.10.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:222 ip-prefix 172.16.10.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:222 ip-prefix 172.16.10.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:444 ip-prefix 172.16.10.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:444 ip-prefix 172.16.10.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65203:111 ip-prefix 172.16.10.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 *  ec   RD: 65203:111 ip-prefix 172.16.10.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 * >     RD: 65001:111 ip-prefix 172.16.20.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65002:111 ip-prefix 172.16.20.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 *  ec   RD: 65002:111 ip-prefix 172.16.20.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 * >     RD: 65001:222 ip-prefix 172.16.30.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65002:222 ip-prefix 172.16.30.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 *  ec   RD: 65002:222 ip-prefix 172.16.30.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 * >Ec   RD: 65003:111 ip-prefix 172.16.30.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65002 ?
 *  ec   RD: 65003:111 ip-prefix 172.16.30.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:111 ip-prefix 172.16.30.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:111 ip-prefix 172.16.30.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:222 ip-prefix 172.16.30.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:222 ip-prefix 172.16.30.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:444 ip-prefix 172.16.30.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:444 ip-prefix 172.16.30.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65203:222 ip-prefix 172.16.30.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 *  ec   RD: 65203:222 ip-prefix 172.16.30.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 * >     RD: 65001:222 ip-prefix 172.16.40.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65002:222 ip-prefix 172.16.40.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 *  ec   RD: 65002:222 ip-prefix 172.16.40.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 * >Ec   RD: 65003:111 ip-prefix 172.16.40.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65002 ?
 *  ec   RD: 65003:111 ip-prefix 172.16.40.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:111 ip-prefix 172.16.40.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:111 ip-prefix 172.16.40.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:222 ip-prefix 172.16.40.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:222 ip-prefix 172.16.40.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >Ec   RD: 65004:444 ip-prefix 172.16.40.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 *  ec   RD: 65004:444 ip-prefix 172.16.40.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65002 ?
 * >     RD: 65001:333 ip-prefix 172.16.50.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65002:333 ip-prefix 172.16.50.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 *  ec   RD: 65002:333 ip-prefix 172.16.50.0/24
                                 10.1.254.202          -       100     0       65100 65002 ?
 * >Ec   RD: 65203:333 ip-prefix 172.16.50.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 *  ec   RD: 65203:333 ip-prefix 172.16.50.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 * >Ec   RD: 65003:111 ip-prefix 172.16.101.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 *  ec   RD: 65003:111 ip-prefix 172.16.101.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 * >Ec   RD: 65004:111 ip-prefix 172.16.101.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 *  ec   RD: 65004:111 ip-prefix 172.16.101.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 * >Ec   RD: 65003:222 ip-prefix 172.16.103.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 *  ec   RD: 65003:222 ip-prefix 172.16.103.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 * >Ec   RD: 65004:222 ip-prefix 172.16.103.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 *  ec   RD: 65004:222 ip-prefix 172.16.103.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 * >Ec   RD: 65003:444 ip-prefix 172.16.110.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 *  ec   RD: 65003:444 ip-prefix 172.16.110.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 * >Ec   RD: 65004:444 ip-prefix 172.16.110.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 *  ec   RD: 65004:444 ip-prefix 172.16.110.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 * >Ec   RD: 65003:111 ip-prefix 200.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65003:111 ip-prefix 200.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65003:222 ip-prefix 200.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65003:222 ip-prefix 200.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65004:111 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65004:111 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65004:222 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65004:222 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65004:444 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65004:444 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65203:444 ip-prefix 200.0.0.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 *  ec   RD: 65203:444 ip-prefix 200.0.0.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
Leaf-2#show ip route

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        10.1.1.2/31 is directly connected, Ethernet1
 C        10.1.2.2/31 is directly connected, Ethernet2
 C        10.1.251.0/31 is directly connected, Vlan4091
 B I      10.1.254.1/32 [200/0] via 10.1.251.0, Vlan4091
 B E      10.1.254.3/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 C        10.1.254.2/31 is directly connected, Loopback0
 B E      10.1.254.4/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.1.254.5/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.1.254.6/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.1.254.101/32 [200/0] via 10.1.1.2, Ethernet1
 B E      10.1.254.102/32 [200/0] via 10.1.2.2, Ethernet2
 C        10.1.254.200/31 is directly connected, Loopback1
 B E      10.1.254.202/32 [200/0] via 10.1.1.2, Ethernet1
                                  via 10.1.2.2, Ethernet2
 C        10.1.255.0/31 is directly connected, Vlan4090
 B E      10.2.254.1/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.2.254.2/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.2.254.3/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.2.254.101/32 [200/0] via 10.1.1.2, Ethernet1
                                  via 10.1.2.2, Ethernet2

Leaf-2#
Leaf-2#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP               Tunnel Type(s)
------------------ --------------
10.1.254.5         unicast
10.1.254.6         unicast
10.1.254.202       unicast, flood
10.2.254.3         unicast, flood

Total number of remote VTEPS:  4
Leaf-2#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI          VLAN       Source       Interface            802.1Q Tag
------------ ---------- ------------ -------------------- ----------
100010       10         static       Port-Channel10       10
                                     Vxlan1               10
100020       20         static       Port-Channel10       20
                                     Vxlan1               20
100030       30         static       Port-Channel10       30
                                     Vxlan1               30
100040       40         static       Port-Channel10       40
                                     Vxlan1               40
100050       50         static       Port-Channel10       50
                                     Vxlan1               50
100100       100        static       Port-Channel10       100
                                     Vxlan1               100

VNI to dynamic VLAN Mapping for Vxlan1
VNI       VLAN       VRF        Source
--------- ---------- ---------- ------------
111       4085       ADM        evpn
222       4088       DEV        evpn
333       4086       STOR       evpn
444       4087       EXT        evpn

Leaf-2#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  50  504c.8961.0a0f  EVPN      Vx1  10.2.254.3       1       0:02:42 ago
  50  50f5.6711.ea60  EVPN      Vx1  10.1.254.202     1       0:02:42 ago
4085  504b.d202.caff  EVPN      Vx1  10.2.254.3       1       12:53:52 ago
4085  5053.c96a.80ff  EVPN      Vx1  10.1.254.6       1       9:36:36 ago
4085  5057.ebb5.cd8b  EVPN      Vx1  10.1.254.202     1       12:55:11 ago
4085  50b6.a924.91d7  EVPN      Vx1  10.1.254.5       1       12:53:48 ago
4085  50d7.8378.a060  EVPN      Vx1  10.1.254.202     1       12:55:11 ago
4086  504b.d202.caff  EVPN      Vx1  10.2.254.3       1       12:53:52 ago
4086  5057.ebb5.cd8b  EVPN      Vx1  10.1.254.202     1       12:55:11 ago
4086  50d7.8378.a060  EVPN      Vx1  10.1.254.202     1       12:55:11 ago
4087  504b.d202.caff  EVPN      Vx1  10.2.254.3       1       12:53:52 ago
4087  5053.c96a.80ff  EVPN      Vx1  10.1.254.6       1       9:36:35 ago
4087  5057.ebb5.cd8b  EVPN      Vx1  10.1.254.202     1       12:55:11 ago
4087  50b6.a924.91d7  EVPN      Vx1  10.1.254.5       1       12:53:48 ago
4087  50d7.8378.a060  EVPN      Vx1  10.1.254.202     1       12:55:11 ago
4088  504b.d202.caff  EVPN      Vx1  10.2.254.3       1       12:53:52 ago
4088  5053.c96a.80ff  EVPN      Vx1  10.1.254.6       1       9:36:36 ago
4088  5057.ebb5.cd8b  EVPN      Vx1  10.1.254.202     1       12:55:11 ago
4088  50b6.a924.91d7  EVPN      Vx1  10.1.254.5       1       12:53:48 ago
4088  50d7.8378.a060  EVPN      Vx1  10.1.254.202     1       12:55:11 ago
Total Remote Mac Addresses for this criterion: 20
Leaf-2#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    502f.9e5a.b0e7    STATIC      Po100
  20    502f.9e5a.b0e7    STATIC      Po100
  30    502f.9e5a.b0e7    STATIC      Po100
  40    502f.9e5a.b0e7    STATIC      Po100
  50    502f.9e5a.b0e7    STATIC      Po100
  50    504c.8961.0a0f    DYNAMIC     Vx1        1       0:02:42 ago
  50    50f5.6711.ea60    DYNAMIC     Vx1        1       0:02:42 ago
 100    502f.9e5a.b0e7    STATIC      Po100
4085    502f.9e5a.b0e7    STATIC      Po100
4085    504b.d202.caff    DYNAMIC     Vx1        1       12:53:52 ago
4085    5053.c96a.80ff    DYNAMIC     Vx1        1       9:36:36 ago
4085    5057.ebb5.cd8b    DYNAMIC     Vx1        1       12:55:11 ago
4085    50b6.a924.91d7    DYNAMIC     Vx1        1       12:53:48 ago
4085    50d7.8378.a060    DYNAMIC     Vx1        1       12:55:11 ago
4086    502f.9e5a.b0e7    STATIC      Po100
4086    504b.d202.caff    DYNAMIC     Vx1        1       12:53:52 ago
4086    5057.ebb5.cd8b    DYNAMIC     Vx1        1       12:55:11 ago
4086    50d7.8378.a060    DYNAMIC     Vx1        1       12:55:11 ago
4087    502f.9e5a.b0e7    STATIC      Po100
4087    504b.d202.caff    DYNAMIC     Vx1        1       12:53:52 ago
4087    5053.c96a.80ff    DYNAMIC     Vx1        1       9:36:35 ago
4087    5057.ebb5.cd8b    DYNAMIC     Vx1        1       12:55:11 ago
4087    50b6.a924.91d7    DYNAMIC     Vx1        1       12:53:48 ago
4087    50d7.8378.a060    DYNAMIC     Vx1        1       12:55:11 ago
4088    502f.9e5a.b0e7    STATIC      Po100
4088    504b.d202.caff    DYNAMIC     Vx1        1       12:53:52 ago
4088    5053.c96a.80ff    DYNAMIC     Vx1        1       9:36:36 ago
4088    5057.ebb5.cd8b    DYNAMIC     Vx1        1       12:55:11 ago
4088    50b6.a924.91d7    DYNAMIC     Vx1        1       12:53:48 ago
4088    50d7.8378.a060    DYNAMIC     Vx1        1       12:55:11 ago
4090    502f.9e5a.b0e7    STATIC      Po100
4091    502f.9e5a.b0e7    STATIC      Po100
Total Mac Addresses for this criterion: 32

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf-2#
Leaf-2#show mlag
MLAG Configuration:
domain-id                          :                 100
local-interface                    :            Vlan4090
peer-address                       :          10.1.255.0
peer-link                          :     Port-Channel100
peer-config                        :          consistent

MLAG Status:
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:2f:9e:5a:b0:e7
dual-primary detection             :            Disabled
dual-primary interface errdisabled :               False

MLAG Ports:
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   0
Active-full                        :                   1

Leaf-2#
Leaf-2#show ip route vrf all

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        10.1.1.2/31 is directly connected, Ethernet1
 C        10.1.2.2/31 is directly connected, Ethernet2
 C        10.1.251.0/31 is directly connected, Vlan4091
 B I      10.1.254.1/32 [200/0] via 10.1.251.0, Vlan4091
 B E      10.1.254.3/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 C        10.1.254.2/31 is directly connected, Loopback0
 B E      10.1.254.4/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.1.254.5/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.1.254.6/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.1.254.101/32 [200/0] via 10.1.1.2, Ethernet1
 B E      10.1.254.102/32 [200/0] via 10.1.2.2, Ethernet2
 C        10.1.254.200/31 is directly connected, Loopback1
 B E      10.1.254.202/32 [200/0] via 10.1.1.2, Ethernet1
                                  via 10.1.2.2, Ethernet2
 C        10.1.255.0/31 is directly connected, Vlan4090
 B E      10.2.254.1/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.2.254.2/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.2.254.3/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.2.254.101/32 [200/0] via 10.1.1.2, Ethernet1
                                  via 10.1.2.2, Ethernet2


VRF: ADM
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 S        172.16.10.0/24 is directly connected, Vlan10
 S        172.16.20.0/24 is directly connected, Vlan20
 B E      172.16.30.0/24 [200/0] via VTEP 10.1.254.5 VNI 111 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                                 via VTEP 10.1.254.6 VNI 111 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.40.0/24 [200/0] via VTEP 10.1.254.5 VNI 111 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                                 via VTEP 10.1.254.6 VNI 111 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.101.0/31 [200/0] via VTEP 10.1.254.5 VNI 111 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
 B E      172.16.101.2/31 [200/0] via VTEP 10.1.254.6 VNI 111 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 C        192.1.10.0/24 is directly connected, Vlan10
 C        192.1.20.0/24 is directly connected, Vlan20
 B E      200.0.0.0/24 [200/0] via VTEP 10.1.254.5 VNI 111 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                               via VTEP 10.1.254.6 VNI 111 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1


VRF: DEV
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      172.16.10.0/24 [200/0] via VTEP 10.1.254.5 VNI 222 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                                 via VTEP 10.1.254.6 VNI 222 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 S        172.16.30.0/24 is directly connected, Vlan30
 S        172.16.40.0/24 is directly connected, Vlan40
 B E      172.16.103.0/31 [200/0] via VTEP 10.1.254.5 VNI 222 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
 B E      172.16.103.2/31 [200/0] via VTEP 10.1.254.6 VNI 222 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 C        192.1.30.0/24 is directly connected, Vlan30
 C        192.1.40.0/24 is directly connected, Vlan40
 B E      200.0.0.0/24 [200/0] via VTEP 10.1.254.5 VNI 222 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                               via VTEP 10.1.254.6 VNI 222 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1


VRF: EXT
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via VTEP 10.1.254.5 VNI 444 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                            via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1

 B E      8.8.8.8/32 [200/0] via VTEP 10.1.254.5 VNI 444 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                             via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 S        100.0.0.0/24 is directly connected, Vlan100
 B E      172.16.10.0/24 [200/0] via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.30.0/24 [200/0] via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.40.0/24 [200/0] via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.110.0/31 [200/0] via VTEP 10.1.254.5 VNI 444 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
 B E      172.16.110.2/31 [200/0] via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 C        192.1.100.0/24 is directly connected, Vlan100
 B E      200.0.0.0/24 [200/0] via VTEP 10.2.254.3 VNI 444 router-mac 50:4b:d2:02:ca:ff local-interface Vxlan1


VRF: STOR
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      172.16.50.122/32 [200/0] via VTEP 10.1.254.202 VNI 333 router-mac 50:57:eb:b5:cd:8b local-interface Vxlan1
 B E      172.16.50.211/32 [200/0] via VTEP 10.2.254.3 VNI 333 router-mac 50:4b:d2:02:ca:ff local-interface Vxlan1
 S        172.16.50.0/24 is directly connected, Vlan50
 C        192.1.50.0/24 is directly connected, Vlan50

Leaf-2#show ip arp vrf all

VRF: ADM
Address         Age (sec)  Hardware Addr   Interface
172.16.10.111     2:03:39  509c.edd9.92a1  Vlan10, not learned
172.16.20.111     2:10:47  509c.edd9.92a1  Vlan20, not learned

VRF: DEV
Address         Age (sec)  Hardware Addr   Interface
172.16.30.111     2:29:42  509c.edd9.92a1  Vlan30, not learned
172.16.40.111     2:01:10  509c.edd9.92a1  Vlan40, not learned

VRF: EXT
Address         Age (sec)  Hardware Addr   Interface
100.0.0.111       2:29:56  509c.edd9.92a1  Vlan100, not learned

VRF: STOR
Address         Age (sec)  Hardware Addr   Interface
172.16.50.111     2:15:24  509c.edd9.92a1  Vlan50, not learned
172.16.50.122           -  50f5.6711.ea60  Vlan50, Vxlan1
172.16.50.211           -  504c.8961.0a0f  Vlan50, Vxlan1

VRF: default
Address         Age (sec)  Hardware Addr   Interface
10.1.1.2          0:00:00  50ef.19d8.dff9  Ethernet1
10.1.2.2          0:00:00  50a5.3448.d4f7  Ethernet2
10.1.255.0        0:00:01  502f.9e5a.b0e7  Vlan4090, Port-Channel100
10.1.251.0        0:00:00  502f.9e5a.b0e7  Vlan4091, Port-Channel100
Leaf-2#

```
#### <u> LEAF3:</u>

```

Leaf-3#show bgp summary
BGP summary information for VRF default
Router identifier 10.1.254.3, local AS number 65002
Neighbor                 AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.1.1.4              65100 Established   IPv4 Unicast            Negotiated             10         10
10.1.2.4              65100 Established   IPv4 Unicast            Negotiated             10         10
10.1.251.3            65002 Established   IPv4 Unicast            Negotiated             13         13
10.1.254.101          65100 Established   L2VPN EVPN              Negotiated             46         46
10.1.254.102          65100 Established   L2VPN EVPN              Negotiated             46         46
Leaf-3#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.254.3, local AS number 65002
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec   RD: 65203:50 mac-ip 504c.8961.0a0f
                                 10.2.254.3            -       100     0       65100 65003 65201 65200 65203 i
 *  ec   RD: 65203:50 mac-ip 504c.8961.0a0f
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65203:50 mac-ip 504c.8961.0a0f 172.16.50.211
                                 10.2.254.3            -       100     0       65100 65003 65201 65200 65203 i
 *  ec   RD: 65203:50 mac-ip 504c.8961.0a0f 172.16.50.211
                                 10.2.254.3            -       100     0       65100 65003 65201 65200 65203 i
 * >Ec   RD: 65001:10 imet 10.1.254.201
                                 10.1.254.201          -       100     0       65100 65001 i
 *  ec   RD: 65001:10 imet 10.1.254.201
                                 10.1.254.201          -       100     0       65100 65001 i
 * >Ec   RD: 65001:20 imet 10.1.254.201
                                 10.1.254.201          -       100     0       65100 65001 i
 *  ec   RD: 65001:20 imet 10.1.254.201
                                 10.1.254.201          -       100     0       65100 65001 i
 * >Ec   RD: 65001:30 imet 10.1.254.201
                                 10.1.254.201          -       100     0       65100 65001 i
 *  ec   RD: 65001:30 imet 10.1.254.201
                                 10.1.254.201          -       100     0       65100 65001 i
 * >Ec   RD: 65001:40 imet 10.1.254.201
                                 10.1.254.201          -       100     0       65100 65001 i
 *  ec   RD: 65001:40 imet 10.1.254.201
                                 10.1.254.201          -       100     0       65100 65001 i
 * >Ec   RD: 65001:50 imet 10.1.254.201
                                 10.1.254.201          -       100     0       65100 65001 i
 *  ec   RD: 65001:50 imet 10.1.254.201
                                 10.1.254.201          -       100     0       65100 65001 i
 * >Ec   RD: 65001:100 imet 10.1.254.201
                                 10.1.254.201          -       100     0       65100 65001 i
 *  ec   RD: 65001:100 imet 10.1.254.201
                                 10.1.254.201          -       100     0       65100 65001 i
 * >     RD: 65002:10 imet 10.1.254.202
                                 -                     -       -       0       i
 * >     RD: 65002:20 imet 10.1.254.202
                                 -                     -       -       0       i
 * >     RD: 65002:30 imet 10.1.254.202
                                 -                     -       -       0       i
 * >     RD: 65002:40 imet 10.1.254.202
                                 -                     -       -       0       i
 * >     RD: 65002:50 imet 10.1.254.202
                                 -                     -       -       0       i
 * >     RD: 65002:100 imet 10.1.254.202
                                 -                     -       -       0       i
 * >Ec   RD: 65203:10 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:10 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65203:30 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:30 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65203:50 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:50 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65203:200 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 *  ec   RD: 65203:200 imet 10.2.254.3
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 i
 * >Ec   RD: 65003:444 ip-prefix 0.0.0.0/0
                                 10.1.254.5            -       100     0       65100 65003 65099 ?
 *  ec   RD: 65003:444 ip-prefix 0.0.0.0/0
                                 10.1.254.5            -       100     0       65100 65003 65099 ?
 * >Ec   RD: 65004:444 ip-prefix 0.0.0.0/0
                                 10.1.254.6            -       100     0       65100 65004 65099 ?
 *  ec   RD: 65004:444 ip-prefix 0.0.0.0/0
                                 10.1.254.6            -       100     0       65100 65004 65099 ?
 * >Ec   RD: 65003:444 ip-prefix 8.8.8.8/32
                                 10.1.254.5            -       100     0       65100 65003 65099 i
 *  ec   RD: 65003:444 ip-prefix 8.8.8.8/32
                                 10.1.254.5            -       100     0       65100 65003 65099 i
 * >Ec   RD: 65004:444 ip-prefix 8.8.8.8/32
                                 10.1.254.6            -       100     0       65100 65004 65099 i
 *  ec   RD: 65004:444 ip-prefix 8.8.8.8/32
                                 10.1.254.6            -       100     0       65100 65004 65099 i
 * >Ec   RD: 65001:444 ip-prefix 100.0.0.0/24
                                 10.1.254.201          -       100     0       65100 65001 ?
 *  ec   RD: 65001:444 ip-prefix 100.0.0.0/24
                                 10.1.254.201          -       100     0       65100 65001 ?
 * >     RD: 65002:444 ip-prefix 100.0.0.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65003:111 ip-prefix 100.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65001 ?
 *  ec   RD: 65003:111 ip-prefix 100.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65001 ?
 * >Ec   RD: 65003:222 ip-prefix 100.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65001 ?
 *  ec   RD: 65003:222 ip-prefix 100.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65001 ?
 * >Ec   RD: 65004:111 ip-prefix 100.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65001 ?
 *  ec   RD: 65004:111 ip-prefix 100.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65001 ?
 * >Ec   RD: 65004:222 ip-prefix 100.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65001 ?
 *  ec   RD: 65004:222 ip-prefix 100.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65001 ?
 * >Ec   RD: 65004:444 ip-prefix 100.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65001 ?
 *  ec   RD: 65004:444 ip-prefix 100.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65001 ?
 * >Ec   RD: 65001:111 ip-prefix 172.16.10.0/24
                                 10.1.254.201          -       100     0       65100 65001 ?
 *  ec   RD: 65001:111 ip-prefix 172.16.10.0/24
                                 10.1.254.201          -       100     0       65100 65001 ?
 * >     RD: 65002:111 ip-prefix 172.16.10.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65203:111 ip-prefix 172.16.10.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 *  ec   RD: 65203:111 ip-prefix 172.16.10.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 * >Ec   RD: 65001:111 ip-prefix 172.16.20.0/24
                                 10.1.254.201          -       100     0       65100 65001 ?
 *  ec   RD: 65001:111 ip-prefix 172.16.20.0/24
                                 10.1.254.201          -       100     0       65100 65001 ?
 * >     RD: 65002:111 ip-prefix 172.16.20.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65003:222 ip-prefix 172.16.20.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65001 ?
 *  ec   RD: 65003:222 ip-prefix 172.16.20.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65100 65001 ?
 * >Ec   RD: 65004:111 ip-prefix 172.16.20.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65001 ?
 *  ec   RD: 65004:111 ip-prefix 172.16.20.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65001 ?
 * >Ec   RD: 65004:222 ip-prefix 172.16.20.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65001 ?
 *  ec   RD: 65004:222 ip-prefix 172.16.20.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65001 ?
 * >Ec   RD: 65004:444 ip-prefix 172.16.20.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65001 ?
 *  ec   RD: 65004:444 ip-prefix 172.16.20.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65100 65001 ?
 * >Ec   RD: 65001:222 ip-prefix 172.16.30.0/24
                                 10.1.254.201          -       100     0       65100 65001 ?
 *  ec   RD: 65001:222 ip-prefix 172.16.30.0/24
                                 10.1.254.201          -       100     0       65100 65001 ?
 * >     RD: 65002:222 ip-prefix 172.16.30.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65203:222 ip-prefix 172.16.30.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 *  ec   RD: 65203:222 ip-prefix 172.16.30.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 * >Ec   RD: 65001:222 ip-prefix 172.16.40.0/24
                                 10.1.254.201          -       100     0       65100 65001 ?
 *  ec   RD: 65001:222 ip-prefix 172.16.40.0/24
                                 10.1.254.201          -       100     0       65100 65001 ?
 * >     RD: 65002:222 ip-prefix 172.16.40.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65001:333 ip-prefix 172.16.50.0/24
                                 10.1.254.201          -       100     0       65100 65001 ?
 *  ec   RD: 65001:333 ip-prefix 172.16.50.0/24
                                 10.1.254.201          -       100     0       65100 65001 ?
 * >     RD: 65002:333 ip-prefix 172.16.50.0/24
                                 -                     -       -       0       ?
 * >Ec   RD: 65203:333 ip-prefix 172.16.50.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 *  ec   RD: 65203:333 ip-prefix 172.16.50.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 * >Ec   RD: 65003:111 ip-prefix 172.16.101.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 *  ec   RD: 65003:111 ip-prefix 172.16.101.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 * >Ec   RD: 65004:111 ip-prefix 172.16.101.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 *  ec   RD: 65004:111 ip-prefix 172.16.101.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 * >Ec   RD: 65003:222 ip-prefix 172.16.103.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 *  ec   RD: 65003:222 ip-prefix 172.16.103.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 * >Ec   RD: 65004:222 ip-prefix 172.16.103.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 *  ec   RD: 65004:222 ip-prefix 172.16.103.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 * >Ec   RD: 65003:444 ip-prefix 172.16.110.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 *  ec   RD: 65003:444 ip-prefix 172.16.110.0/31
                                 10.1.254.5            -       100     0       65100 65003 i
 * >Ec   RD: 65004:444 ip-prefix 172.16.110.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 *  ec   RD: 65004:444 ip-prefix 172.16.110.2/31
                                 10.1.254.6            -       100     0       65100 65004 i
 * >Ec   RD: 65003:111 ip-prefix 200.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65003:111 ip-prefix 200.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65003:222 ip-prefix 200.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65003:222 ip-prefix 200.0.0.0/24
                                 10.1.254.5            -       100     0       65100 65003 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65004:111 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65004:111 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65004:222 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65004:222 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65004:444 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 *  ec   RD: 65004:444 ip-prefix 200.0.0.0/24
                                 10.1.254.6            -       100     0       65100 65004 65099 65003 65201 65200 65203 ?
 * >Ec   RD: 65203:444 ip-prefix 200.0.0.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
 *  ec   RD: 65203:444 ip-prefix 200.0.0.0/24
                                 10.2.254.3            -       100     0       65100 65004 65202 65200 65203 ?
Leaf-3#show ip route

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        10.1.1.4/31 is directly connected, Ethernet1
 C        10.1.2.4/31 is directly connected, Ethernet2
 C        10.1.251.2/31 is directly connected, Vlan4091
 B E      10.1.254.1/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.1.254.2/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 C        10.1.254.3/32 is directly connected, Loopback0
 B I      10.1.254.4/32 [200/0] via 10.1.251.3, Vlan4091
 B E      10.1.254.5/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.1.254.6/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.1.254.101/32 [200/0] via 10.1.1.4, Ethernet1
 B E      10.1.254.102/32 [200/0] via 10.1.2.4, Ethernet2
 B E      10.1.254.201/32 [200/0] via 10.1.1.4, Ethernet1
                                  via 10.1.2.4, Ethernet2
 C        10.1.254.202/32 is directly connected, Loopback1
 C        10.1.255.2/31 is directly connected, Vlan4090
 B E      10.2.254.1/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.2.254.2/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.2.254.3/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.2.254.101/32 [200/0] via 10.1.1.4, Ethernet1
                                  via 10.1.2.4, Ethernet2

Leaf-3#
Leaf-3#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP               Tunnel Type(s)
------------------ --------------
10.1.254.5         unicast
10.1.254.6         unicast
10.1.254.201       unicast, flood
10.2.254.3         unicast, flood

Total number of remote VTEPS:  4
Leaf-3#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI          VLAN       Source       Interface            802.1Q Tag
------------ ---------- ------------ -------------------- ----------
100010       10         static       Port-Channel10       10
                                     Vxlan1               10
100020       20         static       Port-Channel10       20
                                     Vxlan1               20
100030       30         static       Port-Channel10       30
                                     Vxlan1               30
100040       40         static       Port-Channel10       40
                                     Vxlan1               40
100050       50         static       Port-Channel10       50
                                     Vxlan1               50
100100       100        static       Port-Channel10       100
                                     Vxlan1               100

VNI to dynamic VLAN Mapping for Vxlan1
VNI       VLAN       VRF        Source
--------- ---------- ---------- ------------
111       4087       ADM        evpn
222       4088       DEV        evpn
333       4085       STOR       evpn
444       4086       EXT        evpn

Leaf-3#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  50  504c.8961.0a0f  EVPN      Vx1  10.2.254.3       1       0:05:10 ago
4085  502f.9e5a.b0e7  EVPN      Vx1  10.1.254.201     1       12:57:39 ago
4085  504b.d202.caff  EVPN      Vx1  10.2.254.3       1       12:56:20 ago
4085  5069.a447.888f  EVPN      Vx1  10.1.254.201     1       12:57:40 ago
4086  504b.d202.caff  EVPN      Vx1  10.2.254.3       1       12:56:20 ago
4086  5053.c96a.80ff  EVPN      Vx1  10.1.254.6       1       9:39:02 ago
4086  5069.a447.888f  EVPN      Vx1  10.1.254.201     1       12:57:40 ago
4086  50b6.a924.91d7  EVPN      Vx1  10.1.254.5       1       12:56:15 ago
4087  504b.d202.caff  EVPN      Vx1  10.2.254.3       1       12:56:20 ago
4087  5053.c96a.80ff  EVPN      Vx1  10.1.254.6       1       9:39:04 ago
4087  5069.a447.888f  EVPN      Vx1  10.1.254.201     1       12:57:40 ago
4087  50b6.a924.91d7  EVPN      Vx1  10.1.254.5       1       12:56:15 ago
4088  504b.d202.caff  EVPN      Vx1  10.2.254.3       1       12:56:20 ago
4088  5053.c96a.80ff  EVPN      Vx1  10.1.254.6       1       9:39:04 ago
4088  5069.a447.888f  EVPN      Vx1  10.1.254.201     1       12:57:40 ago
4088  50b6.a924.91d7  EVPN      Vx1  10.1.254.5       1       12:56:15 ago
Total Remote Mac Addresses for this criterion: 16
Leaf-3#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    5057.ebb5.cd8b    STATIC      Po100
  20    5057.ebb5.cd8b    STATIC      Po100
  30    5057.ebb5.cd8b    STATIC      Po100
  40    5057.ebb5.cd8b    STATIC      Po100
  50    504c.8961.0a0f    DYNAMIC     Vx1        1       0:05:10 ago
  50    5057.ebb5.cd8b    STATIC      Po100
 100    5057.ebb5.cd8b    STATIC      Po100
4085    502f.9e5a.b0e7    DYNAMIC     Vx1        1       12:57:39 ago
4085    504b.d202.caff    DYNAMIC     Vx1        1       12:56:20 ago
4085    5057.ebb5.cd8b    STATIC      Po100
4085    5069.a447.888f    DYNAMIC     Vx1        1       12:57:40 ago
4086    504b.d202.caff    DYNAMIC     Vx1        1       12:56:20 ago
4086    5053.c96a.80ff    DYNAMIC     Vx1        1       9:39:02 ago
4086    5057.ebb5.cd8b    STATIC      Po100
4086    5069.a447.888f    DYNAMIC     Vx1        1       12:57:40 ago
4086    50b6.a924.91d7    DYNAMIC     Vx1        1       12:56:15 ago
4087    504b.d202.caff    DYNAMIC     Vx1        1       12:56:20 ago
4087    5053.c96a.80ff    DYNAMIC     Vx1        1       9:39:04 ago
4087    5057.ebb5.cd8b    STATIC      Po100
4087    5069.a447.888f    DYNAMIC     Vx1        1       12:57:40 ago
4087    50b6.a924.91d7    DYNAMIC     Vx1        1       12:56:15 ago
4088    504b.d202.caff    DYNAMIC     Vx1        1       12:56:20 ago
4088    5053.c96a.80ff    DYNAMIC     Vx1        1       9:39:04 ago
4088    5057.ebb5.cd8b    STATIC      Po100
4088    5069.a447.888f    DYNAMIC     Vx1        1       12:57:40 ago
4088    50b6.a924.91d7    DYNAMIC     Vx1        1       12:56:15 ago
4090    5057.ebb5.cd8b    STATIC      Po100
4091    5057.ebb5.cd8b    STATIC      Po100
Total Mac Addresses for this criterion: 28

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf-3#
Leaf-3#show mlag
MLAG Configuration:
domain-id                          :                 202
local-interface                    :            Vlan4090
peer-address                       :          10.1.255.3
peer-link                          :     Port-Channel100
peer-config                        :          consistent

MLAG Status:
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:57:eb:b5:cd:8b
dual-primary detection             :            Disabled
dual-primary interface errdisabled :               False

MLAG Ports:
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   0
Active-full                        :                   1

Leaf-3#
Leaf-3#show ip route vrf all

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        10.1.1.4/31 is directly connected, Ethernet1
 C        10.1.2.4/31 is directly connected, Ethernet2
 C        10.1.251.2/31 is directly connected, Vlan4091
 B E      10.1.254.1/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.1.254.2/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 C        10.1.254.3/32 is directly connected, Loopback0
 B I      10.1.254.4/32 [200/0] via 10.1.251.3, Vlan4091
 B E      10.1.254.5/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.1.254.6/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.1.254.101/32 [200/0] via 10.1.1.4, Ethernet1
 B E      10.1.254.102/32 [200/0] via 10.1.2.4, Ethernet2
 B E      10.1.254.201/32 [200/0] via 10.1.1.4, Ethernet1
                                  via 10.1.2.4, Ethernet2
 C        10.1.254.202/32 is directly connected, Loopback1
 C        10.1.255.2/31 is directly connected, Vlan4090
 B E      10.2.254.1/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.2.254.2/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.2.254.3/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.2.254.101/32 [200/0] via 10.1.1.4, Ethernet1
                                  via 10.1.2.4, Ethernet2


VRF: ADM
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      100.0.0.0/24 [200/0] via VTEP 10.1.254.5 VNI 111 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                               via VTEP 10.1.254.6 VNI 111 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 S        172.16.10.0/24 is directly connected, Vlan10
 S        172.16.20.0/24 is directly connected, Vlan20
 B E      172.16.101.0/31 [200/0] via VTEP 10.1.254.5 VNI 111 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
 B E      172.16.101.2/31 [200/0] via VTEP 10.1.254.6 VNI 111 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 C        192.1.10.0/24 is directly connected, Vlan10
 C        192.1.20.0/24 is directly connected, Vlan20
 B E      200.0.0.0/24 [200/0] via VTEP 10.1.254.5 VNI 111 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                               via VTEP 10.1.254.6 VNI 111 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1


VRF: DEV
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      100.0.0.0/24 [200/0] via VTEP 10.1.254.5 VNI 222 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                               via VTEP 10.1.254.6 VNI 222 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.20.0/24 [200/0] via VTEP 10.1.254.5 VNI 222 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                                 via VTEP 10.1.254.6 VNI 222 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 S        172.16.30.0/24 is directly connected, Vlan30
 S        172.16.40.0/24 is directly connected, Vlan40
 B E      172.16.103.0/31 [200/0] via VTEP 10.1.254.5 VNI 222 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
 B E      172.16.103.2/31 [200/0] via VTEP 10.1.254.6 VNI 222 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 C        192.1.30.0/24 is directly connected, Vlan30
 C        192.1.40.0/24 is directly connected, Vlan40
 B E      200.0.0.0/24 [200/0] via VTEP 10.1.254.5 VNI 222 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                               via VTEP 10.1.254.6 VNI 222 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1


VRF: EXT
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via VTEP 10.1.254.5 VNI 444 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                            via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1

 B E      8.8.8.8/32 [200/0] via VTEP 10.1.254.5 VNI 444 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
                             via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 S        100.0.0.0/24 is directly connected, Vlan100
 B E      172.16.20.0/24 [200/0] via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 B E      172.16.110.0/31 [200/0] via VTEP 10.1.254.5 VNI 444 router-mac 50:b6:a9:24:91:d7 local-interface Vxlan1
 B E      172.16.110.2/31 [200/0] via VTEP 10.1.254.6 VNI 444 router-mac 50:53:c9:6a:80:ff local-interface Vxlan1
 C        192.1.100.0/24 is directly connected, Vlan100
 B E      200.0.0.0/24 [200/0] via VTEP 10.2.254.3 VNI 444 router-mac 50:4b:d2:02:ca:ff local-interface Vxlan1


VRF: STOR
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      172.16.50.211/32 [200/0] via VTEP 10.2.254.3 VNI 333 router-mac 50:4b:d2:02:ca:ff local-interface Vxlan1
 S        172.16.50.0/24 is directly connected, Vlan50
 C        192.1.50.0/24 is directly connected, Vlan50

Leaf-3#show ip arp vrf all

VRF: ADM
Address         Age (sec)  Hardware Addr   Interface
172.16.10.122     3:01:44  50f5.6711.ea60  Vlan10, not learned
172.16.20.122     2:47:06  50f5.6711.ea60  Vlan20, not learned

VRF: DEV
Address         Age (sec)  Hardware Addr   Interface
172.16.30.122     1:20:51  50f5.6711.ea60  Vlan30, not learned
172.16.40.122     2:28:08  50f5.6711.ea60  Vlan40, not learned

VRF: EXT
Address         Age (sec)  Hardware Addr   Interface
100.0.0.122       2:18:28  50f5.6711.ea60  Vlan100, not learned

VRF: STOR
Address         Age (sec)  Hardware Addr   Interface
172.16.50.122     2:13:33  50f5.6711.ea60  Vlan50, not learned
172.16.50.211           -  504c.8961.0a0f  Vlan50, Vxlan1

VRF: default
Address         Age (sec)  Hardware Addr   Interface
10.1.1.4          0:00:00  50ef.19d8.dff9  Ethernet1
10.1.2.4          0:00:00  50a5.3448.d4f7  Ethernet2
10.1.255.3        0:00:00  5057.ebb5.cd8b  Vlan4090, Port-Channel100
10.1.251.3        0:00:00  5057.ebb5.cd8b  Vlan4091, Port-Channel100



```
#### <u>Router:</u>

```

vlan 100,200
!
interface Ethernet1
   description Leaf-3
   switchport mode trunk

interface Loopback0
   ip address 8.8.8.8/32

interface Vlan100
   ip address 172.16.100.2/30
!
interface Vlan200
   ip address 172.16.200.2/30
!
ip routing
!
route-map DEL-AS-PATH permit 10
   set as-path match all replacement none
!
router bgp 65300
   router-id 172.16.0.0
   timers bgp 1 3
   neighbor Leaf-3 peer group
   neighbor Leaf-3 remote-as 65003
   neighbor Leaf-3 bfd
   neighbor Leaf-3 route-map DEL-AS-PATH out
   neighbor 172.16.100.1 peer group Leaf-3
   neighbor 172.16.200.1 peer group Leaf-3
   network 8.8.8.8/32


```


### Проверка:

<details>
<summary>Проверка на Spine-1</summary>

```
Spine-1#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.254.101, local AS number 65100
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >     RD: 65001:30 mac-ip 0050.7966.68aa
                                 10.1.254.1            -       100     0       65001 i
 * >     RD: 65001:30 mac-ip 0050.7966.68aa 172.16.30.11
                                 10.1.254.1            -       100     0       65001 i
 * >     RD: 65002:20 mac-ip 0050.7966.68ab
                                 10.1.254.2            -       100     0       65002 i
 * >     RD: 65002:20 mac-ip 0050.7966.68ab 172.16.20.22
                                 10.1.254.2            -       100     0       65002 i
 * >     RD: 65002:50 mac-ip 0050.7966.68ac
                                 10.1.254.2            -       100     0       65002 i
 * >     RD: 65002:50 mac-ip 0050.7966.68ac 172.16.50.22
                                 10.1.254.2            -       100     0       65002 i
 * >     RD: 65003:10 mac-ip 0050.7966.68ad
                                 10.1.254.3            -       100     0       65003 i
 * >     RD: 65003:10 mac-ip 0050.7966.68ad 172.16.10.33
                                 10.1.254.3            -       100     0       65003 i
 * >     RD: 65001:10 imet 10.1.254.1
                                 10.1.254.1            -       100     0       65001 i
 * >     RD: 65001:30 imet 10.1.254.1
                                 10.1.254.1            -       100     0       65001 i
 * >     RD: 65002:20 imet 10.1.254.2
                                 10.1.254.2            -       100     0       65002 i
 * >     RD: 65002:50 imet 10.1.254.2
                                 10.1.254.2            -       100     0       65002 i
 * >     RD: 65003:10 imet 10.1.254.3
                                 10.1.254.3            -       100     0       65003 i
 * >     RD: 65003:20 imet 10.1.254.3
                                 10.1.254.3            -       100     0       65003 i
 * >     RD: 65003:888 ip-prefix 8.8.8.8/32
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65003:999 ip-prefix 8.8.8.8/32
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65001:888 ip-prefix 172.16.10.0/24
                                 10.1.254.1            -       100     0       65001 i
 * >     RD: 65003:888 ip-prefix 172.16.10.0/24
                                 10.1.254.3            -       100     0       65003 i
 * >     RD: 65003:999 ip-prefix 172.16.10.0/24
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65002:999 ip-prefix 172.16.20.0/24
                                 10.1.254.2            -       100     0       65002 i
 * >     RD: 65003:888 ip-prefix 172.16.20.0/24
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65003:999 ip-prefix 172.16.20.0/24
                                 10.1.254.3            -       100     0       65003 i
 * >     RD: 65003:888 ip-prefix 172.16.20.22/32
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65001:999 ip-prefix 172.16.30.0/24
                                 10.1.254.1            -       100     0       65001 i
 * >     RD: 65003:888 ip-prefix 172.16.30.0/24
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65003:888 ip-prefix 172.16.30.11/32
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65002:888 ip-prefix 172.16.50.0/24
                                 10.1.254.2            -       100     0       65002 i
 * >     RD: 65003:999 ip-prefix 172.16.50.0/24
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65003:999 ip-prefix 172.16.50.22/32
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65003:888 ip-prefix 172.16.100.0/30
                                 10.1.254.3            -       100     0       65003 i
 * >     RD: 65003:999 ip-prefix 172.16.200.0/30
                                 10.1.254.3            -       100     0       65003 i
Spine-1#

```
</details>
<details>
<summary>Проверка на Spine-2</summary>

```

Spine-2#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.254.102, local AS number 65100
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >     RD: 65001:10 imet 10.1.254.1
                                 10.1.254.1            -       100     0       65001 i
 * >     RD: 65001:30 imet 10.1.254.1
                                 10.1.254.1            -       100     0       65001 i
 * >     RD: 65002:20 imet 10.1.254.2
                                 10.1.254.2            -       100     0       65002 i
 * >     RD: 65002:50 imet 10.1.254.2
                                 10.1.254.2            -       100     0       65002 i
 * >     RD: 65003:10 imet 10.1.254.3
                                 10.1.254.3            -       100     0       65003 i
 * >     RD: 65003:20 imet 10.1.254.3
                                 10.1.254.3            -       100     0       65003 i
 * >     RD: 65003:888 ip-prefix 8.8.8.8/32
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65003:999 ip-prefix 8.8.8.8/32
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65001:888 ip-prefix 172.16.10.0/24
                                 10.1.254.1            -       100     0       65001 i
 * >     RD: 65003:888 ip-prefix 172.16.10.0/24
                                 10.1.254.3            -       100     0       65003 i
 * >     RD: 65003:999 ip-prefix 172.16.10.0/24
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65002:999 ip-prefix 172.16.20.0/24
                                 10.1.254.2            -       100     0       65002 i
 * >     RD: 65003:888 ip-prefix 172.16.20.0/24
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65003:999 ip-prefix 172.16.20.0/24
                                 10.1.254.3            -       100     0       65003 i
 * >     RD: 65001:999 ip-prefix 172.16.30.0/24
                                 10.1.254.1            -       100     0       65001 i
 * >     RD: 65003:888 ip-prefix 172.16.30.0/24
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65002:888 ip-prefix 172.16.50.0/24
                                 10.1.254.2            -       100     0       65002 i
 * >     RD: 65003:999 ip-prefix 172.16.50.0/24
                                 10.1.254.3            -       100     0       65003 65300 i
 * >     RD: 65003:888 ip-prefix 172.16.100.0/30
                                 10.1.254.3            -       100     0       65003 i
 * >     RD: 65003:999 ip-prefix 172.16.200.0/30
                                 10.1.254.3            -       100     0       65003 i
Spine-2#

```
</details>
<details>
<summary>Проверка на LEAF1</summary>

```
Leaf-1#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.254.1, local AS number 65001
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >     RD: 65001:10 mac-ip 0050.7966.68a9
                                 -                     -       -       0       i
 * >     RD: 65001:10 mac-ip 0050.7966.68a9 172.16.10.11
                                 -                     -       -       0       i
 * >     RD: 65001:30 mac-ip 0050.7966.68aa
                                 -                     -       -       0       i
 * >     RD: 65001:30 mac-ip 0050.7966.68aa 172.16.30.11
                                 -                     -       -       0       i
 * >Ec   RD: 65002:20 mac-ip 0050.7966.68ab
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:20 mac-ip 0050.7966.68ab
                                 10.1.254.2            -       100     0       65100 65002 i
 * >Ec   RD: 65002:20 mac-ip 0050.7966.68ab 172.16.20.22
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:20 mac-ip 0050.7966.68ab 172.16.20.22
                                 10.1.254.2            -       100     0       65100 65002 i
 * >Ec   RD: 65002:50 mac-ip 0050.7966.68ac
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:50 mac-ip 0050.7966.68ac
                                 10.1.254.2            -       100     0       65100 65002 i
 * >Ec   RD: 65002:50 mac-ip 0050.7966.68ac 172.16.50.22
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:50 mac-ip 0050.7966.68ac 172.16.50.22
                                 10.1.254.2            -       100     0       65100 65002 i
 * >Ec   RD: 65003:10 mac-ip 0050.7966.68ad
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:10 mac-ip 0050.7966.68ad
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:10 mac-ip 0050.7966.68ad 172.16.10.33
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:10 mac-ip 0050.7966.68ad 172.16.10.33
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:20 mac-ip 0050.7966.68ae
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:20 mac-ip 0050.7966.68ae
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:20 mac-ip 0050.7966.68ae 172.16.20.33
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:20 mac-ip 0050.7966.68ae 172.16.20.33
                                 10.1.254.3            -       100     0       65100 65003 i
 * >     RD: 65001:10 imet 10.1.254.1
                                 -                     -       -       0       i
 * >     RD: 65001:30 imet 10.1.254.1
                                 -                     -       -       0       i
 * >Ec   RD: 65002:20 imet 10.1.254.2
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:20 imet 10.1.254.2
                                 10.1.254.2            -       100     0       65100 65002 i
 * >Ec   RD: 65002:50 imet 10.1.254.2
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:50 imet 10.1.254.2
                                 10.1.254.2            -       100     0       65100 65002 i
 * >Ec   RD: 65003:10 imet 10.1.254.3
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:10 imet 10.1.254.3
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:20 imet 10.1.254.3
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:20 imet 10.1.254.3
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:888 ip-prefix 8.8.8.8/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:888 ip-prefix 8.8.8.8/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65003:999 ip-prefix 8.8.8.8/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:999 ip-prefix 8.8.8.8/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >     RD: 65001:888 ip-prefix 172.16.10.0/24
                                 -                     -       -       0       i
 * >Ec   RD: 65003:888 ip-prefix 172.16.10.0/24
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:888 ip-prefix 172.16.10.0/24
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:999 ip-prefix 172.16.10.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:999 ip-prefix 172.16.10.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65003:999 ip-prefix 172.16.10.11/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:999 ip-prefix 172.16.10.11/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65002:999 ip-prefix 172.16.20.0/24
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:999 ip-prefix 172.16.20.0/24
                                 10.1.254.2            -       100     0       65100 65002 i
 * >Ec   RD: 65003:888 ip-prefix 172.16.20.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:888 ip-prefix 172.16.20.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65003:999 ip-prefix 172.16.20.0/24
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:999 ip-prefix 172.16.20.0/24
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:888 ip-prefix 172.16.20.22/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:888 ip-prefix 172.16.20.22/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >     RD: 65001:999 ip-prefix 172.16.30.0/24
                                 -                     -       -       0       i
 * >Ec   RD: 65003:888 ip-prefix 172.16.30.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:888 ip-prefix 172.16.30.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65003:888 ip-prefix 172.16.30.11/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:888 ip-prefix 172.16.30.11/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65002:888 ip-prefix 172.16.50.0/24
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:888 ip-prefix 172.16.50.0/24
                                 10.1.254.2            -       100     0       65100 65002 i
 * >Ec   RD: 65003:999 ip-prefix 172.16.50.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:999 ip-prefix 172.16.50.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65003:999 ip-prefix 172.16.50.22/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:999 ip-prefix 172.16.50.22/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65003:888 ip-prefix 172.16.100.0/30
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:888 ip-prefix 172.16.100.0/30
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:999 ip-prefix 172.16.200.0/30
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:999 ip-prefix 172.16.200.0/30
                                 10.1.254.3            -       100     0       65100 65003 i
Leaf-1#

Leaf-1#show ip route vrf all

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        10.1.1.0/31 is directly connected, Ethernet1
 C        10.1.2.0/31 is directly connected, Ethernet2
 C        10.1.254.1/32 is directly connected, Loopback0
 B E      10.1.254.2/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.1.254.3/32 [200/0] via 10.1.1.0, Ethernet1
                                via 10.1.2.0, Ethernet2
 B E      10.1.254.101/32 [200/0] via 10.1.1.0, Ethernet1
 B E      10.1.254.102/32 [200/0] via 10.1.2.0, Ethernet2


VRF: ADM
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      8.8.8.8/32 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.10.33/32 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 C        172.16.10.0/24 is directly connected, Vlan10
 B E      172.16.20.22/32 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.20.0/24 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.30.11/32 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.30.0/24 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.50.22/32 [200/0] via VTEP 10.1.254.2 VNI 888 router-mac 50:aa:28:c1:1f:a0 local-interface Vxlan1
 B E      172.16.50.0/24 [200/0] via VTEP 10.1.254.2 VNI 888 router-mac 50:aa:28:c1:1f:a0 local-interface Vxlan1
 B E      172.16.100.0/30 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1


VRF: DEV
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      8.8.8.8/32 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.10.11/32 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.10.0/24 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.20.22/32 [200/0] via VTEP 10.1.254.2 VNI 999 router-mac 50:aa:28:c1:1f:a0 local-interface Vxlan1
 B E      172.16.20.33/32 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.20.0/24 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
                                 via VTEP 10.1.254.2 VNI 999 router-mac 50:aa:28:c1:1f:a0 local-interface Vxlan1
 C        172.16.30.0/24 is directly connected, Vlan30
 B E      172.16.50.22/32 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.50.0/24 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.200.0/30 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1

Leaf-1#
Leaf-1#show ip bgp vrf ADM
BGP routing table information for VRF ADM
Router identifier 172.16.10.1, local AS number 65001
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    8.8.8.8/32             10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    8.8.8.8/32             10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >      172.16.10.0/24         -                     -       -          -       0       i
 *  Ec    172.16.10.0/24         10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.10.0/24         10.1.254.3            0       -          100     0       65100 65003 i
 * >Ec    172.16.10.33/32        10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.10.33/32        10.1.254.3            0       -          100     0       65100 65003 i
 * >Ec    172.16.20.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.20.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.20.22/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.20.22/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.30.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.30.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.30.11/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.30.11/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.50.0/24         10.1.254.2            0       -          100     0       65100 65002 i
 *  ec    172.16.50.0/24         10.1.254.2            0       -          100     0       65100 65002 i
 * >Ec    172.16.50.22/32        10.1.254.2            0       -          100     0       65100 65002 i
 *  ec    172.16.50.22/32        10.1.254.2            0       -          100     0       65100 65002 i
 * >Ec    172.16.100.0/30        10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.100.0/30        10.1.254.3            0       -          100     0       65100 65003 i
Leaf-1#show ip bgp vrf DEV
BGP routing table information for VRF DEV
Router identifier 172.16.30.1, local AS number 65001
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    8.8.8.8/32             10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    8.8.8.8/32             10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.10.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.10.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.10.11/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.10.11/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.20.0/24         10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.20.0/24         10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.20.0/24         10.1.254.2            0       -          100     0       65100 65002 i
 *  ec    172.16.20.0/24         10.1.254.2            0       -          100     0       65100 65002 i
 * >Ec    172.16.20.22/32        10.1.254.2            0       -          100     0       65100 65002 i
 *  ec    172.16.20.22/32        10.1.254.2            0       -          100     0       65100 65002 i
 * >Ec    172.16.20.33/32        10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.20.33/32        10.1.254.3            0       -          100     0       65100 65003 i
 * >      172.16.30.0/24         -                     -       -          -       0       i
 * >Ec    172.16.50.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.50.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.50.22/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.50.22/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.200.0/30        10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.200.0/30        10.1.254.3            0       -          100     0       65100 65003 i


```
</details>
<details>
<summary>Проверка на LEAF2</summary>

```

Leaf-2#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.254.2, local AS number 65002
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec   RD: 65001:30 mac-ip 0050.7966.68aa
                                 10.1.254.1            -       100     0       65100 65001 i
 *  ec   RD: 65001:30 mac-ip 0050.7966.68aa
                                 10.1.254.1            -       100     0       65100 65001 i
 * >Ec   RD: 65001:30 mac-ip 0050.7966.68aa 172.16.30.11
                                 10.1.254.1            -       100     0       65100 65001 i
 *  ec   RD: 65001:30 mac-ip 0050.7966.68aa 172.16.30.11
                                 10.1.254.1            -       100     0       65100 65001 i
 * >     RD: 65002:20 mac-ip 0050.7966.68ab
                                 -                     -       -       0       i
 * >     RD: 65002:20 mac-ip 0050.7966.68ab 172.16.20.22
                                 -                     -       -       0       i
 * >     RD: 65002:50 mac-ip 0050.7966.68ac
                                 -                     -       -       0       i
 * >     RD: 65002:50 mac-ip 0050.7966.68ac 172.16.50.22
                                 -                     -       -       0       i
 * >Ec   RD: 65003:10 mac-ip 0050.7966.68ad
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:10 mac-ip 0050.7966.68ad
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:10 mac-ip 0050.7966.68ad 172.16.10.33
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:10 mac-ip 0050.7966.68ad 172.16.10.33
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:20 mac-ip 0050.7966.68ae
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:20 mac-ip 0050.7966.68ae
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:20 mac-ip 0050.7966.68ae 172.16.20.33
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:20 mac-ip 0050.7966.68ae 172.16.20.33
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65001:10 imet 10.1.254.1
                                 10.1.254.1            -       100     0       65100 65001 i
 *  ec   RD: 65001:10 imet 10.1.254.1
                                 10.1.254.1            -       100     0       65100 65001 i
 * >Ec   RD: 65001:30 imet 10.1.254.1
                                 10.1.254.1            -       100     0       65100 65001 i
 *  ec   RD: 65001:30 imet 10.1.254.1
                                 10.1.254.1            -       100     0       65100 65001 i
 * >     RD: 65002:20 imet 10.1.254.2
                                 -                     -       -       0       i
 * >     RD: 65002:50 imet 10.1.254.2
                                 -                     -       -       0       i
 * >Ec   RD: 65003:10 imet 10.1.254.3
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:10 imet 10.1.254.3
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:20 imet 10.1.254.3
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:20 imet 10.1.254.3
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:888 ip-prefix 8.8.8.8/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:888 ip-prefix 8.8.8.8/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65003:999 ip-prefix 8.8.8.8/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:999 ip-prefix 8.8.8.8/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65001:888 ip-prefix 172.16.10.0/24
                                 10.1.254.1            -       100     0       65100 65001 i
 *  ec   RD: 65001:888 ip-prefix 172.16.10.0/24
                                 10.1.254.1            -       100     0       65100 65001 i
 * >Ec   RD: 65003:888 ip-prefix 172.16.10.0/24
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:888 ip-prefix 172.16.10.0/24
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:999 ip-prefix 172.16.10.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:999 ip-prefix 172.16.10.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >     RD: 65002:999 ip-prefix 172.16.20.0/24
                                 -                     -       -       0       i
 * >Ec   RD: 65003:888 ip-prefix 172.16.20.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:888 ip-prefix 172.16.20.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65003:999 ip-prefix 172.16.20.0/24
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:999 ip-prefix 172.16.20.0/24
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:888 ip-prefix 172.16.20.22/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:888 ip-prefix 172.16.20.22/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65001:999 ip-prefix 172.16.30.0/24
                                 10.1.254.1            -       100     0       65100 65001 i
 *  ec   RD: 65001:999 ip-prefix 172.16.30.0/24
                                 10.1.254.1            -       100     0       65100 65001 i
 * >Ec   RD: 65003:888 ip-prefix 172.16.30.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:888 ip-prefix 172.16.30.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65003:888 ip-prefix 172.16.30.11/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:888 ip-prefix 172.16.30.11/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >     RD: 65002:888 ip-prefix 172.16.50.0/24
                                 -                     -       -       0       i
 * >Ec   RD: 65003:999 ip-prefix 172.16.50.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:999 ip-prefix 172.16.50.0/24
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65003:999 ip-prefix 172.16.50.22/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 *  ec   RD: 65003:999 ip-prefix 172.16.50.22/32
                                 10.1.254.3            -       100     0       65100 65003 65300 i
 * >Ec   RD: 65003:888 ip-prefix 172.16.100.0/30
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:888 ip-prefix 172.16.100.0/30
                                 10.1.254.3            -       100     0       65100 65003 i
 * >Ec   RD: 65003:999 ip-prefix 172.16.200.0/30
                                 10.1.254.3            -       100     0       65100 65003 i
 *  ec   RD: 65003:999 ip-prefix 172.16.200.0/30
                                 10.1.254.3            -       100     0       65100 65003 i
Leaf-2#

Leaf-2#show ip route vrf all

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        10.1.1.2/31 is directly connected, Ethernet1
 C        10.1.2.2/31 is directly connected, Ethernet2
 B E      10.1.254.1/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 B E      10.1.254.3/32 [200/0] via 10.1.1.2, Ethernet1
                                via 10.1.2.2, Ethernet2
 C        10.1.254.2/31 is directly connected, Loopback0
 B E      10.1.254.101/32 [200/0] via 10.1.1.2, Ethernet1
 B E      10.1.254.102/32 [200/0] via 10.1.2.2, Ethernet2


VRF: ADM
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      8.8.8.8/32 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.10.33/32 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.10.0/24 [200/0] via VTEP 10.1.254.1 VNI 888 router-mac 50:0b:c4:64:e6:2a local-interface Vxlan1
                                 via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.20.22/32 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.20.0/24 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.30.11/32 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.30.0/24 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 C        172.16.50.0/24 is directly connected, Vlan50
 B E      172.16.100.0/30 [200/0] via VTEP 10.1.254.3 VNI 888 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1


VRF: DEV
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      8.8.8.8/32 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.10.0/24 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.20.33/32 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 C        172.16.20.0/24 is directly connected, Vlan20
 B E      172.16.30.11/32 [200/0] via VTEP 10.1.254.1 VNI 999 router-mac 50:0b:c4:64:e6:2a local-interface Vxlan1
 B E      172.16.30.0/24 [200/0] via VTEP 10.1.254.1 VNI 999 router-mac 50:0b:c4:64:e6:2a local-interface Vxlan1
 B E      172.16.50.22/32 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.50.0/24 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1
 B E      172.16.200.0/30 [200/0] via VTEP 10.1.254.3 VNI 999 router-mac 50:5b:d8:15:48:a3 local-interface Vxlan1

Leaf-2#
Leaf-2#show ip bgp vrf ADM
BGP routing table information for VRF ADM
Router identifier 172.16.50.252, local AS number 65002
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    8.8.8.8/32             10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    8.8.8.8/32             10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.10.0/24         10.1.254.1            0       -          100     0       65100 65001 i
 *  ec    172.16.10.0/24         10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.10.0/24         10.1.254.1            0       -          100     0       65100 65001 i
 *  ec    172.16.10.0/24         10.1.254.3            0       -          100     0       65100 65003 i
 * >Ec    172.16.10.33/32        10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.10.33/32        10.1.254.3            0       -          100     0       65100 65003 i
 * >Ec    172.16.20.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.20.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.20.22/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.20.22/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.30.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.30.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.30.11/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.30.11/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >      172.16.50.0/24         -                     -       -          -       0       i
 * >Ec    172.16.100.0/30        10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.100.0/30        10.1.254.3            0       -          100     0       65100 65003 i
Leaf-2#show ip bgp vrf DEV
BGP routing table information for VRF DEV
Router identifier 172.16.20.252, local AS number 65002
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    8.8.8.8/32             10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    8.8.8.8/32             10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.10.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.10.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >      172.16.20.0/24         -                     -       -          -       0       i
 *  Ec    172.16.20.0/24         10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.20.0/24         10.1.254.3            0       -          100     0       65100 65003 i
 * >Ec    172.16.20.33/32        10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.20.33/32        10.1.254.3            0       -          100     0       65100 65003 i
 * >Ec    172.16.30.0/24         10.1.254.1            0       -          100     0       65100 65001 i
 *  ec    172.16.30.0/24         10.1.254.1            0       -          100     0       65100 65001 i
 * >Ec    172.16.30.11/32        10.1.254.1            0       -          100     0       65100 65001 i
 *  ec    172.16.30.11/32        10.1.254.1            0       -          100     0       65100 65001 i
 * >Ec    172.16.50.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.50.0/24         10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.50.22/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 *  ec    172.16.50.22/32        10.1.254.3            0       -          100     0       65100 65003 65300 i
 * >Ec    172.16.200.0/30        10.1.254.3            0       -          100     0       65100 65003 i
 *  ec    172.16.200.0/30        10.1.254.3            0       -          100     0       65100 65003 i
Leaf-2#



```
</details>
<details>
<summary>Проверка на LEAF3</summary>

```

Leaf-3#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.254.3, local AS number 65003
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec   RD: 65001:30 mac-ip 0050.7966.68aa
                                 10.1.254.1            -       100     0       65100 65001 i
 *  ec   RD: 65001:30 mac-ip 0050.7966.68aa
                                 10.1.254.1            -       100     0       65100 65001 i
 * >Ec   RD: 65001:30 mac-ip 0050.7966.68aa 172.16.30.11
                                 10.1.254.1            -       100     0       65100 65001 i
 *  ec   RD: 65001:30 mac-ip 0050.7966.68aa 172.16.30.11
                                 10.1.254.1            -       100     0       65100 65001 i
 * >Ec   RD: 65002:20 mac-ip 0050.7966.68ab
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:20 mac-ip 0050.7966.68ab
                                 10.1.254.2            -       100     0       65100 65002 i
 * >Ec   RD: 65002:20 mac-ip 0050.7966.68ab 172.16.20.22
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:20 mac-ip 0050.7966.68ab 172.16.20.22
                                 10.1.254.2            -       100     0       65100 65002 i
 * >Ec   RD: 65002:50 mac-ip 0050.7966.68ac
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:50 mac-ip 0050.7966.68ac
                                 10.1.254.2            -       100     0       65100 65002 i
 * >Ec   RD: 65002:50 mac-ip 0050.7966.68ac 172.16.50.22
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:50 mac-ip 0050.7966.68ac 172.16.50.22
                                 10.1.254.2            -       100     0       65100 65002 i
 * >     RD: 65003:10 mac-ip 0050.7966.68ad
                                 -                     -       -       0       i
 * >     RD: 65003:10 mac-ip 0050.7966.68ad 172.16.10.33
                                 -                     -       -       0       i
 * >Ec   RD: 65001:10 imet 10.1.254.1
                                 10.1.254.1            -       100     0       65100 65001 i
 *  ec   RD: 65001:10 imet 10.1.254.1
                                 10.1.254.1            -       100     0       65100 65001 i
 * >Ec   RD: 65001:30 imet 10.1.254.1
                                 10.1.254.1            -       100     0       65100 65001 i
 *  ec   RD: 65001:30 imet 10.1.254.1
                                 10.1.254.1            -       100     0       65100 65001 i
 * >Ec   RD: 65002:20 imet 10.1.254.2
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:20 imet 10.1.254.2
                                 10.1.254.2            -       100     0       65100 65002 i
 * >Ec   RD: 65002:50 imet 10.1.254.2
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:50 imet 10.1.254.2
                                 10.1.254.2            -       100     0       65100 65002 i
 * >     RD: 65003:10 imet 10.1.254.3
                                 -                     -       -       0       i
 * >     RD: 65003:20 imet 10.1.254.3
                                 -                     -       -       0       i
 * >     RD: 65003:888 ip-prefix 8.8.8.8/32
                                 -                     -       100     0       65300 i
 * >     RD: 65003:999 ip-prefix 8.8.8.8/32
                                 -                     -       100     0       65300 i
 * >Ec   RD: 65001:888 ip-prefix 172.16.10.0/24
                                 10.1.254.1            -       100     0       65100 65001 i
 *  ec   RD: 65001:888 ip-prefix 172.16.10.0/24
                                 10.1.254.1            -       100     0       65100 65001 i
 * >     RD: 65003:888 ip-prefix 172.16.10.0/24
                                 -                     -       -       0       i
 * >     RD: 65003:999 ip-prefix 172.16.10.0/24
                                 -                     -       100     0       65300 i
 * >Ec   RD: 65002:999 ip-prefix 172.16.20.0/24
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:999 ip-prefix 172.16.20.0/24
                                 10.1.254.2            -       100     0       65100 65002 i
 * >     RD: 65003:888 ip-prefix 172.16.20.0/24
                                 -                     -       100     0       65300 i
 * >     RD: 65003:999 ip-prefix 172.16.20.0/24
                                 -                     -       -       0       i
 * >     RD: 65003:888 ip-prefix 172.16.20.22/32
                                 -                     -       100     0       65300 i
 * >Ec   RD: 65001:999 ip-prefix 172.16.30.0/24
                                 10.1.254.1            -       100     0       65100 65001 i
 *  ec   RD: 65001:999 ip-prefix 172.16.30.0/24
                                 10.1.254.1            -       100     0       65100 65001 i
 * >     RD: 65003:888 ip-prefix 172.16.30.0/24
                                 -                     -       100     0       65300 i
 * >     RD: 65003:888 ip-prefix 172.16.30.11/32
                                 -                     -       100     0       65300 i
 * >Ec   RD: 65002:888 ip-prefix 172.16.50.0/24
                                 10.1.254.2            -       100     0       65100 65002 i
 *  ec   RD: 65002:888 ip-prefix 172.16.50.0/24
                                 10.1.254.2            -       100     0       65100 65002 i
 * >     RD: 65003:999 ip-prefix 172.16.50.0/24
                                 -                     -       100     0       65300 i
 * >     RD: 65003:999 ip-prefix 172.16.50.22/32
                                 -                     -       100     0       65300 i
 * >     RD: 65003:888 ip-prefix 172.16.100.0/30
                                 -                     -       -       0       i
 * >     RD: 65003:999 ip-prefix 172.16.200.0/30
                                 -                     -       -       0       i
Leaf-3#

Leaf-3#show ip route vrf all

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        10.1.1.4/31 is directly connected, Ethernet1
 C        10.1.2.4/31 is directly connected, Ethernet2
 B E      10.1.254.1/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 B E      10.1.254.2/32 [200/0] via 10.1.1.4, Ethernet1
                                via 10.1.2.4, Ethernet2
 C        10.1.254.3/32 is directly connected, Loopback0
 B E      10.1.254.101/32 [200/0] via 10.1.1.4, Ethernet1
 B E      10.1.254.102/32 [200/0] via 10.1.2.4, Ethernet2


VRF: ADM
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      8.8.8.8/32 [200/0] via 172.16.100.2, Vlan100
 C        172.16.10.0/24 is directly connected, Vlan10
 B E      172.16.20.22/32 [200/0] via 172.16.100.2, Vlan100
 B E      172.16.20.0/24 [200/0] via 172.16.100.2, Vlan100
 B E      172.16.30.11/32 [200/0] via 172.16.100.2, Vlan100
 B E      172.16.30.0/24 [200/0] via 172.16.100.2, Vlan100
 B E      172.16.50.22/32 [200/0] via VTEP 10.1.254.2 VNI 888 router-mac 50:aa:28:c1:1f:a0 local-interface Vxlan1
 B E      172.16.50.0/24 [200/0] via VTEP 10.1.254.2 VNI 888 router-mac 50:aa:28:c1:1f:a0 local-interface Vxlan1
 C        172.16.100.0/30 is directly connected, Vlan100


VRF: DEV
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 B E      8.8.8.8/32 [200/0] via 172.16.200.2, Vlan200
 B E      172.16.10.0/24 [200/0] via 172.16.200.2, Vlan200
 B E      172.16.20.22/32 [200/0] via VTEP 10.1.254.2 VNI 999 router-mac 50:aa:28:c1:1f:a0 local-interface Vxlan1
 C        172.16.20.0/24 is directly connected, Vlan20
 B E      172.16.30.11/32 [200/0] via VTEP 10.1.254.1 VNI 999 router-mac 50:0b:c4:64:e6:2a local-interface Vxlan1
 B E      172.16.30.0/24 [200/0] via VTEP 10.1.254.1 VNI 999 router-mac 50:0b:c4:64:e6:2a local-interface Vxlan1
 B E      172.16.50.22/32 [200/0] via 172.16.200.2, Vlan200
 B E      172.16.50.0/24 [200/0] via 172.16.200.2, Vlan200
 C        172.16.200.0/30 is directly connected, Vlan200

Leaf-3#
Leaf-3#show ip bgp vrf ADM
BGP routing table information for VRF ADM
Router identifier 172.16.100.1, local AS number 65003
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      8.8.8.8/32             172.16.100.2          0       -          100     0       65300 i
 * >      172.16.10.0/24         -                     -       -          -       0       i
 *  Ec    172.16.10.0/24         10.1.254.1            0       -          100     0       65100 65001 i
 *  ec    172.16.10.0/24         10.1.254.1            0       -          100     0       65100 65001 i
 * >      172.16.20.0/24         172.16.100.2          0       -          100     0       65300 i
 * >      172.16.20.22/32        172.16.100.2          0       -          100     0       65300 i
 * >      172.16.30.0/24         172.16.100.2          0       -          100     0       65300 i
 * >      172.16.30.11/32        172.16.100.2          0       -          100     0       65300 i
 * >Ec    172.16.50.0/24         10.1.254.2            0       -          100     0       65100 65002 i
 *  ec    172.16.50.0/24         10.1.254.2            0       -          100     0       65100 65002 i
 * >Ec    172.16.50.22/32        10.1.254.2            0       -          100     0       65100 65002 i
 *  ec    172.16.50.22/32        10.1.254.2            0       -          100     0       65100 65002 i
 * >      172.16.100.0/30        -                     -       -          -       0       i
Leaf-3#show ip bgp vrf DEV
BGP routing table information for VRF DEV
Router identifier 172.16.200.1, local AS number 65003
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      8.8.8.8/32             172.16.200.2          0       -          100     0       65300 i
 * >      172.16.10.0/24         172.16.200.2          0       -          100     0       65300 i
 * >      172.16.20.0/24         -                     -       -          -       0       i
 *  Ec    172.16.20.0/24         10.1.254.2            0       -          100     0       65100 65002 i
 *  ec    172.16.20.0/24         10.1.254.2            0       -          100     0       65100 65002 i
 * >Ec    172.16.20.22/32        10.1.254.2            0       -          100     0       65100 65002 i
 *  ec    172.16.20.22/32        10.1.254.2            0       -          100     0       65100 65002 i
 * >Ec    172.16.30.0/24         10.1.254.1            0       -          100     0       65100 65001 i
 *  ec    172.16.30.0/24         10.1.254.1            0       -          100     0       65100 65001 i
 * >Ec    172.16.30.11/32        10.1.254.1            0       -          100     0       65100 65001 i
 *  ec    172.16.30.11/32        10.1.254.1            0       -          100     0       65100 65001 i
 * >      172.16.50.0/24         172.16.200.2          0       -          100     0       65300 i
 * >      172.16.50.22/32        172.16.200.2          0       -          100     0       65300 i
 * >      172.16.200.0/30        -                     -       -          -       0       i



```
</details>
<details>
<summary>Проверка на Router</summary>

```

Router#show ip bgp summary
BGP summary information for VRF default
Router identifier 172.16.0.0, local AS number 65300
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State                                                  PfxRcd PfxAcc
  172.16.100.1     4  65003           1313      1127    0    0 00:18:36 Estab                                                  3      3
  172.16.200.1     4  65003           1315      1123    0    0 00:18:33 Estab                                                  3      3
Router#show ip bgp summary
BGP summary information for VRF default
Router identifier 172.16.0.0, local AS number 65300
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  172.16.100.1     4  65003           1319      1132    0    0 00:18:41 Estab   3      3
  172.16.200.1     4  65003           1321      1128    0    0 00:18:38 Estab   3      3
Router#
Router#show ip bgp
BGP routing table information for VRF default
Router identifier 172.16.0.0, local AS number 65300
Route status codes: s - suppressed, * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

         Network                Next Hop            Metric  LocPref Weight  Path
 * >     8.8.8.8/32             -                     0       0       -       i
 * >     172.16.10.0/24         172.16.100.1          0       100     0       65003 i
 * >     172.16.20.0/24         172.16.200.1          0       100     0       65003 i
 * >     172.16.30.0/24         172.16.200.1          0       100     0       65003 65100 65001 i
 * >     172.16.50.0/24         172.16.100.1          0       100     0       65003 65100 65002 i
 * #     172.16.100.0/30        172.16.100.1          0       100     0       65003 i
 * #     172.16.200.0/30        172.16.200.1          0       100     0       65003 i
Router#
Router#
Router#
Router#ping 172.16.30.11 source 8.8.8.8
PING 172.16.30.11 (172.16.30.11) from 8.8.8.8 : 72(100) bytes of data.
80 bytes from 172.16.30.11: icmp_seq=1 ttl=62 time=50.6 ms
80 bytes from 172.16.30.11: icmp_seq=2 ttl=62 time=42.9 ms
80 bytes from 172.16.30.11: icmp_seq=3 ttl=62 time=33.9 ms
80 bytes from 172.16.30.11: icmp_seq=4 ttl=62 time=29.3 ms
80 bytes from 172.16.30.11: icmp_seq=5 ttl=62 time=22.0 ms

--- 172.16.30.11 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 43ms
rtt min/avg/max/mdev = 22.081/35.801/50.687/10.064 ms, pipe 5, ipg/ewma 10.753/42.526 ms
Router#ping 172.16.50.22 source 8.8.8.8
PING 172.16.50.22 (172.16.50.22) from 8.8.8.8 : 72(100) bytes of data.
80 bytes from 172.16.50.22: icmp_seq=1 ttl=62 time=40.9 ms
80 bytes from 172.16.50.22: icmp_seq=2 ttl=62 time=33.3 ms
80 bytes from 172.16.50.22: icmp_seq=3 ttl=62 time=24.1 ms
80 bytes from 172.16.50.22: icmp_seq=4 ttl=62 time=18.5 ms
80 bytes from 172.16.50.22: icmp_seq=5 ttl=62 time=10.6 ms

--- 172.16.50.22 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 70ms
rtt min/avg/max/mdev = 10.691/25.543/40.988/10.691 ms, pipe 4, ipg/ewma 17.612/32.494 ms



```
</details>
<details>
<summary>Проверка на клиентах</summary>

```
Mgmt-1:
===========

Mgmt-1> ping 172.16.30.11
84 bytes from 172.16.30.11 icmp_seq=1 ttl=59 time=23.766 ms
84 bytes from 172.16.30.11 icmp_seq=2 ttl=59 time=21.855 ms

Mgmt-1> ping 172.16.50.22
84 bytes from 172.16.50.22 icmp_seq=1 ttl=62 time=10.309 ms
84 bytes from 172.16.50.22 icmp_seq=2 ttl=62 time=9.507 ms

Mgmt-1> ping 172.16.20.33
84 bytes from 172.16.20.33 icmp_seq=1 ttl=60 time=25.309 ms
84 bytes from 172.16.20.33 icmp_seq=2 ttl=60 time=17.049 ms

Mgmt-1> ping 8.8.8.8
84 bytes from 8.8.8.8 icmp_seq=1 ttl=62 time=12.646 ms
84 bytes from 8.8.8.8 icmp_seq=2 ttl=62 time=13.552 ms


Test-2:
=========

Test-2> ping 172.16.20.33
84 bytes from 172.16.20.33 icmp_seq=1 ttl=64 time=9.204 ms
84 bytes from 172.16.20.33 icmp_seq=2 ttl=64 time=8.828 ms

Test-2> ping 172.16.50.22
84 bytes from 172.16.50.22 icmp_seq=1 ttl=59 time=23.744 ms
84 bytes from 172.16.50.22 icmp_seq=2 ttl=59 time=22.232 ms

Test-2> ping 172.16.30.11
84 bytes from 172.16.30.11 icmp_seq=1 ttl=62 time=17.898 ms
84 bytes from 172.16.30.11 icmp_seq=2 ttl=62 time=10.330 ms

Test-2> ping 8.8.8.8
84 bytes from 8.8.8.8 icmp_seq=1 ttl=62 time=12.758 ms
84 bytes from 8.8.8.8 icmp_seq=2 ttl=62 time=13.238 ms


Adm-2:
=========

Adm-2> ping 172.16.10.33
84 bytes from 172.16.10.33 icmp_seq=1 ttl=62 time=17.985 ms
84 bytes from 172.16.10.33 icmp_seq=2 ttl=62 time=9.813 ms

Adm-2> ping 172.16.20.22
84 bytes from 172.16.20.22 icmp_seq=1 ttl=59 time=24.365 ms
84 bytes from 172.16.20.22 icmp_seq=2 ttl=59 time=29.148 ms

Adm-2> ping 172.16.30.11
84 bytes from 172.16.30.11 icmp_seq=1 ttl=59 time=25.046 ms
84 bytes from 172.16.30.11 icmp_seq=2 ttl=59 time=21.254 ms

Adm-2> ping 8.8.8.8
84 bytes from 8.8.8.8 icmp_seq=1 ttl=62 time=22.399 ms
84 bytes from 8.8.8.8 icmp_seq=2 ttl=62 time=13.658 ms
 	

```
</details>

Также прилагаются полные конфигурационные файлы всего оборудования сети.
