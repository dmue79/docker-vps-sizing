# Best VPS for Docker: How Much RAM Your Stack Actually Needs, What the Second Year Costs, and Which Plans Block Ports 80/443 (Plan Table + Sizing Rules)

Four gigabytes of RAM is the practical floor for Docker Compose running an app, a database and a cache on one machine. Below that, containers compete for the same memory and the kernel starts terminating them without warning. A lot of the cheap Docker VPS plans people bookmark sit under that line.

Three things decide whether a cheap host stays cheap: the spec your stack actually needs, the gap between the promotional price and the renewal, and how the provider bills outbound traffic. All three are covered below, with Sixtynet's current plan data checked line by line and the gaps marked rather than guessed at.

> **Short version.** For containers that serve HTTP to the public, Sixtynet's LAX-MICRO (4 cores / 4 GB / 80 GB NVMe, 10 TB traffic, DDoS protection) at **$400/year** is the smallest plan that clears the Compose floor. For outbound-only work that needs a residential IP — scraping, automation, account management — Residential-STARTER at **$260/year** runs one container and nothing more. 👉 [Open Sixtynet's plan list and current annual prices](https://sixtynet.net/aff.php?aff=39)

## Sixtynet plans for Docker workloads

Sixtynet runs out of a single Los Angeles datacenter on KVM virtualisation, AMD EPYC processors and NVMe storage, managed through a VirtFusion panel with a one-click Docker installer among its OS templates. Two product families matter for containers, and they behave very differently.

The **LAX line** gets a datacenter IP with DDoS protection and open ports. The **Residential line** gets a residential IP, and Sixtynet's own listing states that residential products do not open ports 80 and 443 because of IP risk control. That one sentence decides which family your stack belongs to before price enters the conversation.

| Plan | vCPU / RAM / NVMe | Traffic | Annual price (USD) | Order |
| --- | --- | --- | --- | --- |
| LAX-STARTER | 2 / 2 GB / 40 GB | 5 TB | $200 (list $240) | [ See plan](https://sixtynet.net/aff.php?aff=39) |
| LAX-MICRO | 4 / 4 GB / 80 GB | 10 TB | $400 (list $480) | [ See plan](https://sixtynet.net/aff.php?aff=39) |
| LAX-MEDIUM | 8 / 8 GB / 160 GB | 20 TB | $800 | [ See plan](https://sixtynet.net/aff.php?aff=39) |
| LAX-LARGE | 8 / 16 GB / 320 GB | 40 TB | $1,600 (list $1,920) | [ See plan](https://sixtynet.net/aff.php?aff=39) |
| Residential-STARTER | 2 / 2 GB / 40 GB | 5 TB | $260 (list $300) | [ See plan](https://sixtynet.net/aff.php?aff=39) |
| Residential-MICRO | 4 / 4 GB / 80 GB | 10 TB | Listed on the plan page | [ See plan](https://sixtynet.net/aff.php?aff=39) |
| Residential-MEDIUM | 8 / 8 GB / 160 GB | 20 TB | Listed on the plan page | [ See plan](https://sixtynet.net/aff.php?aff=39) |
| Residential-LARGE | 16 / 16 GB / 320 GB | 40 TB | Listed on the plan page | [ See plan](https://sixtynet.net/aff.php?aff=39) |
| Mobile-STARTER | 2 / 2 GB / 40 GB | 50 GB | $50 | [ See plan](https://sixtynet.net/aff.php?aff=39) |
| Mobile-MICRO | 4 / 4 GB / 80 GB | 50 GB | $100 | [ See plan](https://sixtynet.net/aff.php?aff=39) |

Every plan includes NVMe with 12,500 to 25,000 IOPS and a 2 Gbps to 10 Gbps premium uplink. Three residential tiers publish specs but not an annual figure on the plan listing, and the Mobile line caps traffic at 50 GB, which rules it out for anything that pulls images.

Two details worth knowing before checkout. Residential plans have discounts between 13% and 17% baked into the annual price, and the listing also mentions quarterly billing with its own discount. On the China-facing side, Sixtynet advertises China Unicom Premium AS9929 routing, free BGP session setup and 95th-percentile bandwidth billing, plus a hardened residential profile that rejects all external probes and scans. That profile issues refunds to account balance only.

## How much VPS your containers actually need

A benchmark-led guide to Docker hosts divides the market into three bands that hold up well: **1–2 vCPU / 2 GB** for a single container, a lightweight dev environment or a simple API; **2–4 vCPU / 4–8 GB** for app + database + cache, CI runners and moderate production; **4+ vCPU / 8+ GB** for orchestrated multi-service deployments, parallel builds and high container counts.

Those bands line up with the configuration rules most operators settle on. Docker Compose with two to four services wants 4 GB recommended (2 GB minimum) and 40 GB of NVMe. Five or more services in production pushes to 8 GB and 4 vCPU. Single-node K3s wants 4–8 GB and 60 GB of NVMe; a control plane plus one worker wants 8 GB total split across two machines.

K3s is the sensible lightweight orchestrator for a single VPS. It ships as one binary around 70 MB, bundles Traefik as ingress and CoreDNS, and Rancher's own documentation lists 1 GB as the floor, though anything real lands at 4 GB. Full kubeadm Kubernetes is a different budget: the control plane components plus etcd need about 2 GB per node before a single workload starts.

Two rules that save trouble later. Don't run Docker and K3s on the same machine, since K3s uses containerd rather than the Docker daemon and the two fight over the same resources. And size for peak container count, not average, because each container holds memory independently.

Mapping that onto the table: the 2 GB plans (LAX-STARTER, Residential-STARTER) are single-container machines. LAX-MICRO at 4 GB covers app + Postgres + Redis + reverse proxy. LAX-MEDIUM at 8 GB handles a single-node K3s or a five-service Compose stack with monitoring. LAX-LARGE at 16 GB is where parallel CI builds and multi-node K3s live.

## Why a shared host can't run Docker at all

Shared hosting restricts the kernel features Docker depends on: user namespaces for isolation, network namespaces per container, seccomp profile enforcement, cgroup-level resource control. Where those are locked down, the daemon may install and containers still misbehave. Even when it technically runs, the resource ceiling means your containers compete with everyone else on the box for CPU and memory.

The failure mode is ugly. Each container consumes memory independently, and when the host runs out, the kernel's OOM killer picks a victim immediately and without negotiation. That victim is usually the largest process, which is usually your database.

So the useful question isn't "does this host allow Docker" but "does this host give me KVM, full root, and RAM it actually guarantees". A review of Docker hosting providers weighted kernel and virtualisation at 20% and resource isolation at another 20%, and dropped providers that restrict kernel features or root access. That weighting is a fair summary of what breaks container workloads in practice.

## The specs that decide whether builds stay fast

Docker's overlay storage driver generates heavy random I/O during image builds, layer extraction and volume operations, which is why NVMe matters more than raw disk size. Sixtynet's 12,500 IOPS on the entry plans and 25,000 on the larger ones is the relevant number, not the 40 GB or 320 GB capacity.

CPU consistency is the second factor. Containers share the host CPU, so noisy neighbours and steal time slow down builds and runtime together, and burst-only capacity hides the problem until you're doing three builds at once. Network throughput is the third: multi-stage builds and large base images need sustained pull speeds, and every pull counts as outbound traffic on most providers.

Backups are where the line items hide. OVHcloud and Liquid Web charge for snapshots, Bluehost doesn't offer them at all, and Hostinger and Kamatera include them. If you plan to keep stateful containers on the box, confirm this before you pay rather than after.

## What the bill looks like after the first term

Hostinger is the cleanest current example of the first-term trap. Its Docker VPS promotional page lists a plan at **$6.49/month**, down from a $19.49 list price, and the same page states the plan **renews at $11.99/month for the same 24-month term**. That's nearly double the promotional rate for the identical server.

Sixtynet's structure is different but has its own risk. The listed annual prices already carry a 13% to 17% discount against a list figure, and no separate renewal rate is published. Budget against the list price ($300 for Residential-STARTER, $240 for LAX-STARTER) rather than the discounted one, and treat the difference as a bonus rather than a guarantee. A published third-party review quotes $140/year for the Residential-STARTER spec, which no longer matches the current listing, so the site's live figure is the one that counts. 👉 [Compare Sixtynet's annual rates against the list prices](https://sixtynet.net/aff.php?aff=39)

Egress runs on three models. Contabo bills flat-rate with unlimited outbound traffic under fair use. Hetzner includes 20 TB per month and charges per TB after that. DigitalOcean charges **$0.01/GiB** above the plan's included transfer. At that rate, a terabyte of overage costs about $10, which is tolerable until a CI pipeline rebuilds multi-gigabyte base images a few hundred times a month. Sixtynet includes 5 TB to 40 TB depending on plan but doesn't state an overage policy on the listing, so ask support if you expect to run near the cap.

One caution on headline prices: DigitalOcean's $4/month Droplet has roughly 512 MiB of RAM and cannot run a Compose stack. The 4 GB Droplet is $24/month. Any comparison that puts those two numbers side by side without the spec is misleading.

## How Sixtynet compares on the numbers

| Provider | Plan closest to the 4 GB Compose floor | Price as listed | Egress policy |
| --- | --- | --- | --- |
| Sixtynet | LAX-MICRO, 4 vCPU / 4 GB / 80 GB | $400/year | 10 TB included, overage policy not published |
| Contabo | Core VPS 4, 4 vCPU / 8 GB / 200 GB SSD | €5.24/month, 12-month term, tax included | Flat rate, unlimited under fair use |
| Hetzner | CPX22, 2 vCPU / 4 GB / 80 GB | €19.49/month, VAT excluded, EU region | 20 TB/month, then per TB |
| DigitalOcean | 2 vCPU / 4 GB / 80 GB | $24/month | $0.01/GiB above the included allowance |

Hetzner's figures reflect its June price adjustment and exclude VAT. DigitalOcean bills per second with a monthly cap, so you never exceed the listed rate. On hardware per dollar, Contabo is far ahead of the rest, and Sixtynet's 4 GB plan at $400/year is roughly $33/month equivalent against Contabo's €5.24 for twice the RAM.

That gap is the honest framing. Sixtynet isn't the cheap Docker host on raw specs. It's the host you buy when you need a residential IP that doesn't get flagged, or AS9929 routing into China, or 95th-percentile bandwidth billing for proxy-heavy work. If none of those apply to you, Contabo or Hetzner will serve the same containers for less. 👉 [Check the LAX line's current annual price before comparing](https://sixtynet.net/aff.php?aff=39)

The trade-offs on Sixtynet's side are real: one datacenter in Los Angeles means no EU or APAC presence, residential plans can't serve public web traffic on 80/443, and a published review puts the money-back window at three days rather than the usual 30. Verify the refund terms on the order page, since the hardened residential line already states that refunds go to account balance.

## When a VPS is the wrong purchase

Managed container platforms like Fly.io and Render remove infrastructure management entirely: you push an image and the platform handles provisioning, scaling and networking. The cost is more per unit of compute and less control over runtime, storage drivers and kernel parameters. They make sense when deployment speed matters more than control, or when nobody on the team owns Linux administration.

Managed Kubernetes is the other fork. Running your own cluster on a VPS avoids control plane fees that sit around $0.10/hour, roughly $73/month, on the major clouds, but you inherit certificate renewal, storage provisioning, load balancing and node failure. DigitalOcean's DOKS is a middle path that keeps the control plane managed while you only pay for worker Droplets.

And if your users are in Europe or Asia, a single Los Angeles location is a hard stop regardless of price.

## Picking a plan in one pass

- One container making outbound requests, residential IP required → **Residential-STARTER, $260/year**. 2 GB is below the Compose floor, so this is a single-service machine.
- Public web stack with app, database and cache → **LAX-MICRO, $400/year**. This is the cheapest plan here that meets the 4 GB rule, and it has open ports and DDoS protection.
- Five or more services, or single-node K3s → **LAX-MEDIUM, $800/year** at 8 GB, which matches the recommended K3s memory.
- Parallel builds or a two-node cluster → **LAX-LARGE, $1,600/year** at 16 GB.

Three things to confirm before you pay anywhere: whether the plan's IP allows inbound traffic on 80 and 443, what the renewal or post-discount rate is rather than the promotional one, and what happens when you cross the traffic allowance. Those three answers move the real monthly cost more than the headline price does.

👉 [Start from Sixtynet's plan page and confirm the annual rate at checkout](https://sixtynet.net/aff.php?aff=39)
