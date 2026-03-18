# 💾 Cisco Storage History: `flash:` vs `bootflash:`

When performing an IOS upgrade on a Cisco device, you often have to specify the boot path. You might wonder which command is correct:
`Switch(config)# boot system flash:new-image.bin`
or 
`Switch(config)# boot system bootflash:new-image.bin`

**The short answer:** Today, there is absolutely no difference. They are the exact same thing. But the reason *why* both exist is a fascinating piece of networking history.

### 🔍 The Modern Reality (The Alias)

In modern Cisco devices, `flash:` is simply an alias (a shortcut) pointing to the `bootflash:` directory. You can easily prove this in the CLI. Notice how checking both directories yields the exact same output and points to the same root path:

<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 14px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
Switch# dir flash:
Directory of bootflash:/

426029  drwx             4096  Mar 18 2026 14:28:55 +00:00  .ishow
417794  drwx             4096  Mar 18 2026 14:27:35 +00:00  .prst_sync

Switch# dir bootflash:
Directory of bootflash:/

426029  drwx             4096  Mar 18 2026 14:28:55 +00:00  .ishow
417794  drwx             4096  Mar 18 2026 14:27:35 +00:00  .prst_sync
</pre>

So, where did this discrepancy come from?

---

### 🕰️ The 90s & 2000s: The Era of the "Dumb" ROMMON

Back in the day, on heavy-duty, modular routers (like the Cisco 7200 series) and core switches, `flash:` and `bootflash:` were **two physically separate pieces of hardware**.

1.  **`flash:` (The Main Drive):** This was usually a removable PCMCIA card (e.g., 64MB or 256MB). It held the full, heavy IOS image, configurations, and logs. However, early flash memory was expensive and highly prone to failure.
2.  **`bootflash:` (The Rescue Chip):** Because the main flash was unreliable, Cisco soldered a smaller, separate memory chip directly onto the motherboard. 

*(Note: Simpler, fixed-configuration access switches like the legendary Catalyst 2960 never had a soldered `bootflash:` chip. They operated solely on a single `flash:` memory).*

**The "Starter" Problem:**
When you powered on an old modular router, the first thing to wake up was the **ROMMON** (ROM Monitor). The old ROMMON was "dumb". It didn't have the drivers to read complex file systems like FAT16/FAT32 located on the removable `flash:` card. 

So, it reached out to the soldered `bootflash:`, which contained a stripped-down, lightweight "Boot Image". This boot image acted like a car's starter motor. It had just enough drivers to read the main `flash:` card and kickstart the full IOS. 
Additionally, if the main flash died or someone formatted the wrong card, this tiny system in `bootflash:` could bring up a single network interface, allowing the admin to download a new IOS via TFTP.

---

### 🧬 Why does it look different today?

Fast forward to today's modern architecture (like the Catalyst 9000 series). Processors and bootloaders (now heavily based on Linux/U-Boot) are incredibly powerful.

*   **The Smart ROMMON:** Today's ROMMON is a beast. It can natively read FAT32, ext4, and even has a full network stack built-in.
*   **No more "Starter":** Because the modern ROMMON is so smart, it no longer needs that tiny "Boot Image" middleman. It can reach directly into a massive SSD and boot the full OS.

**The Merger:**
Since the physical separation was no longer needed, Cisco simply put everything onto one large internal SSD. Because the system now boots directly from this drive, they officially named it **`bootflash:`**. 

However, to prevent millions of legacy automation scripts and old engineer habits from breaking, they kept **`flash:`** alive as a logical alias.

> **💡 Summary:**
> You can use `flash:` and `bootflash:` interchangeably. It’s just Cisco's way of respecting the past while running on modern hardware.

