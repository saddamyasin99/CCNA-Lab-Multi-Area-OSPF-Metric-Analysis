# CCNA-Lab-Multi-Area-OSPF-Metric-Analysis
Multi area OSPF lab using interface level configuration (ip ospf area) across a 3-router topology with cost/metric verification,  built in Cisco Packet Tracer as part of my CCNA self study series.

**Topic:** How OSPF calculates cost from interface bandwidth, and what happens when bandwidth or cost is changed
**Topology:** 3-router, multi-area OSPF (Router-1 ↔ Router-2 ↔ Router-3) — OSPF process 50

## 1. Topology Summary

| Router | Interface | Network | Area | Bandwidth |
|---|---|---|---|---|
| Router-1 | Gig0/0/0 | 192.168.10.0/24 | 10 | 100,000 Kbit |
| Router-1 | Se0/1/0 | 12.0.0.0/8 | 10 | 1,544 Kbit |
| Router-2 | Gig0/0/0 | 192.168.20.0/24 | 0 | 100,000 Kbit |
| Router-2 | Se0/1/0 | 12.0.0.0/8 | 10 | 1,544 Kbit |
| Router-2 | Se0/1/1 | 23.0.0.0/8 | 20 | 1,544 Kbit |
| Router-3 | Gig0/0/0 | 192.168.30.0/24 | 20 | 100,000 Kbit |
| Router-3 | Se0/1/1 | 23.0.0.0/8 | 20 | 1,544 Kbit |

Router-2 is the **ABR**, sitting in Area 0 (Gig0/0/0), Area 10 (Se0/1/0), and Area 20 (Se0/1/1).

---

## 2. The OSPF Cost Formula

OSPF cost (metric) is calculated per interface as:

```
Cost = Reference Bandwidth ÷ Interface Bandwidth
```

By default, Cisco IOS uses a **reference bandwidth of 100,000 Kbps (100 Mbps)**. The result is truncated to a whole number, with a minimum value of 1.

### Applying it to your interfaces

| Interface type | Bandwidth | Calculation | OSPF Cost |
|---|---|---|---|
| GigabitEthernet | 100,000 Kbit | 100,000 ÷ 100,000 | **1** |
| Serial (T1) | 1,544 Kbit | 100,000 ÷ 1,544 = 64.77 → truncated | **64** |

This matches what `show ip ospf interface` / interface cost would report on all three routers, since every Gig link shows 100,000 Kbit and every Serial link shows 1,544 Kbit in your `show interface` output.

---

## 3. Verifying the Metric Against Your Routing Tables

OSPF's total path cost = **sum of outgoing interface costs along the path**, not including the cost of the originating LAN interface where the route is advertised from. Here's the breakdown for each router's `show ip route` output:

### Router-1

| Destination | Path | Cost Math | Routing Table Shows |
|---|---|---|---|
| 192.168.20.0/24 | Se0/1/0 → R2 Gig0/0/0 | 64 + 1 | **65** ✅ |
| 23.0.0.0/8 | Se0/1/0 → R2 Se0/1/1 | 64 + 64 | **128** ✅ |
| 192.168.30.0/24 | Se0/1/0 → R2 Se0/1/1 → R3 Gig0/0/0 | 64 + 64 + 1 | **129** ✅ |

### Router-2

| Destination | Path | Cost Math | Routing Table Shows |
|---|---|---|---|
| 192.168.10.0/24 | Se0/1/0 → R1 Gig0/0/0 | 64 + 1 | **65** ✅ |
| 192.168.30.0/24 | Se0/1/1 → R3 Gig0/0/0 | 64 + 1 | **65** ✅ |

### Router-3

| Destination | Path | Cost Math | Routing Table Shows |
|---|---|---|---|
| 192.168.20.0/24 | Se0/1/1 → R2 Gig0/0/0 | 64 + 1 | **65** ✅ |
| 12.0.0.0/8 | Se0/1/1 → R2 Se0/1/0 | 64 + 64 | **128** ✅ |
| 192.168.10.0/24 | Se0/1/1 → R2 Se0/1/0 → R1 Gig0/0/0 | 64 + 64 + 1 | **129** ✅ |

