# Multi-Zone Enterprise Network & Hardened Identity Sandbox

<!-- Short, high-impact tagline -->
A production-modeled virtual enterprise infrastructure leveraging a Netgate pfSense firewall to segment traffic across four isolated security zones, managed centrally via Active Directory Domain Services (AD DS).

---

## 📊 Network Architecture & Schema
<!-- High-level visual topology overview -->
![Network Topology](network-topology.png)

| Interface | Zone Name | Network Schema | Trust Level | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **`em0`** | **WAN** | DHCP (Bridged) | Untrusted (0) | External internet access and update path. |
| **`em1`** | **LAN_ZONE** | `10.0.20.1/24` | Low-Trusted (1) | General user workstations and endpoints. |
| **`em2`** | **CORPORATE_ZONE**| `10.0.10.1/24` | High-Trusted (3) | Hosts domain controllers and core infrastructure. |
| **`em3`** | **DMZ_ZONE** | `10.0.30.1/24` | Semi-Trusted (2) | Isolated perimeter for public-facing servers. |

---

## 🚀 Core Capabilities & Features Built
* **Network Segmentation:** Zero flat-networking. All inter-subnet traffic passes through the pfSense packet inspection engine.
* **Centralized Identity & Access Management:** Domain-wide authentication controlled by Windows Server 2022 Active Directory.
* **Automated Endpoint Hardening:** Group Policy Objects (GPOs) applied to enforce system-level pre-login warning banners.
* **Isolated SOC Operations:** A dedicated Analyst Workstation in a unidirectional management segment to securely audit the network.

---

## 🛠️ Implementation Phases (Expand for Details)

<!-- Collapsible sections keep your profile sleek and professional -->

<details>
<summary><b>Phase 1: Virtualization & Resource Provisioning</b></summary>

### Objective
Establish a stable virtual hardware baseline capable of running nested operating systems concurrently without host resource starvation.

### Execution Steps
1. Audited host machine specifications to ensure robust resource allocation (32GB RAM host overhead).
2. Provisioned independent VM hardware shells within Oracle VirtualBox:
   * *pfSense Firewall:* 1GB RAM, 1 vCPU.
   * *Windows Server 2022:* 4GB RAM, 2 vCPUs.
   * *Windows 11 Client:* 4GB RAM, 2 vCPUs.
3. Sourced and mounted official evaluation ISO media for Netgate pfSense 2.7.2, Windows Server 2022, and Windows 11 Enterprise LTSC.
</details>

<details>
<summary><b>Phase 2: Core Network Segmentation & Routing</b></summary>

### Objective
Build physical/logical network boundaries to mandate firewalled inspection for all inter-subnet communications.

### Execution Steps
1. Configured pfSense VM hardware with four independent adapters: Adapter 1 (Bridged), Adapters 2–4 set to unique **Internal Networks** (`LAN_Network`, `Corporate_Network`, `DMZ_Network`).
2. Mapped physical adapters (`em0` through `em3`) to software zone designations (**WAN, LAN, CORPORATE_ZONE, DMZ_ZONE**).
3. Set static gateway IPv4 boundaries on all internal interfaces matching the schema.
4. Disabled native pfSense DHCP on the `CORPORATE_ZONE` to prevent conflicts with Active Directory.
</details>

<details>
<summary><b>Phase 3: Identity Infrastructure & Directory Services (AD DS)</b></summary>

### Objective
Establish centralized authentication, logical asset tracking, and initialize the security management perimeter.

### Execution Steps
1. Statically configured the Windows Server network interface to `10.0.10.10` with its Primary DNS pointing to its local loopback (`127.0.0.1`).
2. Utilized Windows Server Manager to install the **Active Directory Domain Services (AD DS)** role and promoted to a Domain Controller for `corp.local`.
3. Engineered a structured Organizational Unit (OU) hierarchy under `Corporate_Headquarters`: `Finance`, `IT_Security`, and `Workstations`.
4. Joined the Analyst Workstation to the domain and explicitly placed its machine account into the `IT_Security` OU to keep its administrative profile isolated.
5. **Defeated Domain Join DNS Hurdle:** Troubleshot client domain-join failures by manually shifting the Windows 11 Primary DNS away from pfSense and directly targeting the DC at `10.0.10.10`.

