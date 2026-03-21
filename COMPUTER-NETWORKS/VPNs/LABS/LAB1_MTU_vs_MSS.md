# 🔬 LAB: IPsec Fragmentation & MSS Adjustment on Cisco FTD

The goal of this experiment is to intentionally trigger IP fragmentation on Firepower (FPR) firewalls and observe the behavior on the endpoints (two Windows VMs running Wireshark 🦈).

<div align="center">
  <img src="IMAGES/TOPO_VPN_S2S_VTI.png" width="200%">
</div>

In this topology, we have an IPsec Site-to-Site VPN built on VTI using IKEv2 with an **AES-GCM** proposal. This is important because AES-GCM significantly reduces the header overhead. 

The physical interface MTU is 1500 bytes. Based on this, `Tunnel1` is created with an MTU of 1445 bytes.

<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 14px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
FPR# show interface Tunnel1 | include MTU
  Hardware is Virtual Tunnel, MAC address N/A, MTU 1445
      IPsec MTU Overhead : 55
</pre>

> **💡 Note on Overhead:** We were always taught that IPsec overhead is around 72 bytes. However, because we are using AES-GCM, the overhead is only 55 bytes. AES-GCM is an absolute game-changer today.

---

### 1️⃣ MSS Adjustment on Firepower (The "Eraser" Effect)

By default, FTDs have TCP MSS adjustment enabled. This means that when they see a TCP SYN packet passing through with a specific proposed MSS, they take a virtual eraser, rub out the original value, and paste their own calculated MSS.

To prove this theory, I ran some tests. I initiated an `iperf` session from Computer B (IP: `192.168.99.10`). In Wireshark running on Computer B, we can clearly see the MSS value we are trying to negotiate:

<div align="center">
  <img src="IMAGES/Transmission_comp_A.png" width="115%">
</div> 

As seen in the capture, Computer B tries to negotiate an MSS of **1460**, but receives a SYN-ACK with an MSS of **1380**. 

Okay, we could argue that maybe Computer A (our server) somehow just prefers 1380. So, let's look at the incoming traffic on Computer A's Wireshark. The incoming SYN from Computer B arrives with an MSS of 1380! The firewall clearly erased the 1460 and injected 1380. 

But wait... why is the return SYN-ACK from Computer A set to 1460? If Computer A sees an incoming request for 1380, why does it reply with 1460?

#### 🧠 Deep Dive: The TCP 3-Way Handshake & Independent Pipes

Let's go back to the TCP protocol for a moment.

The SYN received from Computer B says *only and exclusively*: **"Please send packets through this pipe to me with an MSS of 1380, because my network card cannot process any more."**

Computer A replies: **"Alright, if that's what you want, I'll do it -> ACK flag. However, please send packets to ME with an MSS of 1460, because I have a better card."** So Computer A sends a SYN with 1460, which together makes a SYN-ACK.

As you can see, there are 2 independent pipes here. In one pipe, packets fly with an MSS of 1380, and in the other, packets fly with 1460. At the very end, we just have an ACK, meaning Computer B agrees to that 1460.

But what if one host is downloading and the other is only sending acknowledgments? Do we only need such a large MSS for one of them? Will one pipe be thinner than the other? **Absolutely not.**

The MSS value has absolutely nothing to do with what the application intends to do (whether it will download 100 GB of movies or just send one email).

MSS is rigidly tied to the hardware (the network card). The operating system does not ask the web browser: *"Hey, are you going to download a lot?"*. The operating system only looks at the network card and says: *"The card has an MTU of 1500, so my MSS is 1460. End of discussion."*

**When are the pipes ACTUALLY asymmetric?**
Only when the hardware or network on one side is worse. 
*Example:* You are sitting in the office on a fiber connection (MTU 1500 -> MSS 1460). Your colleague is sitting on a train using an old LTE modem with a VPN enabled (MTU 1340 -> MSS 1300). In that case, the pipes will be physically asymmetric (1460 one way, 1300 the other), regardless of who is downloading the file from whom.

Returning to our topology: the 1460 MSS transmission from Computer A gets changed by FPR-A to 1380. So, one FPR uses the eraser for one host, and the other FPR does it for the second host. The computers have no idea who changed it. They think they negotiated it themselves!

---

### 💥 The Attempt to Break It (FlexConfig)

But let's return to the attempt to disable this pesky MSS adjustment so the firewall stops erasing it. This would allow us to force fragmentation and overload the hardware. 

I looked for this option in the FMC (Firepower Management Center) GUI and, as it turns out, it doesn't exist anywhere. So, I came up with the idea to bypass FMC and inject it directly into the underlying legacy ASA engine (the LINA process). I found out it is possible via **FlexConfig** using the `sysopt connection tcpmss` command.

<div align="center">
  <img src="IMAGES/FlexConfig_MSS.png" width="800">
</div>

However, I got an error. As it turned out later, it cannot be done because the Tunnel MTU is calculated dynamically based on the physical MTU, and the MSS is calculated based on that Tunnel MTU. Even if we change the physical MTU, the tunnel MTU will just adapt automatically. 

**Conclusion:** You cannot force this behavior on these specific FTD devices. I will probably manage to do it when I test this on standard IOS-XE routers later.

---

### 2️⃣ Triggering Fragmentation via UDP (iperf)

*(Work in progress...)*