# Secure & Monitored Cisco SD-WAN Enterprise Lab

**An SD-WAN fabric that does more than connect sites: it detects attackers and watches every device.**

I built a Cisco SD-WAN network in EVE-NG (1 HQ + 3 branches over Internet and MPLS), then closed the two gaps that SD-WAN alone leaves open: **what attackers do once they find you**, and **the health of the devices vManage cannot see**.

> A lab environment, built for learning and for a research paper (under review). It is not a production deployment.

![EVE-NG topology](images/topology-eve.png)

## At a glance

| | |
|---|---|
| **Fabric** | vManage, vSmart, vBond + 5 vEdge routers (2 at HQ, 3 at branches) |
| **Transports** | Internet (INET_ISP) and MPLS (MPLS_ISP) |
| **Routing** | OMP overlay, OSPF behind the edges, BGP at the branches |
| **High availability** | VRRP between the HQ vEdges, HSRP between the HQ core switches |
| **Security add-on** | Cowrie honeypot + ELK, isolated in its own VPN (VPN 102) |
| **Monitoring add-on** | Zabbix on a physical Raspberry Pi (a real, low-cost solution), SNMP monitoring of routers and switches |

## Why I went beyond a basic SD-WAN lab

A working SD-WAN lab is a good start. But I asked what it still cannot do:

| The gap | Why the SD-WAN does not solve it | What I added |
|---|---|---|
| **Security visibility** | The vEdge controls and encrypts traffic, but it does not show *who is probing the network or what an attacker does after getting in* | **Cowrie honeypot + ELK**: a deception layer that records attacker behavior, with ELK replacing log reading in the shell |
| **Device monitoring** | vManage monitors the SD-WAN edges, but not the LAN switches and traditional routers behind them | **Zabbix + SNMP**: one place to watch routers and switches |

Everything below is built around those two ideas.

## 1. The SD-WAN fabric

![vManage dashboard](images/vmanage-dashboard.png)

| Component | Role |
|---|---|
| vManage | Management, templates and monitoring |
| vSmart | Control plane (OMP) |
| vBond | Orchestrator, first contact for edges |
| vEdge1, vEdge2 | HQ edges (redundant pair) |
| vEdge3, vEdge4, vEdge5 | Branch edges, each with a branch router (IOS13, IOS14, IOS15) |
| INET_ISP, MPLS_ISP | The two transports connecting all sites |
| SW16, SW17, SW20 + access switches | HQ switching, with client hosts behind the access switches |
| ASA1 | Firewall at HQ |

Controllers use the system IP range 10.100.0.0/24. Each branch runs BGP (AS 65003, 65004 and 65005) and OSPF Area 0.

### Protocols and design

| Layer | Technology | Where |
|---|---|---|
| Overlay routing | OMP | vEdges and vSmart |
| LAN routing | OSPF | Routers and L3 switches behind the edges |
| External routing | BGP | Branch sites |
| Edge redundancy | VRRP | Between vEdge1 and vEdge2 at HQ |
| Core redundancy | HSRP | Between SW16 and SW20 at HQ |
| Services | NAT, DHCP | Site connectivity and Internet access |

Devices are configured from vManage with feature templates:

![vManage feature templates](images/vmanage-feature-templates.png)

The SD-WAN topology as vManage sees it, and its performance monitoring (tunnel latency, loss and jitter):

![Topology on vManage](images/topology-vmanage.png)

![vManage monitoring](images/vmanage-monitoring.png)

![vManage monitoring, second view](images/vmanage-monitoring-2.png)

![Latency, loss and jitter](images/vmanage-latency-loss-jitter.png)

## 2. Security layer: a honeypot the vEdge cannot be

A vEdge decides what traffic is allowed. A honeypot answers a different question: **what do attackers actually do when they get in?** So I added a Cowrie SSH honeypot with an ELK stack, to turn attacks into data.

**Why ELK:** reading raw logs in the shell is slow and out of date. Scrolling through text files makes it hard to see patterns. ELK collects the logs and makes them searchable and visual, so monitoring becomes fast and clear.

| Item | Detail |
|---|---|
| Host | Ubuntu Server 24.10 VM, outside EVE-NG, bridged NIC |
| Honeypot | Cowrie, SSH trap port 2222 |
| Isolation | Its own separate VPN (VPN 102) |
| Exposure | Port 2222 is open on the vEdge, and connections to it are forwarded directly to the honeypot in VPN 102 |
| Log pipeline | Cowrie, Logstash, Elasticsearch, Kibana |

**Design decision:** the honeypot lives in its own VPN, so even a successful attacker is kept away from the other VPNs.

![Honeypot in the topology](images/honeypot-topology.png)

### The full stack is running

![Cowrie start](images/cowrie-start.png)

