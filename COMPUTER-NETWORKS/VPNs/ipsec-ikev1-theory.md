# 🛡️ IPsec & IKEv1 Deep Dive: Architecture, Phases & Protocols

**IPsec** is not a single protocol; it is a framework—an architecture that utilizes other protocols like IKE and ISAKMP. Although created in the 1990s, it remains the backbone of corporate VPNs (though today, IKEv2 is the standard for 85% of tunnels). Understanding IKEv1 is crucial for exams like CCNP SCOR and for grasping the history of network security.

### Framework vs. Protocol

Both IKE and ISAKMP are used interchangeably as a protocol/framework. However, it is better to call ISAKMP a protocol. Why? Because IANA assigned it **UDP Port 500**, and that is the exact protocol we will see in Wireshark. 
A framework is a purely abstract, logical, intangible concept—a set of rules on paper. Since IANA gave it a port, it becomes tangible, physical, and thus becomes a protocol.

*   **ISAKMP (Internet Security Association and Key Management Protocol):** Defines the message format. It's like an envelope and postal rules. It dictates what the message header should look like, where the payload is, etc.
*   **IKE (Internet Key Exchange):** Decides *how* the key exchange should look (Main Mode or Aggressive Mode). We won't see IKE directly in Wireshark [well, not exactly, but I'll explain]. It is also a protocol because we physically have key exchange (KE) proposals, AES/SHA algorithms, and negotiation info. As mentioned earlier, ISAKMP is the envelope, so whatever is hidden inside won't be visible until we look inside. So yes, we will see it if we click inside the ISAKMP packet. However, newer versions of Wireshark are smart enough to just show IKE normally.

---

## 🔄 IKEv1 Phase 1: Main Mode vs. Aggressive Mode

In IKEv1, we have 2 phases. To make things more interesting, Phase 1 has 2 modes: Main Mode and Aggressive Mode. Fortunately, we don't have this in IKEv2.

*   **Main Mode (6 Packets):** Peers exchange a total of 6 messages in 3 rounds. This mode protects the identity of the peers, regardless of whether we use a Pre-Shared Key (PSK) or certificates. We use this for Site-to-Site (S2S) VPNs.
*   **Aggressive Mode (3 Packets):** Consists of a 3-packet exchange. It does *not* protect identity unless we use certificates. So why was this creature even created? Mainly for Remote Access (RA) VPNs. With RA VPNs, we very often have dynamic IP addresses. In Main Mode, creating a PSK requires static addresses, so the tunnel simply won't come up. To bypass this, Aggressive Mode + certificates was invented.

By default, Cisco sets PSK for both modes just to make it easier and get it working. It is then up to the admin to change it to certificates for RA VPNs. 
**Summary:** For S2S we use Main Mode (PSK/Certs), and for RA we use Aggressive Mode + Certs to solve the dynamic IP problem.

> **🏴‍☠️ The Hacker's Dream:** 
> In the past, hackers rubbed their hands together (and cracked passwords offline in the comfort of their homes) because admins massively used Aggressive Mode with PSK. They did this to save money on buying static IPs from ISPs. This "leaky" setup was a plague not only in RA VPNs but even in S2S tunnels when small branch offices were connected to the HQ via cheap dynamic IP links (like LTE modems).

### The 6-Packet Exchange (Main Mode)

<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 14px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
[ Router A / Initiator ] <=================================> [ Router B / Responder ]
</pre>

**Packets 1 & 2 [Round 1]:**
The initiator router sends its SA proposal. Router B says, "Works for me." What is transmitted? To avoid memorizing this, just remember **"HAGLE"**:
*   **H**ash (e.g., SHA-256)
*   **A**uthentication (PSK or Certs)
*   **G**roup (Diffie-Hellman, e.g., DH 14)
*   **L**ifetime (How long the tunnel lives, default 86400 seconds)
*   **E**ncryption (e.g., AES-256)

