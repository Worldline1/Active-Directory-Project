---
layout: default
---

# Introduction

This project simulates a small enterprise network environment focused on cybersecurity monitoring and attack detection.

The lab consists of four virtual machines connected via a virtual switch, which links to a router providing simulated internet access. All machines are part of a custom NAT network (`192.168.10.0/24`):

- **Ubuntu Server** running Splunk Enterprise (SIEM)
- **Windows Server 2022** configured as the Active Directory Domain Controller (ADDC)
- **Windows 10 Pro** endpoint joined to the domain
- **Kali Linux** used for attack simulation

Both Windows machines are configured with **Sysmon** and the **Splunk Universal Forwarder**, and continuously send event logs to the **Ubuntu Splunk server** for centralized collection and analysis.

Simulated attacks are carried out via **Remote Desktop Protocol (RDP)** and **Atomic Red Team**, enabling validation of detections and identification of visibility gaps across the environment.

![AD Diagram](AD-Diagram.png)

# 🖥️ Environments & Configurations

- **Ubuntu Server 24.04.2** – Hosts Splunk Enterprise for log collection and analysis.

- **Windows Server 2022** – Configured as the Active Directory Domain Controller (AD DC), managing domain users and machines.

- **Windows 10 Pro** – Acts as the target endpoint, joined to the domain. Sysmon and Splunk Universal Forwarder are installed to collect and forward event logs to the Splunk server.  
  Atomic Red Team is also deployed on this machine to simulate adversarial behavior and test detection coverage.

- **Kali Linux** – Used as the attacker machine to simulate intrusions and generate activity on the target system.

All machines are connected through a custom NAT network (`192.168.10.0/24`), allowing isolated communication between them for testing purposes.

Below is a summary table of the machines, their roles, and assigned IP addresses:

| Machine                | Role                     | IP Address         |
|------------------------|--------------------------|--------------------|
| Ubuntu Server 24.04.2  | Splunk Enterprise (SIEM) | 192.168.10.10      |
| Windows Server 2022    | Active Directory (DC)    | 192.168.10.7       |
| Windows 10 Pro         | Target PC                | 192.168.10.100     |
| Kali Linux             | Attacker                 | 192.168.10.250     |

# ⚙️ Lab Setup