![Cowrie status](images/cowrie-status.png)

![Elasticsearch status](images/elasticsearch-status.png)

![Logstash status](images/logstash-status.png)

![Kibana status](images/kibana-status.png)

### Attack simulation

I tested it from an attacker's point of view:

1. **Scan:** the attacker scans the vEdge for open ports and finds port 2222.
2. **Identify:** a service scan shows it runs SSH.
3. **Assume:** the attacker thinks an administrator moved SSH off the default port to hide it, which makes it look like a valuable target.
4. **Attack:** the attacker logs in.

The connection was forwarded straight to the honeypot in VPN 102. Cowrie accepted the login, emulated a Linux shell, and logged the session and every command typed.

![Cowrie log of the login test](images/cowrie-login-test.png)

![Cowrie output after the test](images/cowrie-after-test.png)

### Turning logs into insight

Instead of reading raw logs line by line in the shell, Logstash forwards the events to Elasticsearch, and Kibana visualizes them.

![Kibana view of Cowrie events](images/kibana-cowrie-output.png)

**Result:** an attack that a firewall would only block or allow becomes a recorded, searchable event.

## 3. Monitoring layer: seeing what vManage cannot

vManage shows the health of the SD-WAN edges. It does not show the LAN switches and traditional routers behind them. I added **Zabbix on a physical Raspberry Pi** to monitor those devices over **SNMP**, so the whole network is visible in one place.

**Why a Raspberry Pi:** it is a real hardware solution, not just another VM, and it is cheap. A small business can run network monitoring on a device that costs very little and uses almost no power.

| Item | Detail |
|---|---|
| Server | Zabbix on a physical Raspberry Pi (real hardware, low cost), bridged NIC |
| Reach | EVE-NG, the Ubuntu server and the Pi can all reach each other |
| Method | SNMP, covering most of the routers and switches in the lab |
| Extra nodes | Additional servers added as nodes in EVE-NG |
| Alerts | Notifications delivered to a phone through Telegram |

![Zabbix dashboard](images/zabbix-dashboard.png)

### Monitored hosts

![Zabbix hosts](images/zabbix-hosts.png)

### Graphs

Interface traffic on a core switch (SW20) and an access switch (Switch12), and the CPU of the Raspberry Pi itself:

![SW20 interface traffic](images/sw20-interface-g0-0-traffic.png)

![Switch12 interface traffic](images/access-sw12-interface-traffic.png)

![Raspberry Pi CPU](images/raspberry-pi-cpu.jpg)

### Alert notification

![Telegram notification](images/telegram-notification.jpg)

**Result:** switches and routers that vManage does not cover are monitored, graphed, and able to send me an alert.

## Verification on vEdge1

The overlay and routing checks on vEdge1.

**BFD sessions:** the data-plane tunnels between sites are up.

![BFD sessions](images/bfd-sessions.png)

**Routing tables per VPN:** the transport VPN (VPN 0) and the service VPNs (VPN 1, 100, 101 and 102).

![VPN 0 routes](images/vedge1-routes-vpn0.png)

![VPN 1 routes](images/vedge1-routes-vpn1.png)

![VPN 100 routes](images/vedge1-routes-vpn100.png)

![VPN 101 routes](images/vedge1-routes-vpn101.png)

![VPN 102 routes](images/vedge1-routes-vpn102.png)

**Configuration excerpts** (sensitive values removed):

![System configuration](images/vedge1-run-system.png)

![VPN 0 configuration](images/vedge1-run-vpn0.png)

## Skills demonstrated

- **SD-WAN:** deploying and operating a full Cisco SD-WAN fabric with vManage templates
- **Routing and switching:** OMP, OSPF, BGP, VRRP, HSRP, NAT, DHCP, VLANs
- **Network security:** deploying a honeypot, isolating it in its own VPN, and analyzing attack logs in ELK
- **Monitoring:** Zabbix with SNMP, graphs and alerting
- **Troubleshooting:** verifying the network with BFD, OMP and per-VPN routing tables
- **Design thinking:** finding the gaps in a design and closing them

## Repository structure

```
.
├── README.md
├── images/    # topology and verification screenshots
└── Docs/      # presentation and supporting documents
```

## Related work

- Research paper: *Integrated SD-WAN Framework with Cowrie Honeypot, ELK, and Zabbix for Network Orchestration, Threat Detection, and IoT Monitoring*, submitted to *Scientific Reports* (under review).
- Another project: an Ansible lab that automates the configuration of a Cisco C8000V.

## Author

Ezzaldien Osman, Junior Network Engineer
LinkedIn: [linkedin.com/in/ezzaldeen-osman-mohamed](https://linkedin.com/in/ezzaldeen-osman-mohamed)
