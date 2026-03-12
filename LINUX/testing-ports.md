# 🐧 Linux Port Testing & Listening Services

### 1️⃣ Testing Connectivity with Netcat (nc)

The `netcat` (`nc`) tool is an absolute lifesaver for testing TCP/UDP connectivity. Since many network appliances (firewalls, proxies, load balancers) are Linux-based under the hood, this tool is almost always available.

To test if a specific port is open, use the following command. It's highly recommended to use the `-z` flag so the program only checks the connection and immediately exits without hanging.

<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 15px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
user@linux-client:~# nc -zv 198.51.100.56 8443
Connection to 198.51.100.56 8443 port [tcp/*] succeeded!

user@linux-client:~# nc -zv 198.51.100.56 8444
nc: connect to 198.51.100.56 port 8444 (tcp) failed: Connection refused
</pre>

**Legend (Netcat Flags):**
| Flag | Description |
| :--- | :--- |
| `-z` | **Zero-I/O mode.** Used for scanning. It tells `nc` to just scan for listening daemons without sending any actual data. |
| `-v` | **Verbose.** Provides detailed output (shows whether the connection succeeded or failed). |

---

### 2️⃣ Checking Listening Ports (ss)

If you are on a server and want to check which ports are actively listening (waiting for incoming connections), the modern and fastest way is using the `ss` (Socket Statistics) command.

<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 15px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
root@linux-server:~# ss -tulpn | grep LISTEN
tcp   LISTEN 0      128          0.0.0.0:22        0.0.0.0:*    users:(("sshd",pid=855,fd=3))
tcp   LISTEN 0      511          0.0.0.0:80        0.0.0.0:*    users:(("nginx",pid=900,fd=6))
tcp   LISTEN 0      511          0.0.0.0:443       0.0.0.0:*    users:(("nginx",pid=900,fd=7))
</pre>

**Legend (ss Flags - `tulpn`):**
| Flag | Description |
| :--- | :--- |
| `-t` | Show **TCP** sockets. |
| `-u` | Show **UDP** sockets. |
| `-l` | Show only **Listening** sockets (ignores established connections). |
| `-p` | Show the **Process** using the socket (requires `sudo` / root privileges). |
| `-n` | Show **Numeric** addresses (do not resolve IP addresses and port numbers to names like "http" or "ssh"). |

---

### 3️⃣ The Legacy Tool: netstat

Historically, `netstat` was used for this exact purpose. However, it is now considered deprecated and is often missing from fresh Linux installations. If you still need to use it, you must install the `net-tools` package.

<pre style="background-color: #000000; color: #00ff00; padding: 15px; font-size: 15px; border-radius: 8px; border: 1px solid #444; line-height: 1.2;">
# For Ubuntu / Debian:
root@linux-server:~# apt update
root@linux-server:~# apt install net-tools

# For CentOS / RedHat / Fedora:
root@linux-server:~# yum install net-tools
</pre>