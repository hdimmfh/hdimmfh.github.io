---
title: Network Overlay Protocols(1) — GRE/VXLAN/LISP
by: hdimmfh
date: 2025-11-16 13:30:00 +0900
categories: [Network, Network Virtualization]
tags: [Overlay, Encapsulation]
---

🔍 **Understanding Modern Network Encapsulation**

> “They look similar, but their purpose, header structure, and encapsulation scope are completely different.”  
> In this post, we break down how `GRE`, `VXLAN`, and `LISP` differ in **what they encapsulate**, **which headers they generate**, and **what problems they are designed to solve**.

---

## ① Overview

Modern overlay protocols abstract networks into flexible, virtualized layers.  
Although GRE, VXLAN, and LISP all use encapsulation, their design goals are entirely different:

- **GRE** — traditional L3 tunneling  
- **VXLAN** — L2 extension over L3 underlay (DC standard)  
- **LISP** — EID/RLOC separation for scalable, mobile, multi-site routing  

---

## ② GRE — Generic Routing Encapsulation

GRE provides a simple form of **IP-over-IP tunneling**.  
It’s widely used to run routing protocols (OSPF, EIGRP) across WAN or public networks.

![GRE Header](https://github.com/hdimmfh/hdimmfh.github.io/blob/main/_data/blog-img-repo/img/network/tunneling/gre-header.webp?raw=true)
*Figure 1. GRE Header and Encapsulation.*

✔️ **Header creation**
- New L2 header: **No**  
- New L3 header (Outer IP): **Yes**  
- GRE header: **Yes**

✔️ **Encapsulation scope**
- Typically **the entire L3 packet (IP packet)**  
- Can encapsulate L2, but rarely used this way

✔️ **Purpose**
- Basic tunneling (IP-in-IP)  
- Site-to-site connectivity  
- Carry routing protocols over untrusted or simple IP networks  
- No security on its own (combine with IPsec)

✔️ **Typical architecture**
- Private WAN  
- Site-to-site tunnels  
- Cloud ↔ On-prem hybrid routing  

---

## ③ VXLAN — Virtual Extensible LAN

VXLAN is the modern data center standard for **extending L2 domains over an L3 IP underlay**.

![VXLAN header](https://github.com/hdimmfh/hdimmfh.github.io/blob/main/_data/blog-img-repo/img/network/tunneling/vxlan-vtep-header.png?raw=true)
*Figure 2. VXLAN Header and Encapsulation.*

✔️ **Header creation**
- New L2 header: **Yes (Inner Ethernet preserved)**  
- New L3/UDP header (Outer IP + UDP): **Yes**  
- VXLAN header: **Yes**

✔️ **Encapsulation scope**
- **Entire Ethernet frame (L2 + L3 + payload)**  
- Allows L2 to “ride” over L3 fabric unchanged

✔️ **Purpose**
- L2 domain extension at massive scale  
- EVPN multi-tenant architecture  
- Overcomes VLAN limit (4094) → VXLAN VNI supports 16 million segments  
- VM/container mobility across DC fabric

✔️ **Typical architecture**
- Spine–Leaf data centers  
- EVPN VXLAN fabrics  
- Private underlay L3 IP networks  

---

## ④ LISP — Locator/ID Separation Protocol

LISP separates a device’s identifier (**EID**) from its routing locator (**RLOC**).  
This enables scalable multi-site routing, mobility, and WAN-level optimization.

![LISP header](https://github.com/hdimmfh/hdimmfh.github.io/blob/main/_data/blog-img-repo/img/network/tunneling/LISP-header-ipcisco.png?raw=true)
*Figure 3. LISP header and Encapsulation.*

✔️ **Header creation**
- New L2 header: **No**  
- New L3 header (Outer RLOC IP): **Yes**  
- LISP header: **Yes**

✔️ **Encapsulation scope**
- **Entire L3 packet (EID-based)**  
- Not meant for L2 extension

✔️ **Purpose**
- Separate identity (host) from location (network)  
- Support mobility  
- Enable multi-homing across providers  
- Reduce global routing table size  
- Foundation for SD-WAN overlay architectures

✔️ **Typical architecture**
- SD-WAN  
- Multi-site WAN interconnect  
- Internet-across overlays  
- Cloud ↔ On-prem routing abstraction  

---

## ⑤ Comparison Summary

A clean comparison of GRE, VXLAN, and LISP using the six requested criteria:

| Category | **GRE** | **VXLAN** | **LISP** |
|----------|---------|-----------|----------|
| **1. New L2 header?** | No | Yes ⭐️ | No |
| **2. New L3 header?** | Yes | Yes | Yes |
| **3. Custom header?** | GRE Header | VXLAN Header | LISP Header |
| **4. Encapsulation scope** | L3 packet | **Full L2 Ethernet frame** | L3 packet (EID) |
| **5. Purpose** | Basic tunneling / routing transport | **L2 extension / EVPN / DC overlay** | **ID–Locator separation / SD-WAN / global scalability** |
| **6. Typical architecture** | WAN, Site-to-Site | **DC Spine–Leaf, EVPN** | SD-WAN, multi-site, Internet overlay |

---

## ⑥ Practical Notes

- **GRE** = “Classic IP-in-IP tunnel”  
- **VXLAN** = “L2 expansion over an L3 data center fabric”  
- **LISP** = “Internet/WAN-scale routing abstraction via ID–Locator split”

> In one line:  
> **GRE connects**, **VXLAN extends**, **LISP abstracts**.

---

## ⑦ TL;DR

| Goal | Recommended Tech |
|------|------------------|
| Simple IP tunneling | **GRE** |
| Large-scale L2 extension | **VXLAN** |
| Mobility, SD-WAN, multi-site routing | **LISP** |

---

## 📚 References

- RFC 2784 — GRE  
- RFC 7348 — VXLAN  
- RFC 6830 — LISP  
- Cisco EVPN VXLAN Design Guide  
- Cisco SD-WAN & LISP Architecture  