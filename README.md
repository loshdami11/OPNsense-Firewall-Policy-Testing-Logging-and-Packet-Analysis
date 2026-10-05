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


### Part F: Correlate with Wireshark Packet Captures

Traffic generation tests were performed on `icdfa-nslab-client-v1` (`10.10.10.131`) while sniffing on `enp0s3`:

1. **Blocked ICMP (`icmp && ip.addr == 1.1.1.1`):**
   - **Observation:** 4 outbound `Echo (ping) request` packets generated toward `1.1.1.1`.
   - **Result:** No `Echo reply` packets returned. Confirms firewall rule `LAB2 BLOCK ICMP TO 1.1.1.1` dropped packets on ingress.

2. **Blocked HTTP (`tcp.dstport == 80`):**
   - **Observation:** Initial `[SYN]` packet sent to port 80 followed by continuous `[TCP Retransmission]` attempts.
   - **Result:** Absence of `[SYN, ACK]` or `[RST]` frames confirms OPNsense silently dropped HTTP traffic, leading to client connection timeout.

3. **Permitted HTTPS (`tcp.port == 443`):**
   - **Observation:** Successful TCP handshake followed by active `TLSv1.2` / `TLSv1.3` encrypted application data transport.
   - **Result:** Confirms outbound HTTPS traffic passed through unhindered by rule evaluation.

<img width="940" height="685" alt="image" src="https://github.com/user-attachments/assets/d39a05db-b813-46a1-b0b3-52843a24f8c3" />

<img width="1057" height="676" alt="image" src="https://github.com/user-attachments/assets/00c1884b-5271-4dad-b25d-8ddc29f8fb8f" />


<img width="936" height="675" alt="image" src="https://github.com/user-attachments/assets/3a18d63e-f902-41de-baa8-3a1dfbd7b28c" />



### Part G: Inspect States and Automatic NAT

1. **State Table Inspection (`Firewall -> Diagnostics -> States`):**
   - Filtered active firewall connection states by the client IP address (`10.10.10.131`).
   - Verified active state tracking entries for outbound HTTPS connections:
     - **LAN Ingress State:** `Source: 10.10.10.131:<src_port>` -> `Destination: <dest_ip>:443`
     - **WAN Egress (NAT) State:** `Source: 10.0.2.15:<src_port>` (`NAT: 10.10.10.131`) -> `Destination: <dest_ip>:443`
   - **Analysis:** Demonstrates OPNsense's stateful packet inspection mechanism dynamically mapping internal client connections to translated WAN states for bidirectional session tracking.

2. **Outbound NAT Mode Verification (`Firewall -> NAT -> Source NAT` / `Outbound`):**
   - Verified that the system mode is set to **Automatic Source NAT rule generation**.
   - **Analysis:** Confirms that OPNsense automatically creates translation rules for RFC 1918 private subnets (`10.10.10.0/24`), substituting the private client source IP with the WAN interface IP (`10.0.2.15`) to enable outbound internet routability.
  
<img width="1033" height="748" alt="image" src="https://github.com/user-attachments/assets/88cbcaa5-07b2-44b1-9433-98d80a868719" />


### Part H: Restore the Laboratory

1. **Rule Deactivation (`Firewall -> Rules -> LAN`):**
   - Disabled `LAB2 BLOCK ICMP TO 1.1.1.1` and `LAB2 BLOCK OUTBOUND HTTP` rules without deleting them.
   - Applied the firewall policy changes.

2. **Restored Connectivity Verification:**
   - `ping -c 4 1.1.1.1` -> **0% packet loss (Success)**
   - `curl --max-time 10 -I http://example.com` -> **HTTP 200 OK (Restored)**
   - `curl --max-time 10 -I https://example.com` -> **HTTP 200 OK (Allowed)**

**Conclusion:** Disabling the top-level block rules restored full unrestricted connectivity via the lower `Default allow LAN to any` rule.

<img width="418" height="250" alt="image" src="https://github.com/user-attachments/assets/1f5ad3b2-79c9-434c-a918-6c44797bb797" />


Analysis Questions & Answers
38. Why must the specific block rules be placed above the broad allow rule?
Firewalls evaluate rules sequentially using a top-down, first-match policy. The firewall processes traffic by checking each incoming packet against the list of rules from top to bottom and applies the action (Allow/Block) of the first matching rule. If a specific block rule were placed below a broad default allow rule, all matching traffic would hit the allow rule first and be permitted before ever reaching the block rule. Specific restrictive rules must sit at the top to evaluate and enforce security exceptions first.


39. Which five packet attributes are most useful when explaining a firewall decision?
The five key packet attributes used by a stateless/stateful firewall tuple to evaluate and filter network traffic are:

Source IP Address (identifies originating device).

Destination IP Address (identifies target host or network).

Protocol (e.g., TCP, UDP, ICMP).

Destination Port Number (identifies target service, e.g., Port 80 for HTTP vs. Port 443 for HTTPS).

Interface / Direction (identifies entry or exit interface, e.g., LAN ingress vs. WAN egress).


40. Why did blocking ICMP not block HTTPS?
ICMP and HTTPS operate on completely separate network protocols and OSI layer functions. ICMP is an IP-layer control/diagnostic protocol (Layer 3) used for network utilities like ping. HTTPS is an application service running over TCP (Layer 4) on destination port 443. Because the firewall rule specifically targeted the ICMP protocol to destination IP 1.1.1.1, TCP traffic destined for port 443 did not match the block criteria and fell through to be permitted by the default allow rule.


41. What difference did you observe between the blocked TCP port 80 traffic and permitted TCP port 443 traffic?Blocked TCP Port 80: The client sent initial TCP [SYN] packets, but OPNsense silently dropped them without returning a response (RST or SYN-ACK). In Wireshark, this resulted in repeated [TCP Retransmission] frames until the client reached its connection timeout.   Permitted TCP Port 443: The client and server completed a full TCP 3-way handshake (SYN $\rightarrow$ SYN-ACK $\rightarrow$ ACK), followed immediately by active encrypted TLS handshake and application data exchange


42. What role does outbound NAT play when icdfa-nslab-client-v1 uses a private IPv4 address?
The client VM uses a non-routable RFC 1918 private IPv4 address (10.10.10.131) which cannot travel across the public Internet. Outbound Source NAT (Network Address Translation) rewrites the source address of outgoing packets from the private LAN IP (10.10.10.131) to the public/routable WAN IP address of the OPNsense firewall (10.0.2.15). It tracks these connections in its state table so that returning Internet responses can be mapped and routed back to the internal client.


43. Why is restoring the original state an important part of a controlled security laboratory?
Restoring the laboratory to its baseline configuration is critical for several reasons:

Ensures Repeatability: Leaves the environment in a clean, predictable starting state for future tests or other users.

Prevents Operational Disruption: Lingering restrictive block rules can unexpectedly break connectivity or cause false diagnostic errors in subsequent exercises.

Validates Reversibility: Confirms that security changes can be safely rolled back without causing persistent system failure.













