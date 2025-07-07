# Hyper-V Lab: Network Address Translation (NAT) Configuration on DC01

## Purpose

This guide details the steps to configure Network Address Translation (NAT) on `DC01` (your primary Domain Controller) within your Hyper-V lab environment. This setup allows your internal lab virtual machines (`DC02`, `FS01`, `StanlyPC`, etc.) to access the internet through `DC01`, which acts as a gateway/router.

## Prerequisites

Before starting this NAT configuration, ensure the following steps from the main lab build are completed:

* **Hyper-V Host:**
    * `External_Internet_Switch` is created in Hyper-V Manager and connected to your host's physical network adapter (e.g., Wi-Fi or Ethernet).
    * `Internal_Lab_Switch` is created in Hyper-V Manager as an "Internal network."
* **DC01 Virtual Machine:**
    * `DC01` VM is created, Windows Server is installed, and it is configured with:
        * Static IP `192.168.10.1` on its internal network adapter (connected to `Internal_Lab_Switch`).
        * Promoted to a Domain Controller for `mylabs.com`.
        * Active Directory Domain Services (AD DS), DNS, and DHCP roles installed and configured.
    * **Crucially:** `DC01` must have **two network adapters**:
        1.  One connected to `Internal_Lab_Switch` (IP `192.168.10.1`).
        2.  A **second network adapter** connected to `External_Internet_Switch`. (This adapter typically obtains an IP via DHCP from your home router, e.g., `192.168.1.x`).

## Step-by-Step Configuration

These steps are performed on the `DC01` virtual machine.

### 1. Verify Second Network Adapter

Ensure the second network adapter (connected to `External_Internet_Switch`) is present and configured.

1.  Log in to `DC01` as `mylabs\Administrator`.
2.  Right-click **Start** > **Network Connections** > **"Change adapter options"** (or type `ncpa.cpl` in Run).
3.  You should see two adapters:
    * One named "Ethernet" (or similar) with IP `192.168.10.1` (connected to `Internal_Lab_Switch`).
    * Another adapter (e.g., "Ethernet 2") that should be receiving an IP address from your home router (e.g., `192.168.1.x`). This is your internet-facing adapter.
4.  Note the names of these adapters for clarity during configuration.

### 2. Install Routing and Remote Access Role

This role provides the NAT functionality.

1.  Open **Server Manager** on `DC01`.
2.  Click **"Manage"** > **"Add Roles and Features."**
3.  Follow the wizard:
    * "Before You Begin": Next
    * "Installation Type": "Role-based or feature-based installation" > Next
    * "Server Selection": `DC01` should be selected > Next
    * "Server Roles": Expand **"Network Policy and Access Services"** and check **"Routing and Remote Access."**
        * Click **"Add Features"** if prompted for required features.
    * Click **Next** (through Features).
    * "Network Policy and Access Services (Information Page)": Click **Next**.
    * "Confirmation": Review the roles to be installed (should show "Routing and Remote Access"). Click **"Install."**
4.  Wait for the installation to complete. You do **not** need to promote/configure immediately from Server Manager.

### 3. Configure NAT using Routing and Remote Access Console

This configures `DC01` to act as your internet gateway.

1.  After the role installation is complete, open **Routing and Remote Access** (Tools > Routing and Remote Access in Server Manager, or type `rrasmgmt.msc` in Run).
2.  In the console, in the left pane, right-click on your server's name (e.g., `DC01 (local)`).
3.  Select **"Configure and Enable Routing and Remote Access."**
4.  Follow the wizard:
    * **Welcome:** Click **Next**.
    * **Configuration:** Select **"Network address translation (NAT)."** Click **Next**.
    * **Internet Connection:** From the list, select the network adapter that is connected to your **`External_Internet_Switch`**. This is your **public interface** (e.g., "Ethernet 2" with your home router IP). Click **Next**.
    * **Private Connection:** From the list, select the network adapter that is connected to your **`Internal_Lab_Switch`**. This is your **private interface** (e.g., "Ethernet" with IP `192.168.10.1`). Click **Next**.
    * **Completing the Wizard:** Review the summary. Click **Finish**.

The Routing and Remote Access service will start. The server name in the console (`DC01 (local)`) should now have a green arrow indicating it's running.

## Verification

To ensure NAT is working on `DC01` and providing internet access to your internal network:

1.  **Verify DC01 itself has internet access:**
    * On `DC01`, open a web browser (e.g., Edge) and try to navigate to an external website like `google.com`.
2.  **Verify internal client internet access:**
    * Ensure a client VM (like `StanlyPC`) is running, joined to the domain, and configured to obtain an IP address automatically (DHCP).
    * On `StanlyPC`, open Command Prompt (Admin).
    * Run `ipconfig /all` to confirm it received an IP from `DC01`'s DHCP, and `192.168.10.1` as its Default Gateway and DNS server.
    * Run `ping 8.8.8.8` (successful ping means internet access via IP).
    * Run `ping google.com` (successful ping means internet access via DNS resolution through `DC01`).
    * Open a web browser on `StanlyPC` and navigate to `google.com`.

## Troubleshooting Tips

* **`rrasmgmt.msc` not found:** If the console doesn't open after installing the role, ensure you have sufficient GUI management components. Sometimes, installing the "Remote Desktop" role (even if just the core parts) can resolve this by providing necessary graphical dependencies.
* **Wizard skips Private Connection selection:** If the wizard jumps directly to "Finish" after selecting the Public interface, try disabling and then re-enabling Routing and Remote Access (`rrasmgmt.msc` -> Right-click server -> "Disable Routing and Remote Access" -> Then right-click again -> "Configure and Enable Routing and Remote Access"). This often clears a stuck state.
* **No internet access on clients:**
    * Double-check the DHCP scope options on `DC01` (Default Gateway and DNS servers pointing to `192.168.10.1`).
    * Verify `DC01` itself has internet access.
    * Check Windows Firewall on `DC01` (though RRAS usually configures this automatically).
    * Ensure the correct network adapters were selected for Public and Private interfaces during NAT configuration.
