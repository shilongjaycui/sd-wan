# sd-wan

A mini SD-WAN lab that connects **GCP**, **AWS**, and an **on-prem** site (a Framework Laptop 13 running Ubuntu 24.04), with GCP's **Network Connectivity Center (NCC)** as the hub.

## Goal

My company is starting an SD-WAN project to connect its GCP, AWS, and on-prem environments using NCC. This repo is a small-scale rehearsal on personal accounts. I want to learn the real building blocks (HA VPN, Cloud Router, BGP, router appliances, WireGuard, failover) before touching production.

## Architecture

```
                         ┌────────────────────── GCP (hub) ───────────────────────┐
                         │  Org: shilongjaycui-org                                │
                         │  Shared VPC host: mini-sd-wan-shared-vpc-host          │
                         │                                                        │
                         │            ┌──────────── NCC hub ────────────┐         │
                         │            │                                 │         │
                         │   VPC spoke│     Hybrid spoke     Router appliance     │
                         │  10.10.0.0/16    (HA VPN)          spoke (VM)          │
                         │            │         │                │                │
                         │            │   Cloud Router     WireGuard + FRR        │
                         │            │   + HA VPN GW      + Cloud Router peering │
                         └────────────┼─────────┼────────────────┼────────────────┘
                                      │         │ IPsec + BGP    │ WireGuard + BGP
                                      │         │ (4 tunnels)    │ (laptop dials out,
                                      │         │                │  works behind NAT)
                         ┌────────────▼─────────▼──┐     ┌───────▼────────────────┐
                         │ AWS                     │     │ On-prem (laptop)       │
                         │ Site-to-Site VPN        │     │ WireGuard + FRR        │
                         │ (VGW/TGW), 10.20.0.0/16 │     │ netns subnets          │
                         └─────────────────────────┘     │ 10.30.0.0/16           │
                                                         └────────────────────────┘
```

- **GCP is the hub.** An NCC hub lives in the Shared VPC host project. The lab VPC attaches as a VPC spoke.
- **AWS is a hybrid spoke.** A GCP HA VPN gateway and Cloud Router connect to AWS Site-to-Site VPN over four IPsec tunnels, and BGP exchanges routes. The tunnels are registered as an NCC hybrid spoke.
- **On-prem is a router appliance spoke.** The laptop sits behind home NAT and its public IP changes, which managed cloud VPNs handle poorly. Instead, a small GCP VM runs WireGuard and FRR. The laptop dials out to it over WireGuard and peers BGP with it. The VM is registered as an NCC router appliance spoke and peers BGP with a Cloud Router. (The alternative, HA VPN directly from the laptop, is not used.)
- **Transit:** traffic from on-prem to AWS crosses the NCC hub, which needs site-to-site data transfer enabled on the hybrid spokes, in a supported region.

## Address plan

| Site       | CIDR           | Notes                                                      |
|------------|----------------|------------------------------------------------------------|
| GCP        | `10.10.0.0/16` | Shared VPC in the host project                             |
| AWS        | `10.20.0.0/16` | Lab VPC                                                    |
| On-prem    | `10.30.0.0/16` | Laptop; multiple subnets simulated with network namespaces |

Tunnel inside addresses, BGP ASNs, and per-subnet allocations are tracked in [`docs/address-plan.md`](docs/address-plan.md).

## Milestones

| Milestone | Scope | Done when |
|---|---|---|
| **M0: Foundation** | GCP org + Shared VPC host, AWS account, billing alerts on both, Terraform skeletons, address plan | `terraform plan` runs cleanly for both clouds and alerts are live |
| **M1: Cloud to cloud** | NCC hub, AWS attached via HA VPN + BGP | A GCP VM and an AWS VM reach each other over private IPs |
| **M2: On-prem** | Router appliance VM (WireGuard + FRR) as an NCC spoke, laptop peers BGP | Laptop ↔ GCP ↔ AWS reachability works over private IPs |
| **M3: SD-WAN behaviors** | Two underlays on the laptop (home Wi-Fi + phone hotspot) | Failover, path preference, and centrally managed config are demonstrated |
| **M4: Operate and tear down** | Monitoring, runbook, cost review | `terraform destroy` is verified to leave nothing billable |

Progress is tracked in GitHub milestones and issues.

## ⚠️ Cost and teardown

- **This lab costs money while it runs.** VPN tunnels, external IPs, NAT gateways, VMs, and egress bill hourly or per GB.
- **Budget alerts only notify. They do not stop anything.** Set them up anyway:
  - GCP: [Create a budget](https://docs.cloud.google.com/billing/docs/how-to/budgets)
  - AWS: [Create a cost budget](https://docs.aws.amazon.com/cost-management/latest/userguide/create-cost-budget.html), and also enable the **zero-spend budget** template.
- **Everything billable lives in Terraform.** Run `terraform destroy` at the end of every session, then check both consoles for leftovers such as static IPs, VPN connections, and customer gateways.

## Operational cautions

- **Laptop sleep:** while the laptop acts as the on-prem gateway, disable suspend and lid-close sleep. The laptop has had suspend/resume issues on Ubuntu, and a sleeping gateway makes outages ambiguous.
- **Security:** this lab bridges a home network to two clouds. Keep firewall rules and security groups least-privilege, give test VMs no public IPs, and expose only the WireGuard port on the appliance.
- **MTU/MSS:** IPsec and WireGuard overhead shrink the usable MTU. Clamp TCP MSS and test with `ping -M do -s <size>`.
- **BGP ASNs:** use private ASNs (64512–65534), unique per site. AWS and GCP each reserve or default some values, so check before picking.
- **AWS customer gateway:** needs a stable public IP. That's fine for the GCP HA VPN interfaces, and it's why the laptop doesn't use AWS VPN directly.
- **Org policies:** the GCP org is a new Cloud Identity org and may have secure-by-default constraints. If something is denied unexpectedly, check Organization Policies first.

## Repo layout

```
sd-wan/
├── README.md
├── docs/
│   ├── address-plan.md      # CIDRs, tunnel /30s, BGP ASNs
│   ├── runbook.md           # bring-up, teardown, troubleshooting
│   └── results/             # connectivity, failover, and MTU test results
├── terraform/
│   ├── gcp/                 # Shared VPC, NCC hub/spokes, HA VPN, Cloud Router, appliance VM
│   └── aws/                 # VPC, Site-to-Site VPN, customer gateways, test EC2
├── onprem/
│   ├── wireguard/           # laptop WireGuard config templates
│   ├── frr/                 # FRR configs (laptop + appliance)
│   └── netns/               # scripts that build the simulated on-prem subnets
└── scripts/                 # helpers: connectivity checks, leftover-resource audit
```

Directories are created as each milestone needs them.
