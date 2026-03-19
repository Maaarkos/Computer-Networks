# 📦 Cisco IOS-XE: Install Mode vs Bundle Mode (A Practical Test)

Starting with the Catalyst 9000 series, Cisco strongly recommends using **Install Mode** instead of the legacy **Bundle Mode** for running the operating system. 

To prove why this is so important, I conducted a test on a massive **Catalyst C9610R** core switch. The results? In Install Mode, the switch booted almost 2 minutes faster and consumed **1.4 GB less RAM**. On a large scale, this also translates to lower power consumption and better thermal efficiency.

Here is a deep dive into why this happens and how to prove it.

---

### ⚙️ How the Boot Process Works

**1. Bundle Mode (The Legacy Way - `.bin` file)**
In this mode, the switch boots directly from the compressed `.bin` file. The CPU has to sweat:
*   **Read:** The CPU must read the entire 1.5 GB file from the Flash drive.
*   **Copy to RAM:** It must copy the entire file into the RAM (creating a virtual RAM disk—this is why it eats so much memory!).
*   **On-the-fly Decompression:** The CPU must "unzip" this file in the RAM to find the necessary modules (kernel, drivers, IOSd).
*   **Execution:** Only then does it start the system. This wastes both CPU cycles and massive amounts of memory.

**2. Install Mode (The Modern Way - `packages.conf`)**
In this mode, the `.bin` file is pre-extracted onto the Flash drive (using the `install add` command). It sits there as a collection of small, ready-to-use `.pkg` files.
*   **Read:** The CPU reads a tiny text file called `packages.conf` (which acts as a table of contents).
*   **Execution:** The CPU loads *only* the specific `.pkg` files it needs directly into the RAM and runs them instantly. No on-the-fly decompression, no copying gigabytes of garbage.

---

### 🐧 The IOS-XE Architecture (Linux Under the Hood)

Before looking at the test results, it's crucial to understand how modern Cisco devices manage memory.

If you type `show processes memory`, you are **NOT** seeing the total physical RAM of the switch. You are only seeing the memory allocated to the `IOSd` process (the good old Cisco IOS running as a single application). 

A Supervisor 3XL usually has 16 GB or 32 GB of physical RAM. Where is the rest?
The rest is managed by the underlying **Linux Kernel (Host OS)**. It manages containers, databases (e.g., for SD-Access), hardware processes (FED), and buffers.

To see the *actual* physical RAM usage of the entire machine (like Task Manager in Windows), you must use:
`show platform software status control-processor brief`

---

### 📊 The Proof: My CLI Experiment on C9610R

I booted the same switch using both methods and compared the memory footprint. *(Note: The C9610R has two Supervisors, each with a 16-core CPU, as seen in the output).*

#### 1. The IOSd Process Memory (`show processes memory`)
This is the memory used just by the legacy IOS application.
*   **Install Mode:** Used: ~344 MB
*   **Bundle Mode:** Used: ~378 MB
*   **Difference:** An increase of ~34 MB. It's not a lot, but it shows that the IOSd process has to hold more "garbage" related to unpacking the image.

#### 2. The Total Physical Memory (`show platform software...`) - THE SHOCKER! 💥
This is the memory of the entire Linux Host OS.
*   **Install Mode:** Used: 7,297,132 kB **(~6.9 GB / 23%)**
*   **Bundle Mode:** Used: 8,695,004 kB **(~8.3 GB / 27%)**
*   **Difference:** **An increase of 1.4 GB of RAM!** In Bundle Mode, the switch wastes nearly a gigabyte and a half of physical memory just to hold the compressed `.bin` file in a RAM disk.

---

### 🛠️ How to Upgrade using Install Mode

To properly unpack the `.bin` file into `.pkg` files and set the switch to boot from `packages.conf`, you use the `install` command.

**The Standard Command:**
<pre style="background-color: #000000; color: #ffffff; padding: 15px; font-size: 15px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
Switch# install add file bootflash:cisco9k_iosxe.17.18.02.SPA.bin activate commit
</pre>
*Problem:* This command is interactive. During the 15-20 minute upgrade process, it will pause and ask you questions like *"This will reload the device, do you want to proceed? [y/n]"*. If you walk away to get coffee, the upgrade halts until you press 'y'.

**The Pro-Tip Command (Fire and Forget):**
<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 15px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
Switch# install add file bootflash:cisco9k_iosxe.17.18.02.SPA.bin activate commit prompt-level none
</pre>
*Why use this?* The `prompt-level none` flag tells the switch: *"Do the entire process, unpack the files, change the boot variable, and reboot the switch automatically without asking me a single question."* It is an absolute lifesaver, especially when upgrading a stack of switches late at night!