**Packets 3 & 4 [Round 2] - "The Magic of Mathematics" (Diffie-Hellman):**
This is the absolute heart of IPsec. Since the routers agreed to use AES encryption, where will they get the key (password) for it? They can't just send it over the internet because a hacker will intercept it! 
Enter the Diffie-Hellman (DH) algorithm. The routers exchange mathematical "half-products" in plain text. Thanks to the magic of math (and prime numbers), both routers calculate the exact same, powerful master key locally. A hacker who intercepted packets 3 and 4 only has useless garbage—they can't assemble the final key from it.

**Packets 5 & 6 [Round 3] - Authentication:**
From this moment on, everything is encrypted using the key calculated in the previous round. Now, the routers secretly send each other proofs of identity, namely their IPs and a hash of the PSK. They hash this together and compare if it matches. If it does, authentication is successful.

### IP Spoofing & The Boomerang Effect

What if a hacker along the way wants to generate a DH key with the initiator? They would need the exact same public IP and the PSK (e.g., by eavesdropping on a phone call between two admins). This is practically impossible.

What about spoofing the public address? If a hacker sits in a basement, spoofs their source IP to match your HQ, and sends a packet to your branch, the branch will send the reply... to the real HQ! The hacker in the basement will never see that reply. The packet acts like a boomerang returning to its rightful owner.

**So why does the hacker do it?** 
The hacker sends a million initiation packets with a spoofed IP. Your router sweats, calculates DH keys, sends replies into the void, and the hacker in the basement drinks coffee and watches your corporate network die from overload. This is a classic DoS attack. 
*   **Aggressive Mode:** Vulnerable, because the router immediately calculates DH.
*   **Main Mode:** Safe. It uses "cookies". Since the routing doesn't work as it should (boomerang effect), the cookies won't match, and the DH function won't even start.

*(Note: Today, good ISPs use uRPF/BCP38. It acts like a bouncer in a club. If a customer sends a packet with a spoofed source IP, the ISP drops it in the trash).*

---

## 🚀 IKEv1 Phase 2: Quick Mode

Phase 1 is lightweight and bidirectional, while Phase 2 has **two unidirectional tunnels** for a single network (inbound and outbound). If we have 3 subnets, there will be 6 SA tunnels. 

Why? Phase 2 is a beast, so to improve performance, it's better to separate downloading and uploading. Separation also allows everything to be processed better by ASIC chips. Each tunnel has its own counters, encryption key, and a unique **SPI (Security Parameter Index)** attached to the packet header so the receiver can easily identify which "AES drawer" to reach into.

### The Rekey Storm
What if we had 1000 active users via RA VPN? Haha, then we get 2000 SAs. Try finding something in that jungle XD. The worst part was that in the old IPsec (IKEv1), each of those 1000 users had to renew their keys (Rekey). If 500 users connected at 8:00 AM, then at 9:00 AM the concentrator's CPU suddenly received 500 simultaneous requests to recalculate heavy Diffie-Hellman math. Devices often simply dropped to their knees from overload (a Rekey Storm).
**The solution to this is the new IKEv2 and AnyConnect, which uses port 443 and is very lightweight.**

### The 3 Steps of Quick Mode

**1. Transform Set (The Truck and the Armor)**
*   **Protocol:** ESP or AH. AH encrypts nothing, so in practice, everyone flies on ESP.
*   **Data cipher:** "Okay, we used AES-128 to protect Phase 1, but the accounting data is super secret, so let's use a stronger AES-256 in Phase 2."
*   **Hash:** "We will attach a SHA seal to every packet with a file so that a hacker doesn't change the invoice amount on the fly."

**2. Proxy IDs / Traffic Selectors (ACLs vs. VTI)**
Routers must determine whose traffic is allowed to enter the tunnel.
*   **Policy-Based (Classic ACLs):** Router A says: *"I propose letting traffic from 10.1.1.0 to 10.2.2.0"*. Router B must say: *"I agree, I have exactly the same mirror rule"*. If you make a typo, the entire tunnel drops.
*   **Route-Based (VTI):** Cisco programmers did a brilliant trick. Router A automatically says: *"Hey, I propose letting traffic from 0.0.0.0 (everything) to 0.0.0.0 (everywhere)"*. Router B agrees. Phase 2 negotiates only once as one giant, open pipe. You push traffic there using standard routing (`ip route 10.0.0.0 255.255.255.0 Tunnel 0`).

