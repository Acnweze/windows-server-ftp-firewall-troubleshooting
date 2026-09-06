# Windows Server FTP Firewall Troubleshooting

## Overview

This project demonstrates a hands-on **Windows Server FTP troubleshooting and firewall configuration** scenario using PowerShell, IIS FTP, Windows Defender Firewall, and TCP connectivity testing.

The lab focuses on a practical troubleshooting workflow:

**Identify the problem → collect evidence → isolate the cause → apply a targeted fix → validate the result.**

---

## Objective

Troubleshoot an FTP connectivity issue between two Windows Servers.

The FTP server was running and reachable on the network, but the client could not establish a connection to **TCP port 21**.

The goal was to:

* Verify the Windows FTP Server feature
* Verify network configuration
* Verify the IIS FTP binding
* Review Windows Firewall rules
* Enable firewall logging
* Identify the cause of the connection failure
* Create a targeted firewall rule
* Retest connectivity
* Confirm successful FTP connectivity

---

## Lab Environment

| Component                  | Configuration             |
| -------------------------- | ------------------------- |
| Client / Domain Controller | TW-DC01                   |
| Client IP Address          | 172.16.0.4                |
| FTP Server                 | TW-SRV02                  |
| FTP Server IP Address      | 172.16.0.8                |
| Operating System           | Windows Server 2022       |
| FTP Platform               | IIS FTP                   |
| Protocol                   | FTP                       |
| FTP Port                   | TCP 21                    |
| Administration             | PowerShell                |
| Firewall                   | Windows Defender Firewall |

---

## Technologies & Skills

* Windows Server Administration
* IIS FTP
* PowerShell
* Windows Defender Firewall
* TCP/IP Troubleshooting
* Network Connectivity Testing
* Firewall Rule Management
* Firewall Logging
* Port Verification
* Service Verification
* Evidence-Based Troubleshooting

---

# Troubleshooting Process

## 1. Verify Windows Server FTP Feature

The first step was to verify that the required Windows Server FTP feature was installed.

```powershell
Get-WindowsFeature Web-Ftp-Server
```

The FTP Server feature was installed as part of the IIS configuration.

![Windows Server FTP Feature Installation](get%20and%20install-%20windowsFeature%20.png)

---

## 2. Verify Network Configuration

The FTP server's network configuration was checked to confirm the assigned IP address and network interface.

```powershell
Get-NetIPConfiguration
Get-NetIPAddress
```

The FTP server was configured with:

```text
172.16.0.8
```

![Network Configuration](get-netIPConfiguration%20and%20get%20netIPAddress.png)

---

## 3. Test TCP Port 21

Connectivity from the client to the FTP server was tested using PowerShell.

```powershell
Test-NetConnection 172.16.0.8 -Port 21
```

### Initial Result

```text
TcpTestSucceeded : False
```

The server was reachable, but the client could not establish a TCP connection to port 21.

![Initial TCP Connectivity Test](test-netconnetion%20172.16.png)

---

## 4. Verify IIS FTP Binding

The IIS FTP binding was checked to confirm that FTP was configured for the expected IP address and port.

```powershell
Get-WebBinding -Protocol ftp
```

The FTP binding showed:

```text
172.16.0.8:21
```

![IIS FTP Binding and Connectivity Test](get-webbinding%20and%20test-netconnection%20172.16.0.8-%20port%2021.png)

---

## 5. Review Windows Firewall Rules

The Windows Defender Firewall rules associated with the FTP Server role were reviewed.

```powershell
Get-NetFirewallRule -DisplayGroup "FTP Server"
```

The FTP firewall rules were enabled, including the inbound FTP traffic rule.

![FTP Firewall Rules](get-netfirewallrule%20and%20get-windowsfeature%20web-FTP%20server.png)

At this point, the FTP service and IIS configuration appeared correct.

The next step was to determine whether Windows Firewall was actually dropping the connection.

---

## 6. Enable Windows Firewall Logging

Windows Defender Firewall logging was enabled for the Domain, Private, and Public profiles.

```powershell
Set-NetFirewallProfile `
  -Profile Domain,Private,Public `
  -LogBlocked True
```

The firewall configuration was then verified:

```powershell
Get-NetFirewallProfile
```

![Firewall Logging Configuration](set%20and%20get-netFirewallprofile.png)

---

## 7. Identify the Firewall Drop

After generating another connection attempt from TW-DC01, the Windows Firewall log was examined.

The firewall log showed inbound TCP traffic:

```text
Source:            172.16.0.4
Destination:       172.16.0.8
Destination Port:  21
Action:            DROP
```

This provided the evidence needed to identify the problem.

### Root Cause

