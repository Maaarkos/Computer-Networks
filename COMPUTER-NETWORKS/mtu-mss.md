# 🪚 MSS vs MTU & The Nightmare of Fragmentation

Imagine network encapsulation as an assembly line. The packet is built from the inside (data) to the outside (the cable).

### 📏 The Assembly Line (Scope of Counters)

<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 14px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
+-------------+-------------+-------------+-----------------------+-------------+
| Layer 2     | Layer 3     | Layer 4     | Layer 7               | Layer 2     |
| (Cable)     | (Routing)   | (Transport) | (Application)         | (Cable)     |
+-------------+-------------+-------------+-----------------------+-------------+
| Ethernet    | IP Header   | TCP Header  | RAW DATA (Payload)    | FCS Trailer |
| 14 Bytes    | 20 Bytes    | 20 Bytes    | 1460 Bytes            | 4 Bytes     |
+-------------+-------------+-------------+-----------------------+-------------+
                            | <------- MSS (1460 Bytes) ------->  |
              | <----------------- MTU (1500 Bytes) ------------------------> |
| <------------------------- FRAME SIZE (1518 Bytes) ---------------------------------> |
</pre>

*   ▶ **MSS (Maximum Segment Size) = 1460 bytes**
    Covers ONLY the Raw Data. This is what the computer asks for in the TCP SYN packet.
*   ▶ **MTU (Maximum Transmission Unit) = 1500 bytes**
    Covers: IP Header (20) + TCP Header (20) + Data (1460). 
    *Golden Rule:* MTU is everything from Layer 3 upwards. This is the limit the router looks at to decide whether to pull out the chainsaw (fragmentation).
*   ▶ **Frame Size = 1518 bytes**
    Covers: Ethernet (14) + MTU (1500) + FCS (4). This is the physical size of the electrical impulses flying over the cable.

---

### 🪆 The IPsec Matryoshka (Tunnel MTU vs Physical MTU)

How does this look when we manipulate `tcp adjust-mss` to make a VPN work properly? We set the mechanism on the router to tell devices in the private network to negotiate a 1360 MSS. Thanks to this, we get a Tunnel MTU of 1400 bytes.

**1. The Inside (What the Tunnel Interface sees):**
<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 14px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
+----------------+----------------+--------------------------------+
| Inner IP Hdr   | Inner TCP Hdr  | RAW DATA (Reduced!)            |
| 20 Bytes       | 20 Bytes       | 1360 Bytes                     |
+----------------+----------------+--------------------------------+
| <---------------- TUNNEL MTU (1400 Bytes) ---------------------> |
</pre>
This is our original packet. Before the router starts encrypting it, it checks if it fits in the logical tunnel limit. 1400 bytes is the perfect size because it leaves room for the armor.

**2. The ESP Armor (What the Physical WAN Interface sees):**
Now the router takes that entire 1400 bytes, encrypts it, and wraps it in new headers (Tunnel Mode).
<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 14px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
+------------+-------------+-------------------------+-------------+-----------+
| New IP Hdr | ESP Hdr + IV| [ ENCRYPTED ORIGINAL ]  | ESP Trailer | ESP Hash  |
| 20 Bytes   | ~24 Bytes   | 1400 Bytes (Tunnel MTU) | ~12 Bytes   | ~16 Bytes |
+------------+-------------+-------------------------+-------------+-----------+
| <----------------------- PHYSICAL MTU (1472 Bytes) ------------------------> |
</pre>
The physical door of the router (WAN interface MTU) has a limit of 1500 bytes. Our packed, encrypted, fat IPsec packet weighs a total of 1472 bytes. **1472 < 1500. It passes perfectly!**

