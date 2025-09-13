# Windows 11 Baseline Image for NVMe Deployment  

---
[Full Guide](docs/Win11-Image-FullGuide.md) | [Tools](docs/tools.md)
---

## Overview  
This project provides a **replicable Windows 11 golden image** process using VMware Workstation, converted to NVMe drives for physical deployment. It bypasses:  
- USB installation restrictions  
- TPM and SID conflicts  
- Outlook profile/sign-in errors  

The result is a clean, sysprep-ready image that just works.  

## Why This Matters  
Cloning Windows 11 to NVMe is notoriously painful:  
- TPM and SID mismatches break sign-ins  
- Microsoft Store apps cause sysprep failures  
- VMware Tools introduces service errors after cloning  
- USB/NVMe direct installs are blocked  

This workflow saves hours by producing a **repeatable baseline image** that scales across hardware.  

## High-Level Process  
1. Build Windows 11 VM in VMware  
2. Patch and install baseline apps (**no user sign-ins, no Store apps, no VMware Tools**)  
3. Disable WinRE (recommended), then sysprep → generalize and shut down  
4. Convert VMDK → VHD (StarWind)  
5. Clone VHD → NVMe (AOMEI Backupper)  
6. Extend C:, recreate/enable WinRE if needed, and validate on hardware  

## Lessons Learned  
- Never install Microsoft Store apps in the golden image — they break sysprep.  
- Do not install VMware Tools — it causes errors after sysprep on physical hardware.  
- Treat this like engineering, not patchwork — repeatability saves time.  
- Disable WinRE before sysprep to prevent partition resizing issues; recreate later if needed.  

---
[Full Guide](docs/Win11-Image-FullGuide.md) | [Tools](docs/tools.md)
---