The virtual environment was created using [VirtualBox](https://www.virtualbox.org/), where four separate virtual machines were installed using official ISO images:

- **Windows 10 Pro** – [Download](https://www.microsoft.com/fr-fr/software-download/windows10)
- **Kali Linux** – [Download](https://www.kali.org/get-kali/#kali-platforms)
- **Windows Server 2022** – [Download](https://www.microsoft.com/fr-fr/evalcenter/download-windows-server-2022)
- **Ubuntu Server 24.04.2** – [Download](https://ubuntu.com/download/server)

After installation, a **custom NAT network** named `AD Project` was created using VirtualBox’s network management settings. The network uses the subnet `192.168.10.0/24`. All virtual machines were connected to this isolated NAT network to allow internal communication.

This setup replicates a small enterprise environment for controlled simulation of attacks, event logging, and centralized analysis.

# ⚙️ Ubuntu Server Splunk Setup

After installing the Ubuntu Server VM, the first step is to assign a **static IP address** to prevent the server from receiving a different IP on every DHCP lease renewal. This ensures consistent connectivity within the lab network.

To verify the current IP address, let's run:

```bash
# bash
ip a
```
Ubuntu Server 24.04 uses Netplan for network configuration. The typical Netplan config file (/etc/netplan/00-installer-config.yaml) might not exist by default. If it's missing, it will be created manually using the command:
```bash
# bash
sudo nano /etc/netplan/00-installer-config.yaml
```
Now let's paste the following configuration to assign a static IP address (192.168.10.10/24) and set the default gateway and DNS:
```yaml
# 00-installer-config.yaml
# yaml
network:
  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [192.168.10.10/24]   # Static IP address
      nameservers:
        addresses: [8.8.8.8]           # DNS server (Google DNS)
      routes:
        - to: default
          via: 192.168.10.1            # Default gateway
  version: 2
```
We save the file and apply the changes with:
```bash
# bash
sudo netplan apply
```
The server should now use the static IP 192.168.10.10 on the NAT network subnet. We can check using `ip a`

# 🟧 Splunk Enterprise Installation (Ubuntu Server)

To begin collecting and analyzing logs, Splunk Enterprise must be installed on the Ubuntu Server.

## 1. Download Splunk

From the host machine:

1. Navigate to [splunk.com](https://www.splunk.com) and sign in or create a new account.
2. Go to **Products → Free Trials and Downloads**
3. Select **Splunk Enterprise** and click **Get My Free Trial**
4. Choose the **Linux** platform and download the `.deb` package (Ubuntu is Debian-based)

Transfer the downloaded `.deb` file to the Ubuntu Server. One way to do this is by using VirtualBox’s **shared folder** functionality.

---

## 2. Optional: Set Up Shared Folder (for transferring Splunk installer)

> ⚠️ This step is optional and only needed if using a Virtual Machine to move the `.deb` file from the host to the VM. In a real-world server setup, files would be transferred via other methods.

Install the VirtualBox Guest Additions ISO:

```bash
# bash
sudo apt-get install virtualbox-guest-additions-iso
```
In the VirtualBox menu:
Go to Devices → Shared Folders → Shared Folder Settings

1.  Click the blue icon with a green plus ➕
2.  Select the folder containing your Splunk .deb file
3.  Enable: ✅ Read-only, ✅ Auto-mount, ✅ Make permanent

Back in the terminal:
```bash
# bash
sudo apt-get install virtualbox-guest-utils
sudo reboot
```
After rebooting:
```bash
# bash
sudo adduser <your_username> vboxsf
mkdir share
sudo mount -t vboxsf -o uid=1000,gid=1000 <your-shared-folder-name> share/
cd share
```
## 3. Install Splunk

Navigate to the shared folder and run:
```bash
# bash
sudo dpkg -i splunk-*.deb
```
Then switch to the Splunk directory:
```bash
# bash
cd /opt/splunk
sudo -u splunk bash
cd bin
```
Start Splunk:
```bash
# bash
./splunk start
```
> Use Enter or Space to scroll through the license agreement
> 
> Accept with Y
> 
> Set the admin username and password when prompted (save these credentials)

Exit the Splunk user shell:
```bash
# bash
exit
```
## 4. Enable Splunk to Start on Boot

To ensure Splunk runs automatically on server startup, we use the following command:
```bash
# bash
sudo ./splunk enable boot-start -user splunk
```
Splunk Enterprise is now installed and ready to receive data. The next step is to configure logging on the Windows machines using Sysmon and the Splunk Universal Forwarder.


# 🪟 Windows 10 Configuration and Splunk Universal Forwarder Installation

## 1. Rename PC and Configure Static IP

The target Windows 10 machine is renamed to **TARGET-PC** and configured with a static IP address to ensure consistent network communication within the lab.

**Steps:**

- In the Windows search bar, type **Network status** and open it.
- Click **Change adapter options**.
- Right-click the active network adapter and select **Properties**.
- Select **Internet Protocol Version 4 (TCP/IPv4)** and click **Properties**.
- Choose **Use the following IP address** and enter:

  - **IP address:** `192.168.10.100`
  - **Subnet mask:** `255.255.255.0`
  - **Default gateway:** `192.168.10.1`

- Under **Use the following DNS server addresses**, enter:

  - **Preferred DNS server:** `8.8.8.8`
  - **Alternate DNS server:** *(leave blank)*

Click **OK** to save the settings.

---

### 2. Download and Install Splunk Universal Forwarder

Next, install the Splunk Universal Forwarder (UF) on the Windows 10 machine to forward logs to the Splunk server.

- Visit [splunk.com → Products → Free Trials & Downloads](https://www.splunk.com/en_us/download.html).
- Select **Splunk Universal Forwarder**.
- Choose the appropriate Windows installer:

  - **Windows 10 x64** for 64-bit machines
  - **Windows 10 x86** for 32-bit machines

Download the installer and proceed with installation.

![AD Diagram](/screenshots/splunk.png)

---

*The next step is to configure Sysmon and the Universal Forwarder to collect and forward relevant logs to the Splunk server.*
