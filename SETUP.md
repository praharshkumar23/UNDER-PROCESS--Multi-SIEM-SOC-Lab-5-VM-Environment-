
🔧 Multi-SIEM SOC Lab - Setup Guide

Complete step-by-step installation guide for building the lab from scratch.

**Estimated time:** 8-10 hours (split across 2-3 days)

---

## 📋 Phase 1: Environment Preparation (Day 1)

### **1.1 Download Required ISOs**

| Software | Version | Size | Download Link |
|----------|---------|------|---------------|
| Ubuntu Server | 22.04 LTS | ~1.5GB | https://ubuntu.com/download/server |
| Windows 10 | Pro 22H2 | ~5GB | https://www.microsoft.com/software-download/windows10 |
| VMware Workstation | Pro 17 | ~600MB | https://www.vmware.com/products/workstation-pro/workstation-pro-evaluation.html |

### **1.2 Create Virtual Network**

#### **For VMware:**
Edit → Virtual Network Editor

Add Network → VMnet8 (NAT)

Subnet: 192.168.100.0/24

DHCP: Disabled

Apply

text

#### **For VirtualBox:**
File → Host Network Manager

Create new adapter

IPv4: 192.168.100.1

Mask: 255.255.255.0

DHCP: Disabled

text

---

## 🐧 Phase 2: Deploy Ubuntu VMs (Day 1-2)

### **2.1 Create First Ubuntu VM (Splunk)**

**VM Settings:**
- Name: Splunk-SIEM
- OS: Linux → Ubuntu 64-bit
- RAM: 4GB
- CPU: 2 cores
- Disk: 50GB (thin provisioned)
- Network: VMnet8 (NAT)

