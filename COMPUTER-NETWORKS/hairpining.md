# 🪃 Hairpinning (U-Turn) & Asymmetric Routing

**Hairpinning** (or U-Turn routing) occurs when a packet arrives at a router or firewall interface and is routed back out through the exact same interface. While sometimes configured intentionally (e.g., for NAT loopback), it often happens accidentally due to misconfigurations, leading to dropped traffic.

### 📝 Real-World Scenario: The Subnet Mask Mismatch

**The Goal:** A client (`172.16.3.170`) attempts to connect via SSH to a server (`172.16.3.50`). Both should be in the same `/24` subnet.
**The Problem:** The SSH connection fails. However, when the client IP is changed to `172.16.3.56`, the connection succeeds.

**🔍 Root Cause Analysis:**
The server was misconfigured with a `/25` subnet mask instead of a `/24`.
* **Client 2 (`.56`):** From the server's `/25` perspective, `.56` is in the *same* local subnet (`.0 - .127`). The server replies directly via Layer 2 (MAC address). **Connection works.**
* **Client 1 (`.170`):** From the server's `/25` perspective, `.170` is in a *different* subnet. The server sends the reply to its Default Gateway (the Firewall). **Connection fails.**

---

### ⚙️ The Packet Flow & Why the Firewall Drops It

When the client (`.170`) initiates the connection, we experience **Asymmetric Routing** combined with **Hairpinning**:

1. **Outbound (Client -> Server):** The client has a `/24` mask, so it knows the server is local. It sends the `TCP-SYN` directly via the Layer 2 Switch. *The Firewall never sees this packet.*
2. **Inbound (Server -> Client):** The server has a `/25` mask, thinks the client is remote, and sends the `TCP SYN-ACK` to the Firewall.
3. **Hairpinning:** The Firewall receives the packet on the `inside` interface, checks its routing table, and realizes the destination (`.170`) is also out the `inside` interface. It sends the packet right back out.
4. **The Drop:** Stateful firewalls (like Cisco FTD) track TCP connection states. Because the firewall sees a `SYN-ACK` without ever seeing the initial `SYN`, it drops the packet as invalid. *(Note: ICMP/Ping might sometimes work in this scenario depending on inspection rules, but TCP will always fail).*

### 📊 Network Diagram

<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 15px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
[ SERVER .50 ]                                    [ CLIENT .170 ]
      |                                                 ^
      | 1. SYN-ACK packet leaves                        | 6. Packet returns
      v                                                 |
+-----------------------------------------------------------------+
|                          SWITCH (Layer 2)                       |
+-----------------------------------------------------------------+
      |                                                 ^
      | 2. Switch forwards to Gateway (FW)              | 5. Switch forwards to Client
      |                                                 |
      v                                                 |
===================================================================
|                      FIREWALL (Cisco FTD)                       |
|                                                                 |
|     +--- [ ONE PHYSICAL INTERFACE: inside_p1 ] -----+           |
|     |                                               |           |
|     | (Left side of the "U")      (Right side of the "U")       |
|     v                                               ^           |
|  3. INGRESS                                     4. EGRESS       |
|     |                                               |           |
|     +--------> [ ROUTING PROCESS / TABLE ] ---------+           |
|                 (The U-Turn / Hairpin)                          |
===================================================================
</pre>