**Inbound TCP traffic to port 21 was being dropped by Windows Defender Firewall on the FTP server.**

Rather than disabling the firewall or making broad security changes, a targeted firewall rule was created.

---

## 8. Create a Targeted Firewall Rule

An explicit inbound firewall rule was created for TCP port 21 on the Private profile.

```powershell
New-NetFirewallRule `
  -DisplayName "TechWave FTP TCP 21 Inbound" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 21 `
  -Action Allow `
  -Profile Private
```

The rule allowed the required FTP traffic while keeping the firewall enabled.

![TechWave FTP Firewall Rule](techwaveFTP.png)

---

## 9. Retest TCP Connectivity

The connection was tested again from TW-DC01.

```powershell
Test-NetConnection 172.16.0.8 -Port 21
```

### Result

```text
TcpTestSucceeded : True
```

The TCP connectivity issue was successfully resolved.

![Successful TCP Connectivity Test](get-webbinding%20and%20test-netconnection%20172.16.0.8-%20port%2021.png)

---

## 10. Validate FTP Application Connectivity

A final FTP connection was established from the client.

```powershell
ftp 172.16.0.8
```

The server responded:

```text
220 Microsoft FTP Service
```

This confirmed that the FTP service was accessible from the client.

![Successful FTP Connection](techwaveFTP.png)

---

# Before vs. After

| Test                   | Before |            After            |
| ---------------------- | :----: | :-------------------------: |
| Server reachable       |    ✓   |              ✓              |
| FTP feature installed  |    ✓   |              ✓              |
| FTP binding configured |    ✓   |              ✓              |
| TCP port 21 listening  |    ✓   |              ✓              |
| TCP 21 connectivity    |    ✗   |              ✓              |
| FTP connection         |    ✗   |              ✓              |
| FTP server response    |    —   | `220 Microsoft FTP Service` |

---

# Troubleshooting Methodology

The lab followed a structured troubleshooting process:

```text
Client Connectivity Test
          ↓
Network Configuration Check
          ↓
FTP Feature Verification
          ↓
IIS FTP Binding Verification
          ↓
Firewall Rule Review
          ↓
Firewall Logging
          ↓
Identify Dropped Traffic
          ↓
Targeted Firewall Rule
          ↓
Retest TCP Connectivity
          ↓
Validate FTP Connection
```

### Key Principle

> **Use evidence to identify the failing layer before making configuration changes.**

Instead of immediately disabling the firewall, firewall logging was used to identify the traffic being dropped.

---

# Key PowerShell Commands

### Check FTP Server Feature

```powershell
Get-WindowsFeature Web-Ftp-Server
```

### Check Network Configuration

```powershell
Get-NetIPConfiguration
Get-NetIPAddress
```

### Test TCP Port 21

```powershell
Test-NetConnection 172.16.0.8 -Port 21
```

### Check IIS FTP Binding

```powershell
Get-WebBinding -Protocol ftp
```

### Review FTP Firewall Rules

```powershell
Get-NetFirewallRule -DisplayGroup "FTP Server"
```

### Enable Firewall Logging

```powershell
Set-NetFirewallProfile `
  -Profile Domain,Private,Public `
  -LogBlocked True
```

### Create Targeted FTP Firewall Rule

```powershell
New-NetFirewallRule `
  -DisplayName "TechWave FTP TCP 21 Inbound" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 21 `
  -Action Allow `
  -Profile Private
```

### Test FTP Application Connectivity

```powershell
ftp 172.16.0.8
```

---

# Evidence

The repository contains screenshots documenting the troubleshooting process:

* Windows Server FTP feature installation
* Network configuration
* Initial TCP connectivity failure
* IIS FTP binding
* FTP firewall rules
* Firewall logging configuration
* Firewall rule configuration
* Successful TCP connectivity
* Successful FTP connection

---

# Final Result

The FTP connectivity issue was successfully diagnosed and resolved.

### Before

```text
TcpTestSucceeded : False
```

### After

```text
TcpTestSucceeded : True
```

### Application-Level Validation

```text
220 Microsoft FTP Service
```

The completed lab demonstrates practical experience with:

**Windows Server | IIS FTP | PowerShell | Windows Defender Firewall | TCP/IP | Network Troubleshooting | Firewall Logging**

---

## Career Relevance

This project demonstrates practical troubleshooting skills relevant to:

* Windows Administrator
* Windows Server Administrator
* Azure Administrator
* Azure Cloud Support Engineer
* Infrastructure Support Engineer
* Junior Systems Administrator
* IT Infrastructure Engineer

The focus is on **hands-on troubleshooting rather than theoretical configuration** — identifying the problem, gathering evidence, applying a controlled fix, and validating the outcome.