Every value in your captured routing tables lines up exactly with the bandwidth-derived cost formula — this confirms OSPF on all three routers is using the **default reference bandwidth (100 Mbps)** with no manual cost overrides.

---

## 4. What Happens If You Change the Bandwidth?

The `bandwidth` command on an interface does **not** change the actual physical link speed — it's a value OSPF (and EIGRP) use purely for metric calculation. Changing it directly changes the cost, and can change which path OSPF prefers.

**Example — increasing Router-1's serial bandwidth:**
```
Router-1(config)#interface serial0/1/0
Router-1(config-if)#bandwidth 10000
```
New cost = 100,000 ÷ 10,000 = **10** (down from 64)

Effects:
- The route to 192.168.20.0/24 via R2 would drop from cost 65 to **11**.
- If there were a second, alternate path to the same destination with a higher real cost, OSPF would now wrongly prefer this artificially "fast" path — because OSPF trusts the configured `bandwidth` value, not the real link throughput.
- This is exactly why mismatched `bandwidth` statements are a common real-world misconfiguration: the SPF algorithm picks paths based on declared bandwidth, so if it doesn't reflect reality, suboptimal or even congested paths can get selected as "best."

**Example — lowering bandwidth instead** (e.g., simulating a slower WAN link):
```
Router-2(config-if)#bandwidth 512
```
New cost = 100,000 ÷ 512 = **195**
- This link becomes far less attractive to OSPF, and traffic would be rerouted over any lower-cost alternate path if one existed.

**Key takeaway:** `bandwidth` is a *control-plane* value. It affects routing decisions and metric math, but a `bandwidth` command alone does not throttle or speed up actual data transfer.

---

## 5. What Happens If You Change the Metric (Cost) Directly?

Instead of changing bandwidth, you can override the calculated cost directly with `ip ospf cost`, which bypasses the bandwidth formula entirely:

```
Router-1(config)#interface serial0/1/0
Router-1(config-if)#ip ospf cost 20
```

Effects:
- This interface's OSPF cost becomes a fixed **20**, regardless of what `bandwidth` says.
- It's the more surgical / predictable tool versus changing `bandwidth`, because it has zero side effects on QoS, CIR calculations, or anything else that also reads the `bandwidth` value.
- Common real-world use: traffic engineering — e.g., forcing OSPF to prefer one redundant WAN link over another without touching the interface's reported bandwidth (which other features may depend on).

**Bandwidth vs. cost — when to use which:**

| Goal | Recommended command | Why |
|---|---|---|
| Make OSPF reflect a link's true speed (e.g., correcting a default that doesn't match reality) | `bandwidth` | Keeps cost formula consistent and also feeds other bandwidth-aware features (QoS, EIGRP, interface load %) |
| Influence only the OSPF path selection, leave everything else alone | `ip ospf cost` | Surgical — affects OSPF only |
| Influence path selection across a network using **multiple high-speed links** (Gig, 10G, etc.) where reference bandwidth no longer differentiates them | `auto-cost reference-bandwidth` (process-wide) | Without this, all links ≥100 Mbps get cost 1, making them indistinguishable to OSPF |

**A note on reference bandwidth:** In this lab, the default 100 Mbps reference bandwidth still works fine because only Gig (100 Mbps) and Serial (1.544 Mbps) links exist. But if you later add a Gig link and a 10 Gig link in the same topology, both would calculate to cost 1 under the default reference bandwidth — making them appear equally "expensive" even though one is 100x faster. That's solved with:
```
router ospf 50
auto-cost reference-bandwidth 100000
```
(this raises the reference to 100,000 Mbps so higher-speed links are differentiated) — worth keeping in mind for a future lab once you start mixing link speeds.

---

## 6. Summary

- All costs across Router-1, Router-2, and Router-3 match the expected formula: **Cost = 100,000 ÷ Bandwidth (Kbps)**.
- Gig links = cost 1; Serial (T1) links = cost 64.
- End-to-end path costs (65, 128, 129) all check out against the captured `show ip route` outputs — confirming OSPF is computing the SPF tree correctly with default settings.
- Manually changing `bandwidth` shifts the cost (and can change best-path selection) but does not change real throughput — a frequent source of real-world misconfiguration.
- `ip ospf cost` is the safer, more targeted way to influence OSPF path preference without affecting other bandwidth-dependent behavior.
