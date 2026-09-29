# Active Directory Mini-Office Lab: Technical Setup & Administrator Runbook

This guide contains the exact step-by-step procedure used to deploy, configure, and troubleshoot the Windows Server 2022 and Windows 11 Enterprise Active Directory lab environment.

---

## Technical Specifications & Architecture

- **Host OS:** Windows 10/11 64-bit
- **Hypervisor:** Oracle VM VirtualBox 7.x
- **Virtual Network:** Internal Network (`intnet`) with Promiscuous Mode enabled
- **Domain Name:** `lab.local` (NetBIOS: `LAB`)
- **Domain Controller (`DC-01`):** Windows Server 2022 Standard (Desktop Experience)
  - IP Address: `10.0.2.10` / Subnet: `255.255.255.0`
  - Roles: AD DS, DNS Server, DHCP Server, SMB File Host
- **Client Workstation (`WIN10-CLI`):** Windows 11 Enterprise
  - IP Address: `10.0.2.50` (Static / DHCP Scope: `10.0.2.100 - 10.0.2.200`)
  - Preferred DNS: `10.0.2.10`

---

## Phase 1: Virtual Machine Setup & Network Isolation

### 1. Create Server VM (`DC-01`)
1. Open VirtualBox → Click **New**.
2. **Name:** `DC-01` | **Type:** Windows 2022 (64-bit).
3. **ISO Image:** Select the Windows Server 2022 Evaluation ISO.
4. **CRITICAL:** Check the box **"Skip Unattended Installation"** (prevents missing license terms errors).
5. **Hardware:** Assign **4096 MB RAM** and **2 CPU cores**.
6. **Hard Disk:** Create a dynamic VDI of **50 GB**.

### 2. Create Client VM (`WIN10-CLI`)
1. Click **New** → **Name:** `WIN10-CLI` | **Type:** Windows 11 (64-bit).
2. **ISO Image:** Select the Windows 11 Enterprise Evaluation ISO.
3. Check **"Skip Unattended Installation"**.
4. **Hardware:** Assign **4096 MB RAM** and **2 CPU cores**.
5. **Hard Disk:** Create a dynamic VDI of **80 GB**.

### 3. Configure Internal Virtual Networking
*(Avoid VirtualBox NAT Network issues on Windows 11 hosts by using direct internal bridging)*

1. Power off both VMs.
2. Go to **`DC-01` Settings** → **Network** → **Adapter 1**:
   - **Attached to:** `Internal Network`
   - **Name:** `intnet`
   - Expand **Advanced** → **Promiscuous Mode:** Set to **`Allow All`**.
3. Go to **`WIN10-CLI` Settings** → **Network** → **Adapter 1**:
   - **Attached to:** `Internal Network`
   - **Name:** `intnet` *(Must match DC-01 exactly)*
   - Expand **Advanced** → **Promiscuous Mode:** Set to **`Allow All`**.

---

## Phase 2: Domain Controller Configuration & Promotion

### 1. Set Static IP Address on `DC-01`
1. Boot `DC-01` and complete Windows Server installation.
2. Press `Win + R`, type `ncpa.cpl`, and hit **Enter**.
3. Right-click **Ethernet** → **Properties** → Double-click **Internet Protocol Version 4 (TCP/IPv4)**.
4. Configure:
   - **IP Address:** `10.0.2.10`
   - **Subnet Mask:** `255.255.255.0`
   - **Default Gateway:** *Leave Blank*
   - **Preferred DNS Server:** `127.0.0.1` (or `10.0.2.10`)
5. Press `Win + R`, type `sysdm.cpl`, click **Change...**, rename computer to **`DC-01`**, and restart.

### 2. Install Active Directory DS & Promote Forest
1. Open **Server Manager** → **Add Roles and Features**.
2. Select **Active Directory Domain Services** and **DNS Server** → Complete wizard.
3. Click the **Notification Flag** at the top right → Click **Promote this server to a domain controller**.
4. Select **Add a new forest** → **Root domain name:** `lab.local`.
5. Enter a DSRM password, accept default paths, and click **Install**. The server will automatically reboot.

### 3. Install & Authorize DHCP Server
1. Open **Server Manager** → **Add Roles and Features** → Add **DHCP Server**.
2. Click the **Notification Flag** → Click **Complete DHCP configuration** → Commit credentials (`LAB\Administrator`).
3. Open DHCP Console (`dhcpmgmt.msc`).
4. Right-click `dc-01.lab.local` → Click **Authorize** (Icon turns green upon refresh).
5. Right-click **IPv4** → **New Scope**:
   - **Name:** `Office Scope`
   - **Start IP:** `10.0.2.100` | **End IP:** `10.0.2.200` | **Subnet Mask:** `255.255.255.0`
   - **Router (Gateway):** `10.0.2.1`
   - **DNS Server:** `10.0.2.10`
6. Right-click `Scope [10.0.2.0]` → Click **Activate**.

---

## Phase 3: Active Directory Objects & Storage Provisioning

### 1. Build Organizational Units & Users
1. Open **Active Directory Users and Computers** (`dsa.msc`).
2. Right-click `lab.local` → **New** → **Organizational Unit** → Name: `company_OUs`.
3. Inside `company_OUs`, create sub-OUs: `IT_OU`, `Sales_OU`, `HR_OU`.
4. Inside `company_OUs`, create Global Security Groups: `SG_IT`, `SG_Sales`, `SG_HR`.
5. Create test user accounts:
   - **Sales:** `aaron` (`aaron as. smith`) -> Add to `SG_Sales`.
   - **IT:** `john` (`john jd. doe`) -> Add to `SG_IT`.
   - **HR:** `mary` (`mary ms. sue`) -> Add to `SG_HR`.

