# Cisco SD-WAN Enterprise Lab

A hands-on Cisco SD-WAN lab built in EVE-NG. One HQ and three branch sites are connected over Internet and MPLS transports, with redundancy at HQ, OSPF and BGP toward the LANs, and NAT/DHCP for connectivity.

> This is a lab environment built for learning and for a research paper (under review). It is not a production deployment.

![EVE-NG topology](images/topology-eve.png)

## What this lab covers

- A complete Cisco SD-WAN fabric: vManage, vSmart, vBond and 5 vEdge routers (2 at HQ, 3 at branches)
- Two transports: Internet (INET_ISP) and MPLS (MPLS_ISP)
- OMP as the overlay routing protocol
- OSPF between routers and the L3 switches behind the edges, and BGP at the branch sites
- HQ redundancy at the edge and core layers
- NAT and DHCP for site connectivity
- Central management with vManage feature templates, and performance monitoring

## Components

| Component | Role |
|---|---|
| vManage | Management, templates and monitoring |
| vSmart | Control plane (OMP) |
| vBond | Orchestrator, first contact for edges |
| vEdge1, vEdge2 | HQ edges (redundant pair) |
| vEdge3, vEdge4, vEdge5 | Branch edges, each with a branch router (IOS13, IOS14, IOS15) |
| INET_ISP, MPLS_ISP | Two transports connecting all sites |
| SW16, SW17, SW20 + access switches | HQ switching, with client hosts behind the access switches |
| ASA1 | Firewall at HQ |

Controllers use the system IP range 10.100.0.0/24. Each branch runs BGP (AS 65003, 65004 and 65005) and OSPF Area 0.

## Protocols and features

| Layer | Technology | Where |
|---|---|---|
| Overlay routing | OMP | vEdges and vSmart |
| LAN routing | OSPF | Routers and L3 switches behind the edges |
| External routing | BGP | Branch sites |
| Edge redundancy | VRRP | Between vEdge1 and vEdge2 at HQ |
| Core redundancy | HSRP | Between SW16 and SW20 at HQ |
| Services | NAT, DHCP | Site connectivity and Internet access |

## Management with vManage

The fabric is managed centrally from vManage. The dashboard shows the controllers and edges, and device configuration is built with feature templates.

![vManage dashboard](images/vmanage-dashboard.png)

![vManage feature templates](images/vmanage-feature-templates.png)

The SD-WAN topology as seen by vManage:

![Topology on vManage](images/topology-vmanage.png)

## Monitoring

vManage monitoring and tunnel quality (latency, loss and jitter between sites):

![vManage monitoring](images/vmanage-monitoring.png)

![vManage monitoring, second view](images/vmanage-monitoring-2.png)

![Latency, loss and jitter](images/vmanage-latency-loss-jitter.png)

## Verification on vEdge1

### BFD sessions

`show bfd sessions` confirms the data-plane tunnels between sites are up.

![BFD sessions](images/bfd-sessions.png)

### Routing tables per VPN

Routes on vEdge1 for the transport VPN (VPN 0) and the service VPNs (VPN 1, 100, 101 and 102).

![VPN 0 routes](images/vedge1-routes-vpn0.png)

![VPN 1 routes](images/vedge1-routes-vpn1.png)

![VPN 100 routes](images/vedge1-routes-vpn100.png)

![VPN 101 routes](images/vedge1-routes-vpn101.png)

![VPN 102 routes](images/vedge1-routes-vpn102.png)

### Configuration excerpts

System and VPN 0 configuration on vEdge1 (sensitive values removed).

![System configuration](images/vedge1-run-system.png)

![VPN 0 configuration](images/vedge1-run-vpn0.png)

## Repository structure

```
.
├── README.md
├── images/    # topology and verification screenshots
└── Docs/      # presentation and supporting documents
```

## What I learned

- How the SD-WAN control plane (vBond, vSmart, vManage) forms and how OMP distributes routes
- Integrating the overlay with OSPF and BGP
- Building device configuration with vManage feature templates
- Checking the fabric with BFD, OMP and per-VPN routing tables

## Related work

Research paper: *Integrated SD-WAN Framework with Cowrie Honeypot, ELK, and Zabbix for Network Orchestration, Threat Detection, and IoT Monitoring*, submitted to *Scientific Reports* (under review).

## Author

Ezzaldien Osman, Junior Network Engineer
LinkedIn: [linkedin.com/in/ezzaldeen-osman-mohamed](https://linkedin.com/in/ezzaldeen-osman-mohamed)
