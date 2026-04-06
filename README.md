# MikroTik Router VPS to VLESS Residential Node

A comprehensive setup guide and network architecture for routing specific **VLESS (xBoard/V2bX)** proxy traffic from a Cloud VPS through a home **MikroTik Router** via a **WireGuard** tunnel. 

This architecture allows you to create a Proxy Node on your VPS that securely exits the internet through your home broadband connection, effectively giving your VPN users a **100% Genuine Residential IP** (BD IP in this case) instead of a Data Center IP. It is perfect for bypassing geo-restrictions, heavy enterprise firewalls (like GFW), or accessing local banking apps that block generic VPN IPs.

## ✨ Key Features
- **WireGuard Tunneling:** Establishes a lightweight, high-performance UDP tunnel (`10.0.0.0/24`) between the Cloud VPS and the local MikroTik Router.
- **Linux Policy Routing (fwmark):** Uses `ip route` and `iptables` `mark 255` to selectively push only the traffic from our specific Residential VLESS Node to the MikroTik router, leaving the rest of the VPS's standard proxy nodes completely untouched.
- **V2bX Node Duplication:** Runs a secondary instance of `V2bX` (V2bX-BD) to safely isolate the Residential Node, completely avoiding port and routing conflicts.
- **IPv6 Leak Prevention:** Employs strict IPv6 blackhole routes and Xray's `UseIPv4` domain strategies to guarantee the VPS's data center IPv6 address does not leak during remote connections.
- **TCP Path MTU Fix:** Implements `TCPMSS --clamp-mss-to-pmtu` to resolve TCP timeout issues over WireGuard, ensuring heavy loading and TLS handshakes execute instantly.

## 🛠️ Setup Instructions
The step-by-step installation instructions for both the **aaPanel (Debian/Ubuntu) VPS** and the **MikroTik Router (RouterOS)** are documented carefully in plain text.

👉 **View the Setup Guide:** [`wireguard_full_reference.md`](wireguard_full_reference.md)

## 📌 Prerequisites
1. A VPS running Ubuntu/Debian (aaPanel support included).
2. A MikroTik Router (e.g., hAP ax lite) with RouterOS v7+.
3. V2bX or XrayR installed on the VPS for xBoard connection.
4. Basic knowledge of terminal and Winbox.
