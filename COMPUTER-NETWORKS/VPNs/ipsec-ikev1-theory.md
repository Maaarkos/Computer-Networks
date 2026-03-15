# 🛡️ IPsec & IKEv1 Deep Dive: Architecture, Phases & Protocols

**IPsec** is not a single protocol; it is a framework—an architecture that utilizes other protocols like IKE and ISAKMP. Although created in the 1990s, it remains the backbone of corporate VPNs (though today, IKEv2 is the standard for 85% of tunnels). Understanding IKEv1 is crucial for exams like CCNP SCOR and for grasping the history of network security.

*   **ISAKMP (Internet Security Association and Key Management Protocol):** This is the "envelope" and the postal rules. It defines the message format (headers, payloads). Because IANA assigned it **UDP Port 500**, it is a tangible protocol you will see in Wireshark.
*   **IKE (Internet Key Exchange):** This is the brain inside the envelope. It decides *how* keys are exchanged (Main Mode vs. Aggressive Mode) and negotiates algorithms (AES, SHA). In older Wireshark versions, IKE is hidden inside the ISAKMP packet, but modern Wireshark is smart enough to label it directly.

---

## 🔄 IKEv1 Phase 1: Establishing the Secure Channel

Phase 1 creates a secure, bidirectional management tunnel. IKEv1 does this using one of two modes:

### 1. Main Mode vs. Aggressive Mode

*   **Main Mode (6 Packets):** Protects the identity of the peers. Used primarily for Site-to-Site (S2S) VPNs.
*   **Aggressive Mode (3 Packets):** Does *not* protect peer identity (unless certificates are used). 
    *   *Why was it created?* For Remote Access (RA) VPNs where clients have dynamic IP addresses. Main Mode with Pre-Shared Keys (PSK) requires static IPs. Aggressive Mode bypassed this limitation.
    *   *The Hacker's Dream:* In the past, admins used Aggressive Mode + PSK to save money on static IPs. Hackers loved this because they could easily capture the hash and crack the PSK offline in their basements. 

### 2. The 6-Packet Exchange (Main Mode)

**Packets 1 & 2 (Negotiation):**
Router A (Initiator) sends an SA Proposal. Router B (Responder) agrees. They negotiate the **HAGLE** parameters:
*   **H**ash (e.g., SHA-256)
*   **A**uthentication (PSK or RSA Certificates)
*   **G**roup (Diffie-Hellman, e.g., DH 14)
*   **L**ifetime (Default: 86400 seconds / 1 day)
*   **E**ncryption (e.g., AES-256)

**Packets 3 & 4 (The Math Magic - Diffie-Hellman):**
This is the heart of IPsec. They agreed on AES, but how do they share the password over the internet without a hacker stealing it? Diffie-Hellman (DH) allows them to exchange mathematical "half-products" in plain text. Both routers calculate the same master key locally. A hacker capturing these packets gets useless garbage.

**Packets 5 & 6 (Authentication):**
From now on, everything is encrypted using the DH key. The routers secretly exchange their identities (IP addresses) and a Hash of the PSK to prove who they are.

> **🛡️ Security Note: IP Spoofing & DoS**
> Can a hacker spoof an IP, start a DH calculation, and crash the router? 
> *   In **Aggressive Mode**: Yes. The router immediately calculates DH (heavy CPU load) before verifying the peer, leading to a DoS attack.
> *   In **Main Mode**: No. Main mode uses "Cookies". If the hacker's spoofed IP cannot receive the return routing traffic, the cookie exchange fails, and the router never starts the heavy DH math.
> *   *Modern Defense:* Today, good ISPs use **uRPF (BCP38)**—a "bouncer" that drops packets if the source IP doesn't belong to the customer sending it.

---

## 🚀 IKEv1 Phase 2: Quick Mode (The Data Tunnels)

While Phase 1 is a single bidirectional tunnel, **Phase 2 creates TWO unidirectional tunnels** (Inbound and Outbound) for *each* subnet pair. 

*Why separate them?* Performance. Splitting RX and TX allows hardware ASIC chips to process encryption much faster. Each tunnel has its own counters, encryption keys, and a unique **SPI (Security Parameter Index)**.
The SPI is a 32-bit number attached to the packet header, allowing the receiving router to instantly know which AES key to use for decryption.

### 1. Transform Sets (The Armor)
Here, we choose the protocol (ESP or AH) and the encryption for the actual user data. For example, we might have used AES-128 for Phase 1, but we use AES-256 for Phase 2 to protect highly sensitive accounting data.

### 2. Proxy IDs / Traffic Selectors (ACLs vs. VTI)
The routers must agree on *what* traffic is allowed inside the tunnel.

