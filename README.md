### OPNsense-Firewall-Policy-Testing-Logging-and-Packet-Analysis

### Part A  Record the baseline

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

<img width="380" height="203" alt="image" src="https://github.com/user-attachments/assets/bb0ba266-3507-4ae0-9b00-bd1a000cfd68" />


1. Configured an outbound TCP block rule in OPNsense (**Firewall -> Rules -> LAN**):
   - **Action:** Block | **Interface:** LAN | **Direction:** In
   - **Protocol:** TCP | **Source:** LAN net | **Destination:** Any
   - **Destination Port Range:** HTTP (80)
   - **Log:** Enabled | **Description:** `LAB2 BLOCK OUTBOUND HTTP`
2. Positioned the rule above `Default allow LAN to any` and applied changes.
3. Verified protocol-specific filtering from `icdfa-nslab-client-v1`:
   - `curl --max-time 10 -I http://example.com` -> **Timed out after 10000ms (Blocked on Port 80)**
   - `curl --max-time 10 -I https://example.com` -> **HTTP/2 200 OK (Allowed on Port 443)**
  
### Part E: Examine Firewall Logs

1. Navigated to **Firewall -> Log Files -> Live View** in the OPNsense Web GUI.
2. Filtered logs by the rule description label `LAB2` to isolate block events.
3. Inspected the detailed rule modal for blocked ICMP packets:
   - **Timestamp:** 2026-10-05T11:40:45
   - **Interface:** em1 (LAN)
   - **Source IP:** 10.10.10.131
   - **Destination IP:** 1.1.1.1
   - **Protocol:** ICMP (IP Protocol 1)
   - **Action:** Block
   - **Rule Label:** LAB2 BLOCK ICMP TO 1.1.1.1

#### Rule Evaluation Analysis (Step 16)
The outbound ICMP traffic matched the custom `LAB2 BLOCK ICMP TO 1.1.1.1` rule instead of the `Default allow LAN to any` rule because OPNsense processes firewall filtering using a **top-down, first-match policy evaluation strategy**. 

Because the custom block rule was positioned above the default allow rule in the rule stack, packets originating from `10.10.10.131` destined for `1.1.1.1` triggered an immediate match on the block criteria first. OPNsense executed the `block` action, logged the event, and dropped the traffic without evaluating any lower rules.

<img width="585" height="499" alt="image" src="https://github.com/user-attachments/assets/7898d060-ce57-424c-8946-9a642b9ed6ae" />


<img width="613" height="517" alt="image" src="https://github.com/user-attachments/assets/79640725-cf61-4651-b4ad-21b833bc211f" />