> **Why can't ACLs encrypt Multicast?**
> When IPsec was invented in the 90s, it was designed exclusively for Unicast. Multicast (e.g., OSPF packets) is a "shout in a crowd". Classic IPsec doesn't have a mechanism to handle this shout. For OSPF to send a packet, it needs a real interface. A classic ACL is just an invisible filter—OSPF cannot "hook" into it. VTI, however, is a fully-fledged virtual interface. OSPF looks at it and thinks: "Oh, a normal network card, I can send my Hello packets here!". VTI catches it, encapsulates it, encrypts it via ESP, and sends it.

**3. PFS (Perfect Forward Secrecy)**
We completely ignore the "garbage" from Phase 1, generate a brand new DH key, and encrypt the data with it. 
*(Corporate Reality: Technically, you can juggle 15 different ciphers for 15 subnets. But at 2:00 AM, when a tunnel goes down and you see 30 different SAs, you'll want to quit your job. That's why 99% of corporations apply the KISS principle: one global standard for all subnets).*

---

## 📦 The Protocols: ESP vs. AH

IKEv1 and IKEv2 are protocols that just "talk". They establish passwords, check certificates, and exchange keys. When they finish their job, they go for coffee. By themselves, they cannot encrypt a single byte of your data! Someone has to do the dirty work, and that someone is ESP.

*   **AH (Authentication Header - Protocol 51):** Provides integrity and authentication, but **NO ENCRYPTION**. It hates NAT because it hashes the entire packet (including the IP header). Today, it's a museum exhibit.
*   **ESP (Encapsulating Security Payload - Protocol 50):** Provides encryption. It only hashes the payload, so there is no problem with NAT regarding integrity. 99% use ESP today.

---

## 🏗️ Encapsulation Modes: Transport vs. Tunnel

**1. Transport Mode (The Economical Mode)**
ESP pulls just the "piece of paper with text" (data) out of the envelope, encrypts it, and then puts it back into THE SAME, original envelope. We almost never use this in S2S because private addressing in the headers won't pass through the internet. Used in DMVPN or GRE over IPsec.

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

**2. Tunnel Mode**
We add an additional header in the form of public IPs, and the rest is encrypted. 99% of the time, Tunnel Mode is used. *(Note: These tunnels don't work the way we are taught, that one is inside the other. They operate independently, and one does something for the other).*

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
Tunnel mode has one flaw: it adds extra headers (a lot of weight to the packet). The new IP header is 20 bytes, and ESP headers/trailers are another few dozen bytes. 
If a PC normally sends 1500 bytes and the router adds a new envelope from tunnel mode, the packet swells to, say, 1560 bytes. The ISP's network card says: *"Oh no, my limit is 1500. I'm cutting this packet into two pieces!"* (fragmentation). Fragmentation in IPsec is the death of performance. The router's CPU burns trying to glue it back together, and half of the websites stop loading.

> **💡 Pro-Tip for documentation:** 
> That's why every self-respecting engineer, when configuring IPsec in Tunnel Mode, types the magic command on the interface: `ip tcp adjust-mss 1360`. This tells the computers: *"Hey, send slightly smaller packages, because I need room to pack them into these thick, encrypted envelopes of mine."*

Supposedly routers have a mechanism that automatically reduces MTU, but it's worth typing it in manually anyway:
<pre style="background-color: #000000; color: #ffffff; padding: 15px; font-size: 15px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
Router(config)# interface Tunnel 0
Router(config-if)# ip mtu 1400
Router(config-if)# ip tcp adjust-mss 1360
</pre>
*(Note: Setting MTU for a tunnel interface in the Route-Based method is simple because we set it on the tunnel interface. However, in Policy-Based, we set the MTU on the interface facing the private network).*

### 2. NAT-T (NAT Traversal)
It's also worth mentioning that both ESP and AH are Layer 3 protocols, so there's a problem with devices behind PAT. The solution for this is the pass-through feature. It roughly relies on pulling the SPI number into Layer 4. 
A better feature is **NAT-T**, and it relies on the fact that when VPN peers detect that devices are behind NAT, they add **UDP port 4500** to encapsulate the packet.