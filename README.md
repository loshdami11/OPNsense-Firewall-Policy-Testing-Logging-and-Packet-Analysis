# OPNsense-Firewall-Policy-Testing-Logging-and-Packet-Analysis

Part A  Record the baseline

I started icdfa-nslab-firewall-v1 first, then start icdfa-nslab-client-v1.

<img width="505" height="393" alt="image" src="https://github.com/user-attachments/assets/e74a0b45-1bcc-44f7-af70-25ef11236354" />

<img width="521" height="386" alt="image" src="https://github.com/user-attachments/assets/8241b8d7-f92a-4ade-971e-bedafed54bf4" />

Part B  Review rule order

<img width="494" height="385" alt="image" src="https://github.com/user-attachments/assets/e14e4df1-10c2-44e4-a563-0355cdd22a3d" />

Reviewed existing rules under Firewall -> Rules -> LAN. OPNsense uses top-down, first-match evaluation. To ensure targeted block rules take effect, they must be positioned above the Default allow LAN to any rule


Part C  Block ICMP to one test address

<img width="431" height="303" alt="image" src="https://github.com/user-attachments/assets/fc1ace7c-89dc-403e-9a1a-8788660e4214" />

1. Created a rule under **Firewall -> Rules -> LAN**:
   - **Action:** Block | **Interface:** LAN | **Direction:** In
   - **Protocol:** ICMP | **Source:** LAN net | **Destination:** 1.1.1.1/32
   - **Log:** Enabled | **Description:** `LAB2 BLOCK ICMP TO 1.1.1.1`
2. Positioned rule above `Default allow LAN to any` and applied changes.
3. Verified protocol-specific filtering:
   - `ping -c 4 1.1.1.1` -> **100% packet loss (Blocked as expected)**
   - `getent hosts example.com` -> **Resolved successfully (Permitted)**
   - `curl -I https://example.com` -> **HTTP/2 200 (Permitted)**
  

Part D  Block outbound HTTP while allowing HTTPS








