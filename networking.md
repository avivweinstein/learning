# Computer Networking — Living Notes

Personal learning doc. NVIDIA-flavored. Grows over time as questions come up.

**Conventions:**
- Core concepts live in the **Fundamentals** section — stable reference.
- New questions land in **Q&A Log** at the bottom, dated.
- When a Q&A topic matures, its content gets promoted up into Fundamentals.

---

## Fundamentals

### First, the mental model: think of networking as a postal service

Most of this doc leans on a single analogy: **computer networking is basically a postal service for data.** It's a useful mental model because nearly every piece maps:

- Your computer wants to send data → you're a household sending a letter.
- The data is chopped into **packets** → each packet is one envelope.
- Every envelope has a **destination address** on it.
- Mail doesn't teleport — it moves through **sorting facilities** on its way.
- Sometimes envelopes get **lost in the mail** — this matters a lot later.

Keep this picture in your head. When I introduce a new piece (NIC, switch, InfiniBand, Spectrum-X), it's always a version of something in this postal system.

### The basic vocabulary

Before we name hardware, a few terms that show up everywhere:

- **Packet** — a chunk of data with an address header stuck on the front. Computers never send "a file"; they chop it into packets, send each one separately, and the receiver reassembles. Envelopes, not pallets.
- **Address** — every networked thing has a numeric address so packets can find it. There are two kinds that coexist:
  - **MAC address** — hard-coded into the NIC at the factory. Think of it as the **serial number engraved on the back of your phone**: it's permanently tied to that specific physical device and **never changes**. Your laptop's Wi-Fi NIC has one MAC; your laptop's Ethernet NIC has a totally separate MAC — they're two different pieces of hardware. The MAC travels with the device everywhere. Format: 6 bytes, shown as `a4:c3:f0:85:ac:2d`.
  - **IP address** — assigned by the network you've joined. Think of it as the **phone number your carrier gives you**: it only makes sense on *that network*. Plug your laptop into your home Wi-Fi, the home router hands you one IP (say `192.168.1.42`). Take it to a coffee shop, their network hands you a different IP (say `10.0.0.73`). Same laptop, same MAC, **different IP each time** — because the IP is the device's address *within the current network it's joined to*. A piece of software on the network called a **DHCP server** hands out IPs when devices join.
  - **So:** your MAC is about *which physical piece of hardware you are*. Your IP is about *where you currently are on the network*. Both exist at the same time. Packets use the IP for the big-picture "where to route this" decision, and the MAC for the final "which device on this local segment to hand it to."
- **TCP/UDP port (e.g. port 22, port 80, port 8888)** — a **completely different thing** from the physical port on a NIC. These are numbers stamped in the packet header that tell the **receiving operating system which application** should handle the packet. If the IP address is the street address of an apartment building, the port number is the **apartment number** — once a packet arrives at the building, the port says which tenant (application) gets the delivery. Port 22 = SSH, 80 = HTTP, 443 = HTTPS, 8888 = Jupyter by convention. Has nothing to do with the physical plug on the side of your NIC. (Unfortunate terminology collision — we're stuck with it.)
- **Server CPU vs NIC — who has the MAC/IP?** The CPU in your server does **not** have its own MAC or IP. Addresses belong to **network interfaces** — i.e. NICs. When you say "my server has IP 10.0.0.5," what you actually mean is "the NIC plugged into my server has IP 10.0.0.5." A server with two NICs has two MACs and typically two IPs. The CPU just talks to the NIC over PCIe and asks it to send/receive things.
- **Bandwidth** — how *much* data can flow per second. "How wide is the road." Measured in Gb/s (gigabits per second) or Tb/s.
- **Latency** — how *long* it takes one packet to get from A to B. "How long is the drive." Measured in microseconds (µs) or milliseconds (ms).
- **Throughput** — actual achieved data rate in practice (usually less than the bandwidth number on the box).
- **Protocol** — the rulebook both sides agree to follow so they can understand each other. TCP, UDP, InfiniBand, RoCE, etc. are all protocols.

### NIC vs Switch (the two pieces of hardware)

This is the single most important distinction. Let's go slowly.

#### What is a NIC?

A **NIC** (Network Interface Card) is a piece of hardware that lives **inside one specific computer** and is that computer's only way to talk to the network. Without a NIC, a server is deaf and mute — it can compute, but it can't send anything to or receive anything from the outside world.

**Postal analogy:** a NIC is **the mail slot on your front door**. It's the one and only opening through which mail enters and leaves your house. Every letter you send goes out through it; every letter you receive comes in through it. Your house has exactly one (or maybe a couple — some servers have multiple NICs, just like some houses have a front and a back mail slot).

**Physically:** a NIC is usually a **PCIe card** that plugs into the motherboard, or a chip soldered directly on it. It has:
- One or more **ports** (the physical holes where cables plug in)
- A **MAC address** burned into it at the factory
- Its own small ASIC chip to do the low-level packet work
- Sometimes its own memory (for buffering packets)

On the outside, you see the cable ports. A ConnectX-8 NIC might have one or two ports, each capable of 800 Gb/s.

**How many physical ports does a NIC have?** Typically **one or two**. Sometimes four on specialty parts. Not dozens — NICs are endpoints, not aggregators. If a NIC has two ports, they're usually used for redundancy (two independent paths to two different switches so one cable failure doesn't take the server offline) or for extra bandwidth (both ports active, doubling total throughput). Remember: these are **physical ports** (holes for cables), not TCP/UDP ports — completely different concept.

**What a NIC does:**
- Takes data from your computer's memory, wraps it in packet headers (addresses, checksums, etc.), and pushes the bits down the cable
- Receives bits coming in from the cable, parses the headers, checks they're actually addressed to this machine, and hands the data up to the computer
- Handles all the low-level electrical/optical signaling with the other end of the cable

**Key point:** one NIC connects to **one** cable going to **one** other destination. If you only had NICs and cables, you could connect exactly two machines together (A's NIC ↔ cable ↔ B's NIC). To connect *many* machines, you need switches.

#### Dumb NIC vs SmartNIC vs DPU

Not all NICs are created equal. There's a hierarchy of "how much can this NIC do on its own?":

- **Basic NIC ("dumb" NIC)** — moves packets between the wire and host memory. Has firmware but limited programmability. Does hardware offloads like checksums, RSS, basic RDMA. The entire consumer-market NIC space is this.
- **SmartNIC** — a NIC with significant *programmable* compute onboard, typically ARM cores, FPGAs, or P4-programmable packet-processing engines. Can run customer-written packet filters, encryption, custom protocol offloads. Not a full OS.
- **DPU (Data Processing Unit)** — NVIDIA's term for the next tier up. Basically a SmartNIC that's so capable it's really a **tiny complete server on a card**: full ARM CPU complex, DRAM, PCIe switch, running a real Linux distribution. A BlueField DPU runs Ubuntu on its ARM cores and you can SSH into it like any server.

**Why this hierarchy matters:** in a modern AI cluster, the main server CPU (the "host CPU" — the big x86 or Grace CPU on the motherboard) is busy running training workloads. You don't want it also handling network traffic management, storage protocol translation, encryption, firewalling, etc. A DPU **offloads all that infrastructure work from the host CPU onto itself**, so the host CPU can stay focused on the application. When you see "offloaded from the host CPU" in NVIDIA marketing, "host" means the main motherboard CPU — the one actually running the workload — not anything on the NIC.

#### What is a Switch?

A **switch** is a standalone **box in a server rack** whose job is to forward packets between many NICs. But "is a switch just hardware?" is a great question with a two-part answer:

- **The actual packet-forwarding is done in hardware** — a dedicated switching ASIC reads addresses and forwards packets at nanosecond speeds. This is the "switch as a dumb fast box" picture most people have.
- **But a switch also runs a real Linux-based operating system** on a separate management CPU alongside the ASIC. That OS (the **NOS** — Network Operating System) doesn't touch packets in the hot path, but it handles configuration, runs routing protocols like BGP, monitors health, and programs the rules the ASIC enforces. Without the NOS the switch would be a brainless "everything goes everywhere" box. The "Software on networking hardware" section below covers this in depth (Cumulus, SONiC, NVOS, etc.).

**Postal analogy:** a switch is the **regional post office sorting facility**. The sorting belts (ASIC) fling envelopes between loading docks at blinding speed, following whatever routing rules are currently in effect. A postmaster (the management CPU running a NOS) in the back office updates those rules, coordinates with other post offices, and keeps the operation running — but personally handles almost no mail.