**Installation Steps:**
```bash
# During Ubuntu installation:
1. Language: English
2. Keyboard: US
3. Network: Manual
   - IP: 192.168.100.10
   - Gateway: 192.168.100.2
   - DNS: 8.8.8.8
4. Storage: Use entire disk
5. Profile:
   - Name: splunkadmin
   - Username: splunkadmin
   - Password: [YourPassword]
6. Install OpenSSH Server: Yes
7. No additional packages
8. Reboot
2.2 Clone VMs for Other SIEMs
text
1. Shutdown Splunk VM
2. Right-click → Manage → Clone
3. Clone 3 times:
   - Sentinel-Agent (192.168.100.11)
   - Wazuh-Manager (192.168.100.12)
   - ELK-Stack (192.168.100.13)
4. After cloning each, change IP:
   sudo nano /etc/netplan/00-installer-config.yaml
   # Update IP address
   sudo netplan apply
🪟 Phase 3: Deploy Windows Target VM (Day 2)
3.1 Create Windows VM
VM Settings:

Name: Windows-Target

OS: Windows 10 64-bit

RAM: 4GB

CPU: 2 cores

Disk: 60GB

Network: VMnet8

Post-Installation:

powershell
# Set static IP
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.100.20 -PrefixLength 24 -DefaultGateway 192.168.100.2
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 8.8.8.8

# Disable Windows Defender (for attack simulation only!)
Set-MpPreference -DisableRealtimeMonitoring $true

# Enable RDP
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# Create test users for attacks
net user testuser Password123! /add
net user admin2 Admin@2024 /add
net localgroup Administrators admin2 /add
🔥 Phase 4: Install Splunk (Day 2)
4.1 Install Splunk Enterprise
bash
# On Splunk VM (192.168.100.10)
cd /tmp
wget -O splunk.tgz 'https://download.splunk.com/products/splunk/releases/9.1.2/linux/splunk-9.1.2-b6b9c8185839-Linux-x86_64.tgz'
sudo tar xvzf splunk.tgz -C /opt
sudo /opt/splunk/bin/splunk start --accept-license --answer-yes --no-prompt --seed-passwd admin123

# Enable boot start
sudo /opt/splunk/bin/splunk enable boot-start

# Open firewall
sudo ufw allow 8000/tcp
sudo ufw allow 9997/tcp
sudo ufw enable

# Access Splunk Web: http://192.168.100.10:8000
# Username: admin
# Password: admin123
4.2 Install Splunk Universal Forwarder on Windows
powershell
# On Windows Target VM
# Download from: https://www.splunk.com/en_us/download/universal-forwarder.html

msiexec.exe /i splunkforwarder-9.1.2-x64-release.msi RECEIVING_INDEXER="192.168.100.10:9997" AGREETOLICENSE=Yes /quiet

# Configure inputs (after installation)
cd "C:\Program Files\SplunkUniversalForwarder\etc\system\local"
notepad inputs.conf
Add to inputs.conf:

text
[WinEventLog://Security]
disabled = 0
index = windows

[WinEventLog://System]
disabled = 0
index = windows

[WinEventLog://Application]
disabled = 0
index = windows

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = sysmon
Restart Forwarder:

powershell
Restart-Service SplunkForwarder
☁️ Phase 5: Setup Microsoft Sentinel (Day 3)
5.1 Create Azure Resources
text
1. Login to Azure Portal (portal.azure.com)
2. Create Resource Group:
   - Name: SOC-Lab-RG
   - Region: East US
3. Create Log Analytics Workspace:
   - Name: SOCLabWorkspace
   - Resource Group: SOC-Lab-RG
   - Region: East US
4. Add Microsoft Sentinel:
   - Search "Microsoft Sentinel"
   - Add → Select SOCLabWorkspace
   - Add
5.2 Install Azure Monitor Agent on Windows
powershell
# On Windows Target
# Download agent from Azure Portal:
# Sentinel → Configuration → Data Connectors → Windows Security Events via AMA → Install agent

# Or use PowerShell:
Invoke-WebRequest -Uri https://aka.ms/AzureConnectedMachineAgent -OutFile AzureConnectedMachineAgent.msi
msiexec /i AzureConnectedMachineAgent.msi /quiet

# Connect to Azure (get command from portal)
azcmagent connect --resource-group "SOC-Lab-RG" --tenant-id "<your-tenant-id>" --location "eastus" --subscription-id "<your-sub-id>"
5.3 Configure Data Collection Rules
text
1. In Sentinel → Configuration → Data connectors
2. Search "Windows Security Events via AMA"
3. Open connector → Create data collection rule
4. Rule name: Windows-Security-Events-DCR
5. Resources: Add Windows-Target VM
6. Collect: All Security Events
7. Review + Create
🛡️ Phase 6: Install Wazuh (Day 3)
6.1 Install Wazuh Manager
bash
# On Wazuh VM (192.168.100.12)
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash ./wazuh-install.sh -a

# Save the output - contains dashboard credentials
# Example:
# User: admin
# Password: AbCd1234EfGh5678IjKl

# Access Wazuh Dashboard: https://192.168.100.12
6.2 Install Wazuh Agent on Windows
powershell
# On Windows Target
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.0-1.msi -OutFile wazuh-agent.msi

# Install with Wazuh Manager IP
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="192.168.100.12" WAZUH_AGENT_NAME="Windows-Target"

# Start service
NET START WazuhSvc
📊 Phase 7: Install ELK Stack (Day 4)
7.1 Install Elasticsearch
bash
# On ELK VM (192.168.100.13)
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
sudo apt-get update && sudo apt-get install elasticsearch

# Configure
sudo nano /etc/elasticsearch/elasticsearch.yml
Edit elasticsearch.yml:

text
network.host: 192.168.100.13
http.port: 9200
discovery.type: single-node
xpack.security.enabled: false  # For lab only!
bash
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch

# Test
curl http://192.168.100.13:9200
7.2 Install Kibana
bash
sudo apt-get install kibana

# Configure
sudo nano /etc/kibana/kibana.yml
Edit kibana.yml:

text
server.host: "192.168.100.13"
server.port: 5601
elasticsearch.hosts: ["http://192.168.100.13:9200"]
bash
sudo systemctl start kibana
sudo systemctl enable kibana

# Access: http://192.168.100.13:5601
7.3 Install Logstash
bash
sudo apt-get install logstash

# Configure Windows log input
sudo nano /etc/logstash/conf.d/windows-logs.conf
Add to windows-logs.conf:

text
input {
  beats {
    port => 5044
  }
}

filter {
  if [winlog][event_id] {
    mutate {
      add_field => { "event_type" => "windows" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://192.168.100.13:9200"]
    index => "windows-logs-%{+YYYY.MM.dd}"
  }
}
bash
sudo systemctl start logstash
sudo systemctl enable logstash
7.4 Install Winlogbeat on Windows
powershell
# On Windows Target
Invoke-WebRequest -Uri https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-8.11.0-windows-x86_64.zip -OutFile winlogbeat.zip
Expand-Archive winlogbeat.zip -DestinationPath C:\ProgramData\
cd C:\ProgramData\winlogbeat-8.11.0-windows-x86_64

# Edit winlogbeat.yml
notepad winlogbeat.yml
Update winlogbeat.yml:

text
output.logstash:
  hosts: ["192.168.100.13:5044"]

winlogbeat.event_logs:
  - name: Security
  - name: System
  - name: Application
  - name: Microsoft-Windows-Sysmon/Operational
powershell
.\install-service-winlogbeat.ps1
Start-Service winlogbeat
🔍 Phase 8: Install Sysmon (Day 4)
8.1 Install Sysmon on Windows
powershell
# Download Sysmon
Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile Sysmon.zip
Expand-Archive Sysmon.zip -DestinationPath C:\Tools\Sysmon

# Download SwiftOnSecurity config
Invoke-WebRequest -Uri https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -OutFile C:\Tools\Sysmon\sysmonconfig.xml

# Install
cd C:\Tools\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig.xml

# Verify
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
✅ Phase 9: Verification (Day 4)
9.1 Test Log Flow to All SIEMs
powershell
# On Windows Target - Generate test event
net user testlog Password123! /add
net user testlog /delete
Check in each SIEM:

Splunk:

text
index=windows EventCode=4720 OR EventCode=4726
| table _time, EventCode, Account_Name
Sentinel:

text
SecurityEvent
| where EventID in (4720, 4726)
| project TimeGenerated, EventID, Account
Wazuh:

text
Security Events → Windows → Event ID 4720/4726
ELK (Kibana):

text
Index Pattern: windows-logs-*
Filter: winlog.event_id: 4720 or 4726
🎉 Setup Complete!
You now have:

✅ 5 VMs running

✅ 4 SIEM platforms configured

✅ Logs flowing from Windows to all SIEMs

✅ Sysmon installed for enhanced visibility
