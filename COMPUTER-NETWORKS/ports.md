# 🔍 Testing TCP Connectivity with Telnet (Cisco)

On Cisco devices, the `telnet` command is a highly effective tool for testing TCP port connectivity. Despite its name, in this context, it has nothing to do with the actual Telnet protocol. 

**💡 When is it useful?**
Instead of spinning up a virtual machine and running tools like `nmap`, you can perform a quick preliminary connectivity test directly from your network device.

### 🛠️ Configuration Example
Below is an example of testing two different ports (8305 and 443) using a specific source interface:

```text
KRK-LAB-CORE-1#telnet 3.65.12.175 8305 /source-interface vlan 185
Trying 3.65.12.175, 8305 ...
% Connection timed out; remote host not responding

KRK-LAB-CORE-1#telnet 3.65.12.175 443 /source-interface vlan 185
Trying 3.65.12.175, 443 ... Open