### 2. Configure SMB Department File Share
1. Open **File Explorer** on `DC-01` → Navigate to `C:\`.
2. Create folder `CompanyData` → Inside, create folder `Sales_Data`.
3. Right-click `Sales_Data` → **Properties** → **Sharing** tab → **Advanced Sharing...**:
   - Check **Share this folder** | Set Share Name to `Sales_Data$`.
   - Click **Permissions**: Remove `Everyone`, Add `SG_Sales`, grant **Full Control**.
4. Switch to **Security** (NTFS) tab → Click **Edit...**:
   - Add `SG_Sales` group → Grant **Modify** permissions → Click **Apply**.

---

## Phase 4: Group Policy Object (GPO) Deployment

Open **Group Policy Management** (`gpmc.msc`):

### 1. Account Lockout Policy (Domain Level)
1. Expand **Domains** → `lab.local` → Right-click **Default Domain Policy** → **Edit**.
2. Navigate to: `Computer Configuration` → `Policies` → `Windows Settings` → `Security Settings` → `Account Policies` → `Account Lockout Policy`.
3. Configure **Account lockout threshold:** Set to `3` invalid logon attempts.

### 2. Department Drive Mapping GPO
1. Expand **Domains** → `lab.local` → Right-click `Sales_OU` → **Create a GPO in this domain, and Link it here...**
2. Name: `GPO_Sales_Drive` → Right-click → **Edit**.
3. Navigate to: `User Configuration` → `Preferences` → `Windows Settings` → `Drive Maps`.
4. Right-click panel → **New** → **Mapped Drive**:
   - **Action:** Update
   - **Location:** `\\DC-01\Sales_Data$`
   - Check **[X] Reconnect**
   - **Label as:** `Sales Share`
   - **Drive Letter:** `Use: S`

### 3. Control Panel Restriction GPO
1. Right-click `Sales_OU` → Create & Link GPO: `GPO_Block_Control_Panel`.
2. Navigate to: `User Configuration` → `Policies` → `Administrative Templates` → `Control Panel`.
3. Double-click **Prohibit access to Control Panel and PC settings** → Set to **Enabled**.

---

## Phase 5: Client Onboarding & Verification

### 1. Configure Workstation & Join Domain
1. Boot `WIN10-CLI`. On the Windows 11 setup screen, press `Shift + F10` → Type `regedit` → Navigate to `HKLM\SYSTEM\Setup` → Create Key `LabConfig` → Add DWORD `BypassTPMCheck = 1` and `BypassSecureBootCheck = 1` to bypass Win 11 installation blocks.
2. Complete OOBE by selecting **"I don't have internet"** → **"Continue with limited setup"** → Create local user `localadmin`.
3. In Windows 11, press `Win + R` → `ncpa.cpl` → Set IPv4 properties manually:
   - **IP:** `10.0.2.50` | **Subnet:** `255.255.255.0` | **DNS:** `10.0.2.10`
4. Press `Win + R` → `sysdm.cpl` → Click **Change...** → Select **Domain:** `lab.local`.
5. Enter admin credentials (`LAB\Administrator` + Password) → Confirm welcome message → Restart PC.

### 2. End-User Login & Policy Testing
1. Log into `WIN10-CLI` as domain user `LAB\aaron`.
2. **Verify Mapped Drive:** Open File Explorer → Confirm network location **`Sales Share (S:)`** is present and accessible.
3. **Verify Control Panel Block:** Press `Win + R` → Type `control` → Confirm restriction block message appears.

### 3. Helpdesk Operations (Lockout & Unlock Workflow)
1. On `WIN10-CLI`, sign out.
2. Enter an incorrect password for `aaron` 3 consecutive times.
3. Confirm the client displays: *"The referenced account is currently locked out and may not be logged on to."*
4. Switch to `DC-01` → Open `dsa.msc` → Open properties for `aaron` → **Account** tab.
5. Check **`[X] Unlock account. This account is currently locked out...`** → Click **Apply**.
6. Return to `WIN10-CLI` and log back in successfully with the correct password.

---

## Phase 6: Post-Mortem & Troubleshooting Reference

| Observed Issue | Root Cause Analysis | Corrective Action Applied |
| :--- | :--- | :--- |
| VirtualBox installer throws `Software License Terms` error | VirtualBox auto-enabled Unattended Installation mode | Recreated VM checking **"Skip Unattended Installation"** |
| Client receives APIPA IP (`169.254.x.x`) / DHCP timeout | VirtualBox host NAT driver dropping L2 broadcast frames | Converted both VMs to **Internal Network** (`intnet`) with **Promiscuous Mode: Allow All** |
| User login stuck on Windows 11 spinning screen | Default AD flag `User must change password at next logon` hangs local domain auth | Unchecked forced password reset option in AD, set `Password never expires` |
| Account fails to lock out after multiple bad passwords | Default Domain Policy lockout threshold set to `0` (disabled) | Configured Account Lockout Threshold to `3` in `gpmc.msc` and executed `gpupdate /force` |