*   **Policy-Based (Legacy ACLs):** Router A says: *"I want to route 192.168.1.0/24 to 10.0.0.0/24"*. Router B must have an exact mirror ACL. If you make a typo, the entire tunnel drops. Also, ACLs cannot route Multicast (like OSPF).
*   **Route-Based (VTI - Virtual Tunnel Interfaces):** A brilliant Cisco workaround. VTI automatically negotiates `0.0.0.0/0` to `0.0.0.0/0` (everything). The IPsec tunnel becomes a giant, open pipe. You control what goes inside using standard routing (`ip route 10.0.0.0 255.255.255.0 Tunnel 0`). Because VTI is a real interface, it fully supports Multicast and routing protocols!

### 3. PFS (Perfect Forward Secrecy)
If enabled, Phase 2 does *not* reuse the DH key from Phase 1. It generates a brand new DH key for the data tunnels. If a hacker eventually cracks the Phase 1 key, your Phase 2 data remains secure.

---

## 📦 The Protocols: ESP vs. AH

IKEv1 and IKEv2 are just negotiators. They talk, agree on passwords, and go drink coffee. They do not encrypt your data. The actual heavy lifting is done by ESP or AH.

### 1. AH (Authentication Header - Protocol 51)
Provides integrity and authentication, but **NO ENCRYPTION**. 
*   **The NAT Problem:** AH hashes the *entire* packet, including the original IP header. If a NAT router changes the IP address along the way, the AH hash breaks, and the packet is dropped. Today, AH is a museum exhibit.

### 2. ESP (Encapsulating Security Payload - Protocol 50)
Provides encryption, integrity, and authentication. It only hashes the payload, so it survives NAT (mostly). 99% of modern VPNs use ESP.

---

## 🏗️ Encapsulation Modes: Transport vs. Tunnel

How do we package the data?

### 1. Transport Mode
The "Eco" mode. ESP extracts the data, encrypts it, and puts it back behind the **original** IP header. Rarely used in S2S VPNs because private IPs cannot traverse the internet. Used mainly in DMVPN or GRE over IPsec.

**ESP Transport Mode:**
<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 13px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
+-------------+-----------+-------------------------+-------------+-----------+
| Orig IP Hdr |  ESP Hdr  |         Payload         | ESP Trailer | ESP Auth  |
+-------------+-----------+-------------------------+-------------+-----------+
              | &lt;------------- Encrypted -----------------------&gt; |
              | &lt;------------------ Authenticated --------------------------&gt; |
</pre>

**AH Transport Mode (No Encryption):**
<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 13px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
+-------------+-----------+-------------------------+
| Orig IP Hdr |   AH Hdr  |         Payload         |
+-------------+-----------+-------------------------+
| &lt;---------------- Authenticated ----------------&gt; |
</pre>

### 2. Tunnel Mode
The standard mode (99% of VPNs). ESP takes the entire original packet (including the private IP header), encrypts it, and slaps a **brand new Public IP header** on the front.

**ESP Tunnel Mode:**
<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 13px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
+-------------+-----------+-------------+-----------+-------------+-----------+
| New IP Hdr  |  ESP Hdr  | Orig IP Hdr |  Payload  | ESP Trailer | ESP Auth  |
+-------------+-----------+-------------+-----------+-------------+-----------+
              | &lt;----------------- Encrypted -------------------&gt; |
              | &lt;---------------------- Authenticated ----------------------&gt; |
</pre>

**AH Tunnel Mode (No Encryption):**
<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 13px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
+-------------+-----------+-------------+-----------+
| New IP Hdr  |   AH Hdr  | Orig IP Hdr |  Payload  |
+-------------+-----------+-------------+-----------+
| &lt;---------------- Authenticated ----------------&gt; |
</pre>

---

## ⚠️ The Hidden Killers: MTU & NAT

### 1. The MTU/MSS Fragmentation Problem
Tunnel mode adds massive overhead (20 bytes for the new IP + dozens of bytes for ESP headers). If a PC sends a standard 1500-byte packet, the router adds the ESP envelope, making it ~1560 bytes. The ISP will drop or fragment it, killing network performance.

**Pro-Tip:** Always adjust the TCP MSS (Maximum Segment Size) to tell PCs to send smaller packets, leaving room for the VPN headers.
<pre style="background-color: #000000; color: #ffffff; padding: 15px; font-size: 15px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
Router(config)# interface Tunnel 0
Router(config-if)# ip mtu 1400
Router(config-if)# ip tcp adjust-mss 1360
</pre>
*(Note: For Route-Based VPNs, apply this to the Tunnel interface. For Policy-Based, apply it to the LAN-facing interface).*

### 2. NAT-T (NAT Traversal)
ESP and AH are Layer 3 protocols (they have no ports like TCP/UDP). If your VPN router is behind a NAT/PAT device (like a home router), PAT doesn't know how to translate it because there are no ports to track!

**The Solution:** `NAT-T`. When VPN peers detect a NAT device between them, they wrap the entire ESP packet inside a standard **UDP Port 4500** header. Now the PAT router is happy, and the VPN works flawlessly.