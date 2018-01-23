---
title: "VMVare and Windows 10"
date: 2018-01-22T01:39:03Z
tags: ["vmware"]
---

I recently upgraded by developer machine to Windows 10. 
After the upgrade, VMWare would fail to start any image with an error about disabling Hyper-V. 
Specifically this was the error

```
VMware Workstation and Hyper-V are not compatible. Remove the Hyper-V role from the system before running VMware Workstation
```

After many cycles of uninstalling and reinstalling VMWare and Hyper-V, 
turns out the problem is actually related to something called 
[Device Guard](https://docs.microsoft.com/en-us/windows/device-security/device-guard/introduction-to-device-guard-virtualization-based-security-and-code-integrity-policies).

I found the solution [here](https://social.technet.microsoft.com/Forums/ie/en-US/4fdb9c2a-8437-4fe2-b48a-dbc3d65cce9e/build-14295-vmware-workstation-pro-12-thinks-hyperv-is-installed?forum=WindowsInsiderPreview).
But the gist of the solution is

1. Download the [Device Guard and Credential Guard hardware readiness tool](https://www.microsoft.com/en-us/download/details.aspx?id=53337)
 
2. Run the extracted script: 
```bash
DG_Readiness_Tool_v3.2.ps1 -disable 
```

3. Make sure you have the feature uninstalled 
```bash
 Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V –All
```

4. You may also need to disable Hyper-V from starting by using the following command line:
```bash
bcdedit /set hypervisorlaunchtype off
```