![AD OU Hierarchy](screenshots/ad-ou-hierarchy.png)
</details>

<details>
<summary><b>Phase 4: Endpoint Hardening via Group Policy (GPO)</b></summary>

### Objective
Automate and enforce mandatory corporate compliance and pre-login security hardening parameters remotely across domain endpoints.

### Execution Steps
1. Opened Group Policy Management Console on the DC, targeted the `Workstations` OU, and generated a linked object named `Workstation_Security_Policy`.
2. Edited the GPO under **Computer Configuration** to enforce an administrative login banner:
   * *Interactive logon: Message text:* `Authorized Access Only. Property of Corporate.`
   * *Interactive logon: Message title:* `Security Notice`
3. **Resolved GPO Filtering Issue:** Diagnosed why the policy failed to apply by running `gpresult /r` on the client. Migrated the computer object out of the generic `Computers` folder into the `Corporate_Headquarters/Workstations` OU.
4. Forced an immediate update via the client command line: `gpupdate /force`.

![GPO Applied](screenshots/gpo-applied.png)
</details>

<details>
<summary><b>Phase 5: Zone-Based Firewall Hardening & Traffic Control</b></summary>

### Objective
Enforce the Principle of Least Privilege across subnets, completely isolating public zones from the internal domain directory to stop lateral movement.

### Execution Steps
1. Connected to the pfSense WebConfigurator (`http://10.0.20.1`) from an authorized endpoint.
2. Engineered the **DMZ Isolation Rules Matrix** (processed top-to-bottom):
   * **Rule 1 (Management - Pass):** Source: `CORPORATE_ZONE subnets` ➔ Destination: `DMZ_ZONE subnets` (Allows administrative maintenance).
   * **Rule 2 (Lateral Defense - Block):** Source: `DMZ_ZONE subnets` ➔ Destination: `CORPORATE_ZONE subnets` (Stops lateral attacks on the Domain Controller).
   * **Rule 3 (Lateral Defense - Block):** Source: `DMZ_ZONE subnets` ➔ Destination: `LAN_ZONE subnets` (Stops pivots to user workstations).
3. Engineered the **Analyst Workstation Traffic Matrix**:
   * Granted the `IT_Security` subnet outbound scanning capability into the `LAN_ZONE` and `DMZ_ZONE`.
   * Enforced strict block rules preventing standard `LAN_ZONE` machines from initiating connections inward toward the `IT_Security` segment.

![Firewall Rules](screenshots/pfsense-rules.png)
</details>

---

## 🎯 Key Engineering Takeaways
> [!IMPORTANT]
> * **DNS is the AD Lifeline:** Active Directory cannot function without precise DNS mapping. Clients must look directly to the Domain Controller for resolution, not the network gateway.
> * **GPO Link Scope:** Group Policies apply to the container, not the asset itself. Baseline objects must be actively migrated into targeted OUs for policy enforcement.
> * **First-Match Firewall Logic:** Firewalls process rules sequentially. Placing granular administrative "Pass" rules directly above strict "Block All" constraints is required to keep a perimeter secure without losing management capabilities.

---

## 🛠️ Sandbox Roadmap (Future Projects)
- [ ] **SIEM Integration:** Deploy a centralized Wazuh or Elastic agent architecture across the segments to monitor the Domain Controller from the Analyst Station.
- [ ] **Network IDS/IPS:** Activate the **Snort** package within pfSense to perform deep packet inspection on the `DMZ_ZONE`.
- [ ] **Vulnerability Management:** Run automated credentialed scans across the internal subnets using **Nessus Essentials**.