*(Note: Even though Cisco automatically adjusts MTU when you use the MSS feature, it's always better to set the MTU manually for peace of mind).*

---

### 🎥 What about UDP? (The Video Call Dilemma)

MSS only applies to TCP. What if we want to push a massive amount of UDP video traffic? We have two mechanisms: one network-based (which often fails) and one application-based (which saves the world).

**1. The Network Mechanism: PMTUD (Path MTU Discovery)**
This is the official internet standard. 
*   Your PC sends a large UDP packet (1500B) and intentionally sets the **DF (Don't Fragment)** bit to 1. It says: *"I'm sending this, but I forbid you to cut it!"*
*   The packet hits your router (Tunnel MTU 1400). The router says: *"It's too big, and I'm not allowed to cut it. I'm throwing it in the trash!"*
*   **The Magic:** The router doesn't drop it silently. It sends a small return letter (**ICMP Type 3, Code 4 - Fragmentation Needed**) to your PC: *"Hey PC, I killed your packet. My door is only 1400 bytes. Shrink your packets and try again."* The OS receives this and starts sending smaller packets.
*   **Why engineers hate it:** Overzealous security admins block ALL ICMP on firewalls "for security". The return letter never reaches the PC. The PC keeps sending huge packets, the router keeps killing them, and the video fails (The famous "Black Hole" issue).

**2. The App Mechanism: Smart Software (The Real Solution)**
Since the network can be unreliable and firewalls break PMTUD, app developers (MS Teams, Webex, Zoom) took matters into their own hands. They stopped trusting network engineers!
When you start a video call, the app assumes a worst-case scenario (e.g., there's an IPsec tunnel somewhere) and proactively encodes the video into small chunks, like 1100 or 1200 bytes (RTP payload). Even if a router adds 80 bytes of IPsec headers, it easily passes through any MTU.

> **⚠️ The Tragedy of Cut UDP:**
> If a dumb app sends a huge UDP packet with DF=0, the router cuts it. If piece #1 arrives but piece #2 gets lost in the internet, what happens?
> *   **TCP:** The PC asks for a retransmission.
> *   **UDP:** The PC cannot reassemble the packet without the missing piece. It throws the ENTIRE packet in the trash.
> *   **Result:** You see giant green squares (artifacts) on the screen, the video freezes, and the voice sounds like a robot.

---

### 🧟‍♂️ Hacker Attacks: The Fragmentation Nightmare

Fragmentation is a nightmare for routers in two main ways:
1.  **Cutting (On the fly):** The router must take a large packet, divide it, copy IP headers to each piece, recalculate checksums, and send it. This eats the **CPU**.
2.  **Reassembling (At the destination):** The receiving router must catch all pieces, put them in RAM, wait for the missing parts, and glue them together. This eats **RAM**.

Hackers have invented several brilliant attacks based on the lack of the DF bit:

*   🪓 **The Lumberjack Attack (Killing the CPU):**
    The hacker sends millions of 1500-byte packets (DF=0) through a link they know has an MTU of 1400. The poor ISP router becomes a "lumberjack". It does nothing but cut packets. The CPU spikes to 100%, and normal user traffic drops.
*   💧 **The Teardrop Attack (Killing the RAM / OS):**
    An absolute hit in the 90s. The hacker didn't force the router to cut; they cut the packets themselves and sent them to the victim. 
    *The Catch:* They manipulated the "Fragment Offset" so the pieces overlapped (e.g., piece #2 claimed it started in the middle of piece #1). When an old OS (like Windows 95) tried to glue it together, it got a mathematical error (kernel panic) and threw a Blue Screen of Death.
*   ⏳ **"Waiting for Godot" (Buffer Exhaustion):**
    The hacker sends fragmented packets but intentionally *never* sends the final piece. Your VPN concentrator catches pieces 1, 2, and 3. It puts them in RAM and waits for piece 4. It waits, and waits... The hacker sends millions of these unfinished puzzles. The RAM fills up with "garbage" waiting for endings until the router crashes.

### 🛡️ Modern Defenses & The "Smart Fridge" Botnet

If hackers are so smart, why does the internet still work? Engineers had to invent defensive shields:

*   **ASIC Hardware Cutting:** Modern, expensive routers don't cut packets with the main CPU anymore. They have dedicated chips (ASICs) that can slice millions of packets per second without breaking a sweat.
*   **VFR (Virtual Reassembly):** A brilliant feature in firewalls (like Cisco ASA). The firewall looks at the incoming pieces. If it sees a missing ending or overlapping pieces (like Teardrop), it immediately throws the whole puzzle in the trash before passing it to the main CPU.
*   **Rate Limiting:** Routers have built-in limits: *"If I get more than 10,000 fragmented packets per second from one IP, that IP gets banned for 5 minutes."*

**The Botnet Scenario (The Ultimate Test):**
One PC sending large packets is a mosquito bite to an ISP. But what if a hacker takes over 100,000 poorly secured IoT devices (CCTV, smart bulbs, Wi-Fi fridges) and tells them to send 1500B packets (DF=0) to your 1400 MTU link?
*(This is exactly how the famous Mirai botnet took down half the US internet in 2016).*

Today, you would survive this thanks to two massive walls:
1.  **CoPP (Control Plane Policing):** A Cisco feature that protects the router's brain. *"Dear CPU, spend a max of 5% of your power on fragmentation. Drop the rest ruthlessly."*
2.  **Scrubbing Centers (Traffic Laundromats):** ISPs see the unnatural wave of fragmentation traffic, reroute it to a "laundromat", filter out the smart fridge garbage, and send only clean traffic to your router.