**Physically:** a switch is a **rack-mounted box**, typically 1U to 4U tall (1U = about 1.75 inches thick, fits in a standard 19-inch server rack). It has:
- Tens to hundreds of **ports** on the front (where cables plug in)
- A beefy cooling system (the switching chip runs hot)
- One or two power supplies
- A **management CPU** — a small x86 or ARM processor with a few GB of DRAM, running a full Linux-based NOS
- The **switching ASIC** — the big custom chip that does the actual packet forwarding at terabit speeds

**Port counts in practice:**
- Small top-of-rack switch: 32–48 ports
- Typical modern AI-datacenter leaf: 64 ports at 400–800 Gb/s
- High-end spine switch: 128 ports
- NVIDIA Quantum-X800 (biggest): **144 ports × 800 Gb/s** in a 4U box — roughly 115 Tb/s of total switching capacity

**What a switch does:**
- Packet arrives on port 3
- The ASIC reads the destination address in the header
- It consults its internal **forwarding table** to figure out "which port should I send this out?" (the NOS programmed this table earlier)
- It sends the packet out the correct port (say, port 47)
- All of the above takes on the order of **nanoseconds**

A switch does not store the packet, doesn't open the payload, doesn't do anything useful with the data. The packet-moving part is a purely mechanical high-speed sorter. The **software on the side** is what makes the sorting rules *smart* — knowing where Server B actually lives, adapting when a link fails, etc.

#### Why you need both — a concrete example

Let's trace one packet.

Say **Server A** wants to send a chunk of training data to **Server B**, and there's a switch between them.

1. **Server A's CPU** writes the data into memory and tells its NIC "send this to Server B, here's the address."
2. **Server A's NIC** wraps the data in a packet, stamps Server B's address on it, and pushes the bits out its port.
3. The bits travel down the **cable** (copper or fiber) to **port 3 of the switch**.
4. The **switch** reads the address, figures out Server B is on port 47, and forwards the packet out port 47.
5. The packet travels down another cable to **Server B's NIC**.
6. **Server B's NIC** reads the packet, confirms it's actually for this machine, and puts the data into Server B's memory.
7. **Server B's CPU** gets a notification that new data has arrived.

Every one of those pieces is necessary. No NIC on Server A = it can't send. No switch = Server A can only reach one other machine (the one it's directly cabled to). No NIC on Server B = it can't receive.

#### Putting it in a datacenter — "leaf-spine"

A real datacenter has thousands of servers. You can't plug 10,000 servers into one giant switch — switches have limited port counts. So you build a hierarchy:

```
                     [ SPINE SWITCH ]   [ SPINE SWITCH ]
                            |                  |
              +-------------+-----+-----+------+-------------+
              |                   |     |                    |
       [ LEAF / ToR ]      [ LEAF / ToR ]              [ LEAF / ToR ]
         |  |  |  |          |  |  |  |                  |  |  |  |
       server server...    server server...          server server...
       (rack 1)            (rack 2)                  (rack N)
```

- **Leaf switches** (also called **ToR** — Top-of-Rack) live at the top of each server rack. Every server in the rack cables up to the ToR.
- **Spine switches** sit at a higher level and connect all the ToRs to each other.
- When Server A in rack 1 wants to talk to Server B in rack 5, the packet goes: A → ToR-1 → Spine → ToR-5 → B. Three hops.

This is called a **leaf-spine** topology and it's the standard way modern datacenters are wired.

**Does a leaf switch only talk to the NICs in its rack?** Not exactly — a leaf has *two groups* of ports:

- **Downlink / server-facing ports** go to the NICs in its own rack. These are the only NICs the leaf talks to directly.
- **Uplink / fabric-facing ports** go **up to the spine switches**. This is how the leaf exchanges traffic with other racks.

So a 64-port leaf might be configured as 48 ports downward (48 servers in the rack) plus 16 ports upward (to 16 different spine switches). The exact split depends on **oversubscription** — if you give the servers more total downlink bandwidth than you give the leaf's uplink bandwidth, you're betting that not all servers will talk off-rack at peak at the same time. AI fabrics typically run **1:1 non-oversubscribed** (equal downlink and uplink bandwidth) because AI workloads actually do saturate everything.

**How many leaves can one spine connect to?** As many as it has ports. A Quantum-X800 (144 ports) can theoretically connect to 144 leaves if you use just one uplink per leaf. In practice you want **multiple uplinks per leaf** for redundancy (so a single link failure doesn't partition the fabric), so the real ratio is lower.

**How many spines do you need?** A typical modern AI datacenter uses **4–16 spine switches**. You need *at least* 2 for redundancy, and you scale up to match the total uplink bandwidth coming from your leaves. Every leaf connects to every spine — that's a "CLOS" topology. Example sizing:

- 32 racks × 48 servers = 1,536 servers
- 32 leaf switches (one per rack), each with 16 uplinks
- 16 spine switches, each receiving 32 uplinks (one from each leaf)

**Is there anything above the spine?** Yes, for very large fabrics. When one set of spines can't fit enough leaves, you add another tier:

- **Leaf → Spine → Super-spine (or "core")** — a 3-tier CLOS. Used for 100k+ GPU fabrics.
- The super-spine layer aggregates multiple "pods" (each pod being its own leaf-spine fabric) into one big fabric.
- Beyond that, at the scale of connecting *multiple datacenters together*, you get **DCI (Data Center Interconnect)** gear — longer-distance, higher-latency links between geographically separated sites. NVIDIA's **Spectrum-XGS** is specifically for this "connect several datacenters into one logical AI factory" job.

So the hierarchy, top to bottom, at full scale: **DCI → super-spine → spine → leaf → NIC → server**.

### Ethernet vs InfiniBand (the two universes)

These are the two competing **protocols** (rulebooks) that NICs and switches can speak. They are not the same hardware — a switch speaks one or the other, not both at runtime.

#### Ethernet — the universal mail service

Ethernet is what the entire internet runs on. It was invented in 1973 and has been improved continuously since. It's everywhere: your home Wi-Fi router, your laptop, every office building, every cloud datacenter.

Ethernet's design philosophy: **best-effort delivery.** The network tries to deliver your packet, but if something goes wrong (congestion, a noisy cable, a packet arriving out of order), the packet gets **dropped** — silently thrown away. It's then the sender's job (via a higher-level protocol like TCP) to notice the packet never arrived and resend it.

**Postal analogy:** regular mail. Most letters arrive fine. A small percentage get lost, damaged, or delayed. If it really matters, you buy certified mail with tracking and return receipts — at extra cost and complexity.

**Pros:**
- Universal — every vendor makes Ethernet gear, everything supports it
- Cheap — massive economies of scale
- Well-understood — every network engineer on earth knows it
- Flexible — you can mix vendors, sizes, speeds freely

**Cons:**
- **Lossy by design** — packets drop under congestion
- **Higher latency** — ~20–50 microseconds end-to-end typical
- Weak at distinguishing big "elephant flows" from small ones — under heavy load, they collide and throughput tanks

#### InfiniBand — the bonded courier service

InfiniBand (IB) was invented around 2000 specifically for **supercomputers**. It's optimized for exactly the workload where Ethernet's losses hurt most: many machines doing tightly synchronized calculations where one dropped packet stalls the whole cluster.

IB's design philosophy: **losslessness by construction.** A sender is literally not *allowed* to transmit until the receiver has explicitly said "I have buffer space for your packet, go ahead." This is called **credit-based flow control**, and it means packets fundamentally cannot be dropped due to congestion.

**Postal analogy:** think of a high-security bonded courier service. A courier won't accept your package for pickup until they've radioed ahead and confirmed the destination warehouse has a loading bay free. Because of this handshake, nothing ever gets lost in transit. But every step follows the courier's strict rules, and only couriers and warehouses trained in this system can participate.

**Key properties:**
- **Lossless** — no packet drops from congestion (bad cables can still corrupt, but that's rare)
- **~1 microsecond latency** end-to-end — ~20–50× faster than Ethernet/TCP
- **RDMA is native** (explained later) — remote memory access with zero CPU involvement
- **Centralized control** — a single "Subnet Manager" configures the whole fabric, vs Ethernet's distributed routing
- **SHARP** — switches can actually do math on packets as they pass through (sum, min, max), which halves traffic for AI's all-reduce step

**Used in:** supercomputers (weather simulation, molecular dynamics, nuclear labs) and the highest-end AI training clusters where even small efficiency losses cost millions.

#### Why won't the big cloud providers just adopt InfiniBand?

You'd think if IB is faster and more reliable, everyone would use it. They don't. Reasons:

1. **Vendor lock-in.** InfiniBand is effectively a **single-vendor** technology — NVIDIA (via Mellanox) makes essentially all of it. Hyperscalers (Meta, AWS, Google, Microsoft) have a religion about never being dependent on one supplier. Ethernet has dozens of competing vendors (Cisco, Arista, Broadcom, Marvell, plus the hyperscalers' own custom designs). They'll sacrifice raw performance to keep the ability to play vendors off each other on price and supply.

2. **Decades of Ethernet expertise.** Thousands of network engineers at every big company know Ethernet deeply. IB has a different control plane (the Subnet Manager), different debugging tools, different troubleshooting patterns. Retraining everyone is slow and expensive.

3. **Existing tooling.** Every monitoring tool, traffic analyzer, security appliance, and automation script a hyperscaler owns is built around Ethernet. IB has a parallel ecosystem but it's smaller and less mature for cloud-style operations.

4. **Multi-tenant cloud semantics.** IB was designed around "one big HPC job owns the whole fabric." Cloud providers need to run thousands of customers on shared hardware with strict isolation (VPCs, security groups, per-tenant firewalls). Ethernet has 25+ years of hardening for this; IB is catching up but isn't there.

5. **No gradual upgrade path.** Ethernet gear and IB gear don't talk directly. Converting a datacenter is a **forklift replacement** — rip out everything and start over. That's a billion-dollar decision for a hyperscaler, and there's no way to do it one rack at a time.

6. **Integration with everything else.** The rest of the datacenter (storage systems, management networks, internet ingress) runs on Ethernet. An IB fabric needs gateway boxes to translate back and forth, which adds cost and complexity.

This is exactly the gap Spectrum-X is designed to fill.

### Spectrum-X — AI-tuned Ethernet

**Spectrum-X** is NVIDIA's answer to the above: if hyperscalers won't buy InfiniBand, bring IB-class tricks to Ethernet so they can get most of the performance without giving up their Ethernet ecosystem.

It's not a single product — it's a **tightly coupled bundle**:
- **Spectrum-4 switches** (NVIDIA's top-end Ethernet switch)
- **ConnectX-8 SuperNICs** (NICs that speak Ethernet but add IB-style hardware features)
- **BlueField DPUs** (smart NICs that handle extra congestion logic)
- **Cables and optics** (LinkX)
- **Software stack** (NCCL, drivers)

The pieces only deliver the promised performance when you buy them **as a set**. You can plug a ConnectX-8 into a random Arista switch and it'll work fine as plain Ethernet, but you won't get the Spectrum-X speedup.

**What Spectrum-X adds on top of plain Ethernet:**

1. **Adaptive per-packet routing.** Normal Ethernet uses a hash of the packet header to pick one of several possible paths (**ECMP** — Equal-Cost Multi-Path). Problem: if two huge GPU-to-GPU "elephant flows" happen to hash to the same path, they collide and cripple each other. Spectrum-X switches instead **spray packets across every available path**, and the receiving NIC reorders them. This is what IB switches do.

2. **Telemetry-based congestion control.** Switches continuously report their queue depths back to the NICs in hardware, with microsecond reaction time. NICs throttle senders before drops happen.

3. **RoCE (RDMA over Converged Ethernet) tuned for near-losslessness.** Standard Ethernet is lossy; Spectrum-X's flow control makes it *effectively* lossless for the AI traffic specifically, so RDMA works well.

You get roughly **95% of InfiniBand's performance** on an Ethernet fabric — as long as you buy the whole bundle from NVIDIA.

**Mental model:**
- **InfiniBand** = purpose-built bonded courier service.
- **Regular Ethernet** = the regular postal service.
- **Spectrum-X** = the postal service with GPS trackers on every envelope, dynamic rerouting around traffic jams, and express-lane agreements at every sorting facility. It's still the postal service on paper, but upgraded hard where it matters.

### Is this all NVIDIA-only? The competitor landscape

Fair question. The honest answer: the *standards* are mostly open, but **NVIDIA dominates by volume** at both ends, and competitors are scrambling to catch up with open alternatives.

#### InfiniBand

- **The InfiniBand standard is open** — maintained by the IBTA (InfiniBand Trade Association), a multi-vendor industry body.
- **In practice, NVIDIA (via Mellanox) is ~the only volume vendor.** That's why "InfiniBand" and "NVIDIA networking" are near-synonymous in most conversations.
- **Cornelis Networks** is the credible challenger — a spin-out that revived Intel's old Omni-Path assets. Their **CN5000** (400 Gb/s) shipped in June 2025 and is available through major OEMs. Roadmap: CN6000 at 800G, CN7000 at 1.6T. Real, but tiny compared to NVIDIA's IB business.
- Net: if you want an IB-class lossless fabric today, you're almost certainly buying from NVIDIA. Cornelis gives HPC shops an escape hatch.

#### Spectrum-X (AI-optimized Ethernet)

Spectrum-X itself is a **NVIDIA-only bundle** — it's their brand name for their specific integrated stack. But the *category* (AI-tuned Ethernet with adaptive routing + hardware congestion control + near-lossless RoCE) is where the industry-wide competition is actually happening.

- **Ultra Ethernet Consortium (UEC)** — an industry-wide effort launched in 2023 to build an *open standard* for AI-optimized Ethernet. Founders: AMD, Arista, Broadcom, Cisco, HPE, Meta, Microsoft. **Notably, NVIDIA is not a founder.** **UEC 1.0 spec was ratified in June 2025** — 560 pages covering transport, congestion control, packet spraying, and link-level semantics. First UEC-compliant NIC (AMD Pollara 400) shipped to Oracle Cloud in 2025. This is the explicit "open alternative to Spectrum-X" play.
- **Broadcom** — makes the **Tomahawk 5** (51.2 Tb/s) and **Tomahawk 6** (102.4 Tb/s) switch silicon that's inside almost every non-NVIDIA AI switch on the market. Broadcom's **Jericho3-AI** is a deep-buffer spine variant. Broadcom is arguably NVIDIA's biggest competitor at the switch-silicon level.
- **Arista** — ships **Etherlink** AI switching platforms (7060X6 leaves on Tomahawk 5, 7800R4 spines on Jericho3-AI). UEC-compliant. The go-to Ethernet vendor for hyperscale AI buyers who want to avoid NVIDIA lock-in.
- **Cisco** — Nexus 9000 series with Silicon One G200 ASIC. Less AI-training mindshare than Arista/NVIDIA but a huge enterprise footprint.

#### AI NIC competitors to ConnectX/BlueField

- **AMD Pensando Pollara 400** — the first UEC-ready NIC, shipping now. **Vulcano 800G** coming in 2026 with PCIe Gen6, supporting both UEC (scale-out) and UALink (scale-up).
- **Intel** — E2200 "Mount Morgan" IPU still on the roadmap (400G+, ARM Neoverse N2 cores). Intel has been noticeably de-prioritized in AI networking.
- **Hyperscaler custom silicon** — AWS Nitro powers Trainium fabrics. Google partners with Broadcom for current TPUs and is reportedly co-designing with Marvell for future inference TPUs. Meta's MTIA uses custom networking on Broadcom silicon. Marvell has a huge custom-silicon business across 18+ cloud design wins (and has also taken NVIDIA investment to build NVLink-compatible parts — so Marvell competes with *and* complements NVIDIA).

#### Where NVIDIA still wins

The **end-to-end bundle**: GPU + NVLink (scale-up) + ConnectX/BlueField + Spectrum-X or Quantum-X + NCCL + CUDA. **NCCL in particular is the real moat** — it's the GPU-to-GPU collective communication library every major ML framework uses, tightly tuned to NVIDIA hardware. AMD's RCCL is a fork playing catch-up. UEC 1.0 standardizes the wire but it does not standardize the collective library, the GPU-to-GPU memory semantics, or CUDA.

#### Where competitors are closing in

At the **switch layer**, Broadcom TH6 has roughly matched NVIDIA on raw bandwidth and is ahead on CPO timing. Arista + Broadcom is now a credible UEC-compliant alternative for customers who don't want full NVIDIA lock-in. At the **NIC layer**, Pollara 400 proves the ecosystem can build a first-generation UEC NIC without NVIDIA. The long-term pattern: NVIDIA may lose share at the individual component level, but keeps the *bundle sale* as long as NCCL + CUDA is what AI teams build against.

### NVIDIA product lineup (early 2026)

| Name | What it is | Current generation |
|---|---|---|
| **ConnectX** | The NIC family. Dual-mode (can speak IB or Ethernet depending on config). Lives inside servers. | CX-7 = 400 Gb/s; CX-8 = 800 Gb/s, PCIe Gen6 |
| **BlueField** | **DPU** — a NIC that also has its own ARM CPUs and accelerators. Runs networking, storage, and security services offloaded from the host CPU. Think "a little server inside your server, handling infrastructure work." | BF-3 shipping today; BF-4 (800 Gb/s) launches 2026 with the Vera Rubin GPU platform |
| **Quantum** | The **InfiniBand switch** family. Standalone rack-mount boxes. | Quantum-X800 (XDR-generation IB, 800 Gb/s per port, 144 ports in a 4U box) |
| **Spectrum** | The **Ethernet switch** family. Same role as Quantum but speaks Ethernet. | Spectrum-4 (51 Tb/s switching ASIC, up to 800 Gb/s ports); Spectrum-X Photonics (with co-packaged optics) lands in 2H 2026 |
| **LinkX** | The cables and optical transceivers that tie it all together. | **DAC** = passive copper, up to ~3m; **AEC** = active copper, 3–7m; **AOC** = active optical, up to 100m; pluggable **transceivers** for longer fiber runs. 800G OSFP is standard; 1.6T ships late 2026 |

### Co-Packaged Optics (CPO)

#### What it is

Today, optical signaling in a switch works like this: the switch ASIC is in the middle of the board. Cables plug into the front panel via **pluggable transceivers** — little modules (OSFP, QSFP-DD) that each contain their own laser, modulator, and driver electronics. Every bit that leaves the ASIC has to travel several inches of copper PCB trace, get cleaned up by a power-hungry retimer, and *then* get converted to light at the front panel.

At 800 Gb/s and especially 1.6 Tb/s per lane, that copper journey is miserable — the signal fades like a radio station on a long drive, and the retimers that keep it clean burn a *lot* of watts.

**Co-packaged optics (CPO)** moves the electrical-to-optical conversion right up against the switch ASIC — same package, sometimes the same silicon substrate. Electrical traces shrink from inches to millimeters. Fiber (not copper) carries the signal the rest of the way out to the faceplate.

**Layman analogy:** it's the difference between a car engine connected to the wheels via a long, lossy driveshaft versus an in-wheel electric motor. Less mechanical loss, less energy wasted as heat, better efficiency at the extremes of performance.

#### Why the industry is moving there

Three forces, all driven by the jump to 200G- and 400G-per-lane SerDes:

1. **Power.** NVIDIA and Broadcom both claim ~3.5× lower optics power vs pluggables. On a 102.4 Tb/s switch, that's hundreds of watts saved per box — multiplied across tens of thousands of boxes in a hyperscale datacenter.
2. **Bandwidth density.** Fiber takes less faceplate area than copper, so you can fit more of it. Enables 100–400 Tb/s single-switch throughput.
3. **Signal integrity / reliability.** Shorter electrical trace = fewer retimers. NVIDIA claims 4× fewer lasers and 63× better signal integrity on their CPO platforms.

#### Does NVIDIA make CPO?

Yes. Announced at GTC 2025:

- **Quantum-X Photonics** — co-packaged-optics version of Quantum (InfiniBand). 144 × 800 Gb/s, liquid-cooled. **Commercial availability early 2026.**
- **Spectrum-X Photonics** — co-packaged-optics version of Spectrum (Ethernet). Up to 512 × 800 Gb/s (400 Tb/s). **Ships 2H 2026.**

Both are built on TSMC's **COUPE** silicon-photonics platform — so NVIDIA's CPO is NVIDIA-designed but foundry-produced at TSMC.

#### Who else is in the CPO game

| Vendor | Product | Status (Q2 2026) |
|---|---|---|
| **Broadcom** | Tomahawk 6-Davisson (BCM78919), 102.4 Tb/s | First volume CPO Ethernet switch. Announced Oct 2025, sampling to early customers, built on TSMC COUPE. |
| **NVIDIA** | Quantum-X Photonics, Spectrum-X Photonics | Quantum-X Photonics in early 2026 GA; Spectrum-X Photonics in 2H 2026 |
| **TSMC** | COUPE silicon-photonics platform | Volume production 2026 — the foundry tech under NVIDIA's and Broadcom's CPO |
| **Cisco** | Silicon One CPO demos | Still demoware as of Q2 2026; no shipping CPO switch |
| **Marvell** | 3D SiPho Engine + Celestial AI ($3.25B acquisition Dec 2025) | Sampling; meaningful revenue not expected until late 2028 |
| **Ayar Labs** | TeraPHY optical I/O chiplet + SuperNova external laser | Raised $500M March 2026; targets XPU-side CPO (on GPUs/accelerators), not switch-side |
| **Lightmatter** | Passage M1000 (3D photonic interposer, 114 Tb/s) shipping; L200 CPO | L200 expected 2026 |
| **Intel** | Silicon photonics | Largely pulled back from the switch-CPO race; roadmap quiet in 2026 |
| **Coherent, Lumentum** | External laser sources (ELS), DFB lasers, optical components | Critical supply partners to all the above — not CPO system vendors themselves |

#### What's still hard

- **Thermals.** Lasers hate heat. Switch ASICs throw a lot of it. Quantum-X Photonics is liquid-cooled for exactly this reason.
- **Serviceability.** In a pluggable world, a bad optic is a 30-second swap. In traditional CPO, a bad laser means pulling the whole switch. Industry workaround: put the laser *outside* the package as an **External Laser Source (ELS)** that's still field-replaceable.
- **No interop standard yet** for mechanical, optical, and test interfaces. Every vendor's CPO is proprietary.
- **Cost and yield.** Heterogeneous integration of photonics + electronics is harder to yield than plain silicon.

#### Realistic timeline

CPO is shipping but **not yet at scale** in 2026. Broadcom's TH6-Davisson is the first real volume CPO Ethernet switch; NVIDIA's CPO parts are entering early production through 2026. **Pluggable 1.6T optics will continue to dominate through 2026**, with CPO taking over at 3.2T lane rates around 2027–2028.

### Software on networking hardware — what's actually running on this stuff?

Switches, NICs, and DPUs aren't dumb silicon. Each is a real computer with real software on it. This section walks through the operating systems, firmware, and SDKs that make the hardware do anything useful.

#### The two halves of a switch: data plane vs control plane

A modern switch is really two things bolted together in one chassis:

1. **The switching ASIC** — the big custom chip that moves packets at terabits per second. Purpose-built, extremely fast, but doesn't "think."
2. **A management CPU** — a small x86 or ARM processor with a few GB of RAM, running a full Linux-based operating system. Slow compared to the ASIC, but smart. Handles configuration, talks to other switches, monitors health, and programs rules into the ASIC.

**Postal analogy:** go back to the regional post office sorting facility:
- The **conveyor belts and sorting arms** fling letters into bins at blinding speed. They're fast and mechanical, but they don't *decide* anything — they follow whatever routing cards are slotted into the machine.
- The **postmaster** in a back office reads the news, talks to other post offices by phone, updates the routing cards when a bridge closes or a new zip code is added. Handles almost no mail personally.

In networking terms:
- **Data plane** = the sorting belts (the ASIC). Forwards packets based on lookup tables.
- **Control plane** = the postmaster (the management CPU + its OS). Runs routing protocols, serves the CLI/API, talks to orchestration systems, updates the ASIC's lookup tables.

**The NOS (Network Operating System)** is the software running on the control plane. When people say "what OS does this switch run," they mean the control plane's Linux distribution + network-management stack.

#### SAI — the "standard plug" that makes vendor-independent NOSes possible

Every ASIC vendor (NVIDIA for Spectrum, Broadcom for Tomahawk, Marvell, Cisco, etc.) has its own quirky chip-programming SDK. That used to mean every NOS had to be custom-ported to every chip — painful.

**SAI (Switch Abstraction Interface)** is a standardized C-language API that sits on top of all those vendor SDKs and exposes one common "set this VLAN, add this route, install this ACL" interface. A NOS that speaks SAI can run on any switch whose chip vendor provides a SAI shim.

**Layman analogy:** SAI is the standard **wall outlet shape**. As long as a lamp (the NOS) has the right plug, it works everywhere — doesn't matter which power company (ASIC vendor) provides the electricity.

SAI is governed by the Open Compute Project (OCP). NVIDIA contributes the SAI implementation for Spectrum.

#### SONiC — open-source NOS, originally from Microsoft Azure

**SONiC** (Software for Open Networking in the Cloud) is an **open-source Linux-based network operating system**. It was built inside Microsoft Azure around 2016. Azure operates at a scale where buying locked-in switch+OS bundles from traditional vendors (Cisco, Arista) made neither economic nor strategic sense — any feature they wanted was on someone else's roadmap, not theirs. So Microsoft built their own NOS, open-sourced it, and in April 2022 handed governance to the Linux Foundation.

**How SONiC is built:** Debian Linux at the base. Every network function (BGP routing, LLDP neighbor discovery, the SAI-to-ASIC bridge called `syncd`, etc.) is its own Linux container. Configuration state lives in a shared Redis database. A config change is a database write; the relevant container notices and pushes new rules down to the ASIC via SAI. Very modular.

**Production users:** Microsoft Azure (their entire AI-datacenter fabric), Alibaba Cloud (~100K switches across 28 regions), Tencent, Baidu, Meta. Gartner projected 40%+ of large datacenter operators would be running SONiC by 2026 — and that's on track.

**NVIDIA and SONiC:** NVIDIA sells a product called **Pure SONiC** — literally the community upstream, unmodified, but with NVIDIA's QA, release engineering, documentation, training, and enterprise support wrapped around it. The "Pure" is a marketing jab at competitors (Dell's Enterprise SONiC, Broadcom's version, Aviz) that ship forked or proprietary-extended builds. Every Spectrum Ethernet switch is a first-class SONiC target.

#### Cumulus Linux — NVIDIA's in-house Ethernet NOS

**Cumulus Linux** is a Debian-based NOS that NVIDIA inherited when it acquired Cumulus Networks in 2020 (a year after the Mellanox acquisition). It predates SONiC by several years. The original pitch: "your switch feels like a Linux server." Standard Linux tooling (`ip`, `systemd`, `ifupdown2`) instead of Cisco-style IOS. Engineers who could manage server fleets could manage switch fleets without learning a whole new CLI language.

Post-acquisition NVIDIA narrowed Cumulus to run *only* on NVIDIA Spectrum switches. It's actively developed — v5.10+ is in production, v5.15 in development as of early 2026. The 4.x branch reached end-of-life in May 2023.

**Cumulus vs SONiC in one sentence:** both are Debian-based, SAI-driven, run on Spectrum. **Cumulus is NVIDIA-owned, turnkey, opinionated, comes with support.** **SONiC is community-governed, modular, hyperscaler-style.** NVIDIA sells both — enterprises tend to pick Cumulus; hyperscalers pick Pure SONiC.

#### The InfiniBand side — MLNX-OS, Onyx, and NVOS

InfiniBand switches don't run Cumulus or SONiC. They run different OSes:

- **MLNX-OS** — the original InfiniBand switch OS, going back to the Mellanox Voltaire era. Still actively maintained (v3.12.6200 LTS) on Quantum QM9700/9790 platforms.
- **Onyx** — formerly "MLNX-OS Ethernet," rebranded around 2017. The Ethernet NOS for pre-Spectrum-3 Mellanox switches. Legacy only — no new investment after Cumulus/SONiC took over Ethernet.
- **NVOS** (NVIDIA Network Operating System) — the newer unified OS. Ships on the current **Quantum-X800 XDR InfiniBand switches**. Configured *exclusively* via NVUE (no Cisco-style `configure terminal` CLI) — it's a YAML/JSON configuration model with a REST API underneath. Intended to eventually absorb the Ethernet side too.

#### NVUE — NVIDIA's unified configuration layer

**NVUE (NVIDIA User Experience)** is NVIDIA's effort to give every NVIDIA networking product — Cumulus, NVOS, even host-side networking — a single YAML/CLI config model. Regardless of what switch or NIC you're touching, the config feels the same. It's the convergence play that's gradually rolling out across the portfolio.

#### What runs on ConnectX NICs

A ConnectX NIC isn't a dumb pipe. Its firmware (tens of megabytes, runs on an onboard microcontroller) implements a ton of offloads in hardware-assisted fashion:

- RDMA, RoCE, GPUDirect
- VXLAN encap/decap (for overlay networks)
- SR-IOV (hardware-level virtualization — one physical NIC looks like many to the OS)
- Flow steering, ACL filtering
- NVMe-over-Fabrics target offload
- Congestion control algorithms

ConnectX-8 goes further — it has an integrated PCIe Gen6 switch on the card itself, useful for GPU-dense topologies where the NIC fans out PCIe lanes to multiple GPUs.

You flash firmware with `mlxfwmanager`; you don't typically write code for it yourself — NVIDIA ships it, you just update it. The host driver (`mlx5` in mainline Linux, or NVIDIA's full **MLNX_OFED** distribution) exposes the NIC as a standard network device plus a bunch of knobs via `devlink` and `ethtool`.

#### BlueField DPU — a real Linux computer on a PCIe card

A BlueField DPU is a ConnectX NIC + a full ARM computer + DRAM + PCIe switch, all on one card:

- **BlueField-3** — 16 ARM Cortex-A78 cores, 32 GB DRAM, ConnectX-7 at 400 Gb/s. Shipping broadly today.
- **BlueField-4** — announced CES 2026, shipping 2H 2026 with Vera Rubin. 64-core Grace CPU die + ConnectX-9 at 800 Gb/s. Roughly 6× the compute and 2× the bandwidth of BF-3, plus a new secure root-of-trust subsystem called ASTRA.

Those ARM cores **run a real Linux distribution**, not firmware. NVIDIA's reference OS is **Ubuntu**, distributed as a **BFB** (BlueField Boot stream) file, installed via the **BSP (BlueField Board Support Package)** — current v4.14.x as of Q1 2026. Flash a BFB → bootloader + kernel + drivers + DOCA runtime get laid down → you SSH in and run `apt install` like any other Linux box. Customers can replace Ubuntu with RHEL, SUSE, or custom in-house distros.

#### DOCA — the DPU programming platform

**DOCA (Data-Center-on-a-Chip Architecture)** is to DPUs what CUDA is to GPUs — NVIDIA's SDK and runtime for writing apps that run on BlueField.

Ships as three meta-packages:
- `doca-runtime` — runs on every DPU
- `doca-tools` — CLI utilities
- `doca-sdk` — headers and libraries for developers

Libraries cover: flow programming (`doca_flow`), crypto acceleration, regex matching, packet processing, GPUNetIO (GPU-driven networking), and service orchestration. A typical DOCA app is a containerized Linux service using these libraries to offload something the host CPU used to do.

**Real-world DOCA apps people have actually built:**
- Palo Alto Networks' cloud firewall running on-DPU
- VMware NSX data-plane offload
- Red Hat OpenShift's networking layer
- Storage vendors' NVMe-oF targets
- Cisco Hypershield zero-trust agents
- Various observability and telemetry collectors

#### DPF — Kubernetes for DPU fleets

**DPF (DOCA Platform Framework)** is the orchestration layer above DOCA. Think of it as a Kubernetes control plane whose "nodes" are BlueField cards instead of servers. It handles:

- Fleet-wide DPU provisioning
- DOCA service lifecycle management (deploy, upgrade, roll back apps across thousands of DPUs)
- Firmware updates
- Zero-touch rollout

Current release is v25.07 (NVIDIA uses year.month versioning, so that's the July 2025 line). Red Hat has a productized DPF integration for OpenShift.

#### NVIDIA networking software stack at a glance

| Hardware | Software / OS | Notes |
|---|---|---|
| **Spectrum Ethernet switches** | **Cumulus Linux** (NVIDIA-owned, turnkey) **or** **Pure SONiC** (open-source, community) | Same hardware; customer picks. Cumulus 5.x / SONiC 202x. |
| **Quantum-X800 InfiniBand (current XDR)** | **NVOS** | NVUE/YAML/REST only. No Cisco-style CLI. |
| **Older Quantum IB (QM97xx)** | **MLNX-OS** | Legacy but still actively maintained LTS. |
| **Legacy pre-Spectrum-3 Ethernet** | **Onyx** | Legacy only; no new investment. |
| **ConnectX NICs** | NIC firmware + `mlx5` driver / MLNX_OFED on host | Flash via `mlxfwmanager`. |
| **BlueField DPUs** | **Ubuntu** (BFB via BSP) on ARM + **DOCA** SDK/runtime | Real full Linux, not firmware. |
| **BlueField fleets at scale** | **DPF** | Kubernetes-style orchestration. |
| **Cross-cutting: ASIC abstraction** | **SAI** | What NOSes program switch chips through. |
| **Cross-cutting: unified config** | **NVUE** | YAML/CLI consistency across Cumulus, NVOS, host. |

### Why networking is a huge deal at NVIDIA

Here's the business and technical reason networking matters so much — and why NVIDIA spent $6.9 billion acquiring Mellanox in 2019.

**The AI training traffic pattern:**

A modern AI model is too big to train on one GPU. It gets **sharded** across thousands of GPUs that work in parallel. On every single training step:

1. Each GPU computes its piece (gradients for its chunk of the model) — takes ~tens of milliseconds
2. Every GPU must then **exchange** its gradients with every other GPU so they all agree on the updated weights — this is called an **all-reduce**
3. Only after the all-reduce completes can the next training step begin

This all-reduce step is pure networking. And it's brutal:
- **Every** GPU talks to **every** other GPU
- All GPUs wait at a **barrier** until the slowest link finishes
- If one link in a 10,000-GPU cluster is slow or dropping packets, all 10,000 GPUs are idle waiting for it

This traffic is called **east-west** — server talking to server, inside the datacenter — as opposed to **north-south** (datacenter talking to the outside internet). In AI clusters, east-west traffic dwarfs north-south by orders of magnitude.

**The money math:**

- A single H100 GPU costs ~$30,000; a B200 costs more. A 10,000-GPU cluster is ~$300M+ in GPUs alone.
- That cluster's electricity bill is ~$1M/month.
- The difference between a fabric at 90% vs 95% efficiency is the difference between a 10-day and a 15-day training run.
- **50% longer training = 50% more money wasted on idle GPUs and electricity.**

Which is why: **GPUs are expensive, idle GPUs are wasted money, and the network decides how much time GPUs spend idle.**

**The acceleration tech stack:**

To keep networking from being the bottleneck, NVIDIA (and the industry) layered in a bunch of tricks:

- **RDMA (Remote Direct Memory Access).** Normally, when Server A sends data to Server B, the operating system gets involved: a copy from app memory to kernel memory, then to the NIC, then back up the stack on the other end. That takes ~50 microseconds per packet and burns CPU time. RDMA lets the **NIC read/write remote memory directly**, bypassing both OSes and both CPUs. Latency drops to ~1 microsecond.

- **GPUDirect RDMA.** Goes further — the NIC talks **directly to GPU memory (VRAM)** over PCIe, skipping host RAM entirely. The path is literally: `GPU A's memory → NIC A → wire → NIC B → GPU B's memory`. No CPU touches the data at any point. This is critical because training data is already in GPU memory, and copying it out to host RAM and back is wasted time.

- **NCCL (NVIDIA Collective Communications Library).** The software layer that orchestrates all of the above. When PyTorch calls `all_reduce()`, NCCL figures out the topology (which GPUs share a machine, which are across the network), picks an efficient communication pattern (ring, tree), and issues the right GPUDirect RDMA operations. If NCCL can't use GPUDirect (e.g. the `nvidia-peermem` kernel module isn't loaded), it silently falls back to slow host-memory staging — 3–5× slower. This is a common real-world failure mode worth knowing about.

**Why lossless matters for AI specifically:**

In an all-reduce across 10,000 GPUs, every GPU is waiting at a barrier. A single dropped packet means **the sender has to notice, retransmit, and the receiver has to wait for the retry** — while every other GPU in the cluster sits idle at the barrier. Even a 0.001% packet loss rate can cut throughput 10–30%. This is why InfiniBand exists and why Spectrum-X goes to such lengths to approximate losslessness over Ethernet.

---

## Glossary (quick reference)

- **NIC** — Network Interface Card. The endpoint hardware that lets one computer talk on the network.
- **DPU** — Data Processing Unit. A NIC that also has CPUs and accelerators for offloaded infrastructure work.
- **Switch** — Multi-port packet forwarder. A rack appliance that routes packets between many NICs.
- **ASIC** — Application-Specific Integrated Circuit. A fixed-function chip. The brain of a NIC or switch.
- **Port** — a physical socket on a NIC or switch where a cable plugs in.
- **Packet** — a chunk of data wrapped with address headers. The atomic unit of network traffic.
- **MAC address** — factory-assigned hardware address on a NIC. Like a specific house's street address.
- **IP address** — network-assigned address. Like the name of the resident currently living in the house.
- **Bandwidth** — max data rate (Gb/s, Tb/s). Width of the road.
- **Latency** — time for one packet to travel A→B (µs, ms). Length of the drive.
- **Protocol** — the rulebook two devices agree on so they can understand each other.
- **Ethernet** — the universal best-effort networking standard. Used by everything.
- **InfiniBand (IB)** — lossless HPC-grade networking standard. Competes with Ethernet.
- **RDMA** — Remote Direct Memory Access. NIC-to-NIC memory access with zero CPU or OS involvement.
- **RoCE** — RDMA over Converged Ethernet. RDMA bolted onto Ethernet.
- **GPUDirect RDMA** — NIC reads/writes GPU VRAM directly, skipping host RAM entirely.
- **NCCL** — NVIDIA Collective Communications Library. Orchestrates GPU-to-GPU collectives like all-reduce.
- **NVLink** — NVIDIA's *intra-node* GPU-to-GPU fabric (GPUs inside the same server). Separate from NIC-based networking between servers.
- **all-reduce** — Collective operation where every participant ends up with the sum (or other reduction) of everyone's inputs. The dominant traffic pattern in distributed AI training.
- **east-west traffic** — Server-to-server traffic inside a datacenter.
- **north-south traffic** — Datacenter-to-internet traffic.
- **leaf-spine** — Two-tier datacenter topology: ToR ("leaf") switches aggregate racks, spine switches aggregate the leaves.
- **ToR** — Top-of-Rack switch. The first-hop switch every server in a rack connects to.
- **SHARP** — Scalable Hierarchical Aggregation and Reduction Protocol. Lets IB switches do in-network math (sum, min, max).
- **Subnet Manager** — the centralized controller that configures an InfiniBand fabric.
- **ECMP** — Equal-Cost Multi-Path. Classic Ethernet load-balancing via hashing. Causes flow collisions under AI workloads.
- **Elephant flow** — a long-lived, high-bandwidth data stream (e.g. GPU-to-GPU gradient transfer). Opposite of "mice flows" — many small short bursts.
- **Forklift replacement** — industry slang for an upgrade that can't be done incrementally; you rip out everything and start over.
- **DAC / AEC / AOC** — Direct Attach Copper / Active Electrical Cable / Active Optical Cable. Short-to-medium cable types.
- **NOS** — Network Operating System. The software running on a switch's management CPU.
- **Data plane** — the part of a switch that actually forwards packets (the ASIC).
- **Control plane** — the part of a switch that makes decisions (management CPU + NOS).
- **SAI** — Switch Abstraction Interface. A standard C API that lets a NOS program any vendor's ASIC. The "standard wall outlet" of network switches.
- **SONiC** — open-source Linux-based NOS originally from Microsoft Azure, now governed by the Linux Foundation. Heavy hyperscaler adoption.
- **Pure SONiC** — NVIDIA's commercial distribution of upstream SONiC, with support.
- **Cumulus Linux** — NVIDIA's in-house Debian-based NOS for Spectrum Ethernet switches. Acquired 2020.
- **Onyx** — legacy Ethernet NOS for pre-Spectrum-3 Mellanox switches. De-emphasized post-acquisition.
- **MLNX-OS** — legacy but still actively maintained InfiniBand switch OS (Quantum QM9700/9790).
- **NVOS** — NVIDIA Network Operating System. The newer unified OS for Quantum-X800 XDR IB switches. NVUE-only.
- **NVUE** — NVIDIA User Experience. Unified YAML/CLI/REST config model across Cumulus, NVOS, and host networking.
- **DOCA** — Data-Center-on-a-Chip Architecture. NVIDIA's SDK + runtime for BlueField DPUs. The DPU equivalent of CUDA.
- **DPF** — DOCA Platform Framework. Kubernetes-style orchestration for fleets of BlueField DPUs.
- **BSP / BFB** — BlueField Board Support Package / BlueField Boot stream. The installer package and boot image for Ubuntu-on-DPU.
- **MLNX_OFED** — NVIDIA's full-featured Linux driver distribution for ConnectX NICs. The mainline kernel has `mlx5` as a slimmer upstream alternative.
- **RoCE** — RDMA over Converged Ethernet. (Already listed above — included here for software-stack context.)
- **SmartNIC** — a NIC with programmable compute onboard (ARM cores, FPGA, P4 engines). More than a dumb NIC, less than a DPU.
- **Host CPU** — the main server motherboard CPU (the big x86 or Grace CPU running the application). Distinct from the smaller CPUs on NICs, DPUs, or switches.
- **Uplink / downlink** — uplink ports go *up* to the next tier (leaf→spine), downlink ports go *down* to the tier below (leaf→servers).
- **Oversubscription ratio** — the ratio of downlink bandwidth to uplink bandwidth on a switch. AI fabrics typically run 1:1 (non-oversubscribed).
- **CLOS** — a multi-tier switching topology (named after Charles Clos). Leaf-spine and leaf-spine-super-spine are CLOS topologies.
- **Super-spine / core switch** — third-tier aggregation switches sitting above spines in very large fabrics (100k+ GPUs).
- **DCI** — Data Center Interconnect. Gear and links for connecting geographically separated datacenters.
- **Spectrum-XGS** — NVIDIA's DCI-tier product for "connect multiple datacenters into one logical AI factory."
- **DHCP** — Dynamic Host Configuration Protocol. The service that hands out IP addresses to devices when they join a network.
- **CPO (Co-Packaged Optics)** — moves the optical-to-electrical conversion into the same chip package as the switch ASIC. Big power/density/signal-integrity wins vs traditional pluggable optics at 800G+.
- **Pluggable transceiver (OSFP, QSFP-DD)** — the older/current way to do optics — a removable module that plugs into the front panel of a switch.
- **ELS (External Laser Source)** — in CPO designs, the laser is sometimes kept outside the package so it stays field-replaceable.
- **TSMC COUPE** — the TSMC foundry's silicon-photonics platform that underlies both NVIDIA's and Broadcom's CPO products.
- **SerDes** — Serializer/Deserializer. The circuit that converts parallel data to/from high-speed serial on the wire. Generation name (e.g. "200G PAM4 SerDes") drives what bandwidth a NIC/switch port can hit.
- **UEC (Ultra Ethernet Consortium)** — industry effort (AMD, Broadcom, Arista, Cisco, HPE, Meta, Microsoft) to standardize AI-optimized Ethernet. UEC 1.0 ratified June 2025. The open-standard alternative to NVIDIA's Spectrum-X.
- **Cornelis Networks** — company reviving Intel's old Omni-Path as a credible alternative to NVIDIA InfiniBand. CN5000 shipping at 400G.
- **Pollara / Vulcano** — AMD Pensando's UEC-compliant AI NICs.
- **IBTA** — InfiniBand Trade Association. The standards body that owns the IB spec.
- **Physical port** vs **TCP/UDP port** — don't confuse them. Physical port = the socket where a cable plugs into a NIC or switch. TCP/UDP port = a number in the packet header used to route to the right application on the receiving host. Same word, totally different concept.

---

## Q&A Log

New questions land here, dated. Content gets promoted up into Fundamentals when a topic matures.

### 2026-04-20 — Why won't hyperscalers adopt InfiniBand?

**Q:** The doc said "hyperscalers want IB-class AI performance but won't adopt IB." Why won't they adopt it?

**A:** Six reasons, in rough order of importance:

1. **Vendor lock-in.** IB is effectively single-vendor (NVIDIA via Mellanox). Ethernet has dozens of vendors. Hyperscalers refuse single-vendor dependencies as a matter of corporate religion.
2. **Decades of Ethernet expertise.** Thousands of network engineers already know Ethernet deeply; retraining is slow and costly.
3. **Existing tooling.** All monitoring, telemetry, security, and automation is Ethernet-based. IB has a parallel but smaller ecosystem.
4. **Multi-tenant cloud semantics.** IB was built for "one big HPC job owns the fabric." Clouds need strict tenant isolation at scale; Ethernet has 25+ years of hardening for this.
5. **No gradual upgrade path.** You can't mix IB and Ethernet directly — it's a forklift replacement, not an incremental rollout.
6. **Everything else runs on Ethernet.** Storage, management, and internet ingress are all Ethernet. An IB fabric needs gateway boxes to bridge back to the rest of the datacenter.

This is why Spectrum-X exists — it gives hyperscalers ~95% of IB's performance without forcing them to abandon their Ethernet ecosystem. *(Promoted into Fundamentals → Spectrum-X section.)*

### 2026-04-20 — What software/OS/firmware runs on all this hardware? And what is SONiC?

**Q:** I've heard of SONiC, some OS that seems to run on a NIC or switch. What is it, and what else runs on this networking hardware?

**A:** The short version: every piece of networking hardware is actually a real computer, with real software on it.

- **Switches** have two halves: the **data plane** (the ASIC — forwards packets at terabit speeds, "dumb but fast") and the **control plane** (a small management CPU running a full Linux-based **NOS** — Network Operating System). The NOS is what people mean when they ask "what OS does this switch run."
- **SONiC** (Software for Open Networking in the Cloud) is an open-source Linux-based NOS originally built inside Microsoft Azure, now governed by the Linux Foundation (since 2022). Used in production by Microsoft, Alibaba (~100K switches), Tencent, Baidu, Meta. Debian at the base, with every network function as a container and config stored in a Redis database. NVIDIA ships a supported distribution called **Pure SONiC** that runs on every Spectrum switch.
- **Cumulus Linux** is NVIDIA's in-house NOS (acquired 2020). Also Debian-based, but NVIDIA-owned and turnkey. Runs on Spectrum too. Customers pick Cumulus for "works out of the box with support" or SONiC for "hyperscaler-style modularity."
- **On the InfiniBand side**: Quantum-X800 switches run **NVOS**, NVIDIA's new unified OS. Older Quantum switches run **MLNX-OS**. Legacy Ethernet Mellanox switches ran **Onyx**, now de-emphasized.
- **SAI** is the standard API that makes all this possible — it's the "wall-outlet shape" that lets a NOS program any vendor's ASIC chip.
- **NICs** (ConnectX) run firmware — tens of MB of hardware-offload code (RDMA, RoCE, GPUDirect, SR-IOV, VXLAN, etc.). You flash it, you don't really write code for it.
- **DPUs** (BlueField) are different — they run **full Ubuntu Linux** on their ARM cores, installed via a BFB file from the BSP (Board Support Package). You SSH in and `apt install` like a normal server.
- **DOCA** is NVIDIA's SDK + runtime for writing DPU apps — the DPU equivalent of CUDA. Real customers (Palo Alto, VMware, Red Hat, Cisco, storage vendors) ship DOCA apps today.
- **DPF** (DOCA Platform Framework) is the Kubernetes-style orchestrator for fleets of DPUs.
- **NVUE** is NVIDIA's unified config layer — YAML/REST/CLI — spanning Cumulus, NVOS, and host networking so everything configs the same way.

*(Full detail + summary table promoted into Fundamentals → "Software on networking hardware" section.)*

### 2026-04-20 — Batch of foundational questions

**Q1:** How can a MAC address never change but an IP address can? My laptop always has the same MAC but its IP changes depending on the network.

**A1:** MAC = the **serial number of the physical NIC chip**, burned in at the factory. It never changes. IP = the **phone number your current network carrier assigned you**, only meaningful while you're on that network. Plug your laptop into home Wi-Fi, you get `192.168.1.42`. Take it to a cafe, a different network hands you `10.0.0.73`. Same laptop, same MAC, different IP each time. The IP is assigned by a DHCP server on whichever network you've joined. *(Clarified in Fundamentals → Basic vocabulary.)*

**Q2:** What is CPO? Does NVIDIA make it? Other companies?

**A2:** **Co-Packaged Optics** — puts the electrical→optical conversion in the same chip package as the switch ASIC, instead of at the front-panel pluggable module. Huge power and signal-integrity wins at 800G+. **Yes, NVIDIA makes CPO** (Quantum-X Photonics for IB, early 2026 GA; Spectrum-X Photonics for Ethernet, 2H 2026) — both built on TSMC's COUPE platform. **Other players**: Broadcom (Tomahawk 6-Davisson is the first volume CPO Ethernet switch, shipping 2025-2026), Cisco (demos only), Marvell (sampling, revenue 2028+), Ayar Labs (chip-side CPO, not switch-side), Lightmatter, Intel (largely exited), Coherent/Lumentum (laser component suppliers). Pluggable optics still dominate 2026; CPO really takes over at 3.2T lane rates in 2027-2028. *(Full section promoted into Fundamentals → CPO.)*

**Q3:** What are the different ports on a NIC used for? Are NIC ports the same as computer ports like 22 or 8888?

**A3:** **Completely different concepts** that unfortunately share the word "port":
- **Physical NIC ports** = literal holes where cables plug in. A NIC typically has 1 or 2 (sometimes 4). Used for redundancy (two paths to two switches) or bandwidth (both active at once).
- **TCP/UDP ports** (like port 22, 80, 8888) = numbers in the packet header that tell the receiving OS which *application* should get the packet. If the IP is the street address, the port is the apartment number. Port 22 = SSH, 80 = HTTP, 443 = HTTPS, 8888 = Jupyter. Nothing to do with physical plugs.

*(Clarified in Fundamentals → Basic vocabulary and NIC section.)*

**Q4:** The NIC has its own MAC. Does the server's CPU have its own MAC? IP?

**A4:** **No.** MAC and IP addresses belong to **network interfaces (NICs)**, not to CPUs. When you say "my server has IP 10.0.0.5," you really mean "the NIC in my server has IP 10.0.0.5." A server with two NICs has two MACs and typically two IPs. The CPU just talks to the NIC over PCIe and asks it to send/receive. *(Clarified in Fundamentals → Basic vocabulary.)*

**Q5:** Each NIC cables into a switch? So switches have 20, 30, 40+ ports?

**A5:** Yes. Typical switch port counts:
- Small top-of-rack: 32–48 ports
- Typical AI leaf: 64 ports at 400–800 Gb/s
- High-end spine: 128 ports
- NVIDIA Quantum-X800: **144 × 800 Gb/s ports** in a 4U box

**Q6:** So a switch just reroutes packets from one NIC to another — basically an expander for getting servers to talk to each other. Is that correct? Does a switch need an OS, or is it pure hardware?

**A6:** The *forwarding* is done in dedicated ASIC hardware, so yes — a switch is conceptually a fast programmable packet-forwarder. **But there IS a real OS on every modern switch** — it runs on a separate management CPU alongside the ASIC. That OS (the NOS — Cumulus, SONiC, NVOS, etc.) doesn't handle packets in the hot path, but it:
- Takes configuration (which ports, which VLANs, which ACLs)
- Runs routing protocols (BGP, OSPF) to learn the fabric
- Monitors health, handles alarms, pushes telemetry
- Programs the forwarding rules into the ASIC

Without the NOS, the switch would be a dumb "everything goes everywhere" box. With it, you can do real network engineering. *(Clarified in Fundamentals → What is a Switch + Software on networking hardware.)*

**Q7:** Does a leaf switch only talk to the NICs in its rack? So it only needs 8–12 ports?

**A7:** A leaf has **two groups of ports**:
- **Downlink ports** go to servers in its rack (usually 32–48 of them)
- **Uplink ports** go *up* to multiple spine switches (usually 8–16 of them)

So a typical leaf is 48 down + 16 up = 64 total ports. Not just 8–12. *(Expanded in Fundamentals → Putting it in a datacenter.)*

**Q8:** How many spine switches can talk to how many leaf switches? Can one spine talk to 50 leaves?

**A8:** As many leaves as the spine has ports. A Quantum-X800 (144 ports) can theoretically serve 144 leaves with one uplink each — in practice you want multiple uplinks per leaf for redundancy, so the ratio is lower. A typical modern AI datacenter uses **4–16 spine switches**, each connected to every leaf. Example: 32 racks → 32 leaves → each leaf uplinks to 16 spines → you need 16 spine switches. *(Expanded in Fundamentals → Putting it in a datacenter.)*

**Q9:** Is there anything higher than a spine switch?

**A9:** Yes — for very large fabrics you add a third tier called **super-spine** (or "core"). Used at 100k+ GPU scale. Structure: **Leaf → Spine → Super-spine**. Beyond that, connecting multiple datacenters into one logical fabric uses **DCI** gear (NVIDIA's **Spectrum-XGS** is their DCI play). Full hierarchy: DCI → super-spine → spine → leaf → NIC → server. *(Expanded in Fundamentals → Putting it in a datacenter.)*

**Q10:** What's a SmartNIC? How is it different from a "dumb" NIC?

**A10:** Three tiers:
- **Basic NIC ("dumb")** — moves packets between wire and host memory. Firmware-based offloads (checksum, RSS, basic RDMA).
- **SmartNIC** — NIC with real programmable compute (ARM cores, FPGA, P4 engines). Can run customer packet processing, custom protocols, encryption.
- **DPU** (NVIDIA's BlueField) — a SmartNIC powerful enough to be a **full tiny server**: multi-core ARM CPU complex, DRAM, PCIe switch, running a real Linux distribution. You SSH into it.

*(Promoted into Fundamentals → Dumb NIC vs SmartNIC vs DPU.)*

**Q11:** Is Spectrum-X NVIDIA-only? InfiniBand? What do other vendors have?

**A11:** 
- **InfiniBand**: standard is open (IBTA), but in volume NVIDIA is effectively the only vendor. **Cornelis Networks** (Omni-Path revival) is the credible alternative — CN5000 at 400G shipping now.
- **Spectrum-X** as a branded bundle: NVIDIA-only. But the *category* (AI-optimized Ethernet) has real competition via the **Ultra Ethernet Consortium (UEC)** — AMD, Broadcom, Arista, Cisco, HPE, Meta, Microsoft (notably *not* NVIDIA). **UEC 1.0 was ratified in June 2025**. First UEC NIC: AMD's Pollara 400. Switch silicon: Broadcom Tomahawk 5/6. Complete alternative stacks (Arista + Broadcom + AMD Pollara) exist today for customers who want to avoid NVIDIA lock-in. *(Promoted into Fundamentals → Is this all NVIDIA-only?)*

**Q12:** "Offloaded from the host CPU" — host CPU = main server CPU or CPU on the NIC?

**A12:** **Main server CPU** — the big x86 or Grace CPU on the motherboard, the one running the application. "Offloaded from the host CPU" means "the DPU does this work instead of burning host cycles on it." *(Added to Basic vocabulary and DPU section; glossary now has "Host CPU" entry.)*

**Q13:** Data plane = the fast ASIC, control plane = the CPU/management. Correct?

**A13:** Exactly. Data plane = what touches packets (the ASIC, nanosecond-speed). Control plane = what makes decisions and programs the data plane (the management CPU + NOS). *(Covered in Fundamentals → The two halves of a switch.)*

**Q14:** What does SONiC stand for?

**A14:** **S**oftware for **O**pen **N**etworking **i**n the **C**loud. Built inside Microsoft Azure around 2016, open-sourced, governance moved to the Linux Foundation in 2022.

---

## What's still missing from this doc (future learning topics)

You now have a solid picture of: hardware (NICs, DPUs, switches), the leaf-spine topology, Ethernet vs InfiniBand vs Spectrum-X, NVIDIA's product lineup and the competitor landscape, CPO, and the software stack (NOSes, DOCA, etc.). A hardware-and-firmware engineer reading this has a strong *high-level* picture.

Things that would round out the picture, listed roughly in order of "next thing to learn":

1. **The OSI model / layers** — the conceptual 7-layer (or simpler 4-layer TCP/IP) model. L1 = physical (SerDes, cables), L2 = link (Ethernet, MAC addresses), L3 = network (IP), L4 = transport (TCP/UDP), L7 = application. Maps every concept in this doc into a stack position.
2. **Routing vs switching** — switches operate at L2 (MAC), routers at L3 (IP). What's the actual difference, and why do both exist?
3. **TCP vs UDP** — TCP is reliable-stream (retransmits lost packets, ordered delivery); UDP is fire-and-forget. Why each exists and when to pick one.
4. **DNS** — how human names like `github.com` become IP addresses. The whole name resolution layer.
5. **NAT, firewalls, and gateways** — the L3+ boxes that sit between networks and rewrite/filter traffic.
6. **VLANs and VXLAN** — how one physical network gets carved into many logical isolated networks. Essential for multi-tenant cloud.
7. **Security basics** — TLS (encryption at L7), IPsec (at L3), MACsec (at L2). What gets encrypted where.
8. **Congestion control in depth** — ECN, DCQCN, PFC, and why they matter for RoCE.
9. **Telemetry and observability** — SNMP, sFlow, NetFlow, INT (In-band Network Telemetry), gNMI.
10. **NVLink vs NIC networking** — NVLink is the *intra-node* GPU-to-GPU fabric (GPUs inside a single server). NICs + switches are *inter-node* (between servers). Why both exist and how they interact. NVSwitch, NVLink Switch System.
11. **Physical media in depth** — copper vs fiber, single-mode vs multi-mode, SerDes PAM4 vs PAM6, what limits reach.
12. **The economics** — optics is 40–50% of datacenter networking cost. Why optical is so expensive, and how CPO changes that.

Ask about any of these and I'll add sections.

<!-- Template:

### YYYY-MM-DD — <short question>

**Q:** <the question as asked>

**A:** <answer>

-->
