# 🔍 Testing TCP Connectivity with Telnet (Cisco)

On Cisco devices, the `telnet` command is a highly effective tool for testing TCP port connectivity. Despite its name, in this context, it has nothing to do with the actual Telnet protocol. 

**💡 When is it useful?**
Instead of spinning up a virtual machine and running tools like `nmap`, you can perform a quick preliminary connectivity test directly from your network device.

### 🛠️ Configuration Example
Below is an example of testing two different ports (8305 and 443) using a specific source interface:

<pre style="background-color: #000000; color: #e6e6e6; padding: 15px; font-size: 16px; border-radius: 8px; border: 1px solid #444;">
KRK-LAB-CORE-1#telnet 3.65.12.175 8305 /source-interface vlan 185
Trying 3.65.12.175, 8305 ...
% Connection timed out; remote host not responding

KRK-LAB-CORE-1#telnet 3.65.12.175 443 /source-interface vlan 185
Trying 3.65.12.175, 443 ... Open
</pre>

### 📊 Possible Responses & Troubleshooting

| Status | Description & Troubleshooting |
| :--- | :--- |
| 🟢 **Open** | The connection is successful. The port is open and a service is actively listening. |
| 🔴 **Connection timed out** | Traffic is likely being dropped by a firewall along the path. SYN packets are sent, but no SYN-ACK is received. It may also indicate that the destination IP is completely unreachable. |
| 🟡 **Connection refused** | The host is reachable, but there is no service listening on that specific port, or the server is actively rejecting the connection. |
| 🟠 **Destination unreachable** | This message comes from an intermediate device (like a router). It usually indicates a routing issue (e.g., no route to host) or an ACL/firewall rule explicitly blocking the traffic. |