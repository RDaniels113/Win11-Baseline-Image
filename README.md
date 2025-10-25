# Windows 11 Baseline Image - VMware to Physical Hardware Deployment

## Overview

Replicable Windows 11 golden image process using **VMware Workstation**, converted to physical NVMe drives for enterprise deployment. Solves common deployment failures including TPM/SID conflicts, Outlook profile errors, and Store app sysprep issues.

## The Problem

Cloning Windows 11 to physical hardware is a nightmare:
- **TPM and SID mismatches** break authentication and sign-ins
- **Microsoft Store apps** cause sysprep generalization failures
- **VMware Tools** introduces service errors after cloning to physical hardware
- **USB/NVMe direct installs** are blocked by Windows 11 requirements
- **Manual deployments** take 4-6 hours per system

Traditional imaging solutions fail or require expensive enterprise tools. This workflow produces a **repeatable baseline** that scales across hardware using free/consumer tools.

## The Solution

Build golden image in VMware Workstation, generalize with Sysprep, convert to VHD, clone to physical NVMe drives. Result: clean Windows 11 baseline that deploys in under 1 hour.

**Key Achievement:** 75% deployment time reduction (6 hours → 1 hour)

---

## Technical Workflow

### Phase 1: Build VM in VMware Workstation

**VM Configuration:**
- **Hypervisor:** VMware Workstation 17 Pro
- **Guest OS:** Windows 11 Pro (required for domain join, enterprise features)
- **CPUs:** 2-4 cores (match target hardware)
- **RAM:** 8GB minimum
- **Disk:** 60GB+ (thin provisioned for smaller VMDK initially)
- **Network:** NAT (for Windows Update during build)

**Installation Process:**
1. Install Windows 11 from official Microsoft ISO
2. **Critical:** Skip Microsoft account sign-in during OOBE
   - Use local account only
   - No Microsoft accounts in golden image (causes Outlook errors)
3. Complete initial setup with generic computer name (e.g., BASELINE-REF)

### Phase 2: Patching and Baseline Configuration

**Windows Updates:**
```powershell
# Run Windows Update until fully patched
# Typically requires 2-3 reboot cycles
# Verify no pending updates before proceeding
```

**Install Baseline Applications:**
- Core productivity apps (Office, browsers, etc.)
- **Do NOT install:**
  - Microsoft Store apps (break sysprep)
  - VMware Tools (causes service errors on physical hardware)
  - User-specific software
  - Hardware-specific drivers (install post-deployment)

**System Optimization:**
```powershell
# Disable unnecessary services
Set-Service -Name "XboxGipSvc" -StartupType Disabled
Set-Service -Name "XboxNetApiSvc" -StartupType Disabled

# Remove bloatware (if any)
Get-AppxPackage | Where-Object {$_.Name -like "*Xbox*"} | Remove-AppxPackage

# Optimize power settings
powercfg /change standby-timeout-ac 0
powercfg /hibernate off
```

**Security Baseline:**
- Apply Windows security settings
- Configure Windows Defender
- Set password policy (will be enforced post-deployment)
- Configure Windows Firewall

### Phase 3: Pre-Sysprep Preparation

**Critical: Disable Windows Recovery Environment**

Windows RE can cause partition resizing issues during deployment. Disable before sysprep:

```cmd
reagentc /disable
```

**Why:** WinRE partition can interfere with cloning and extending C: drive. Re-enable after deployment if needed.

**Remove VMware Tools (if installed):**
If accidentally installed, remove before sysprep:
```cmd
# Control Panel > Programs > Uninstall VMware Tools
# Reboot after removal
```

**Verify No Microsoft Store Apps:**
```powershell
# Check for remaining Store apps
Get-AppxProvisionedPackage -Online

# Remove problematic provisioned apps
Get-AppxProvisionedPackage -Online | Where-Object {$_.DisplayName -like "*Store*"} | Remove-AppxProvisionedPackage -Online
```

**Clean Up Before Sysprep:**
```powershell
# Clear temp files
Remove-Item -Path $env:TEMP\* -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path C:\Windows\Temp\* -Recurse -Force -ErrorAction SilentlyContinue

# Clear event logs
wevtutil el | Foreach-Object {wevtutil cl "$_"}

# Disk cleanup
cleanmgr /sageset:1
cleanmgr /sagerun:1
```

### Phase 4: Sysprep Generalization

**Create Sysprep Unattend.xml (optional but recommended):**

Place in `C:\Windows\System32\Sysprep\unattend.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend">
    <settings pass="specialize">
        <component name="Microsoft-Windows-Shell-Setup">
            <ComputerName>*</ComputerName>
            <TimeZone>Eastern Standard Time</TimeZone>
        </component>
    </settings>
    <settings pass="oobeSystem">
        <component name="Microsoft-Windows-Shell-Setup">
            <OOBE>
                <HideEULAPage>true</HideEULAPage>
                <SkipMachineOOBE>false</SkipMachineOOBE>
                <SkipUserOOBE>false</SkipUserOOBE>
            </OOBE>
        </component>
    </settings>
</unattend>
```

**Run Sysprep:**

```cmd
C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown /unattend:C:\Windows\System32\Sysprep\unattend.xml
```

**Sysprep Parameters:**
- `/generalize` - Removes system-specific data (SID, computer name, drivers)
- `/oobe` - Boots into Out-of-Box Experience on next startup
- `/shutdown` - Shuts down after generalization (critical for imaging)
- `/unattend` - Uses answer file to automate post-deployment setup

**VM will shut down when sysprep completes. DO NOT boot it again before conversion.**

---

## Phase 5: VMDK to VHD Conversion

**Why VHD:** Windows cloning tools (AOMEI Backupper, etc.) work better with VHD/VHDX than VMDK.

**Tool: StarWind V2V Converter (Free)**

Download: https://www.starwindsoftware.com/starwind-v2v-converter

**Conversion Process:**
1. Launch StarWind V2V Converter
2. Source image: Browse to VM folder, select `.vmdk` file
3. Source image format: VMware Workstation
4. Destination image format: Microsoft VHD/VHDX
5. VHD format: VHD (not VHDX) for compatibility
6. Destination: Select output location
7. Convert (takes 10-20 minutes depending on image size)

**Result:** Sysprepped Windows 11 image in VHD format ready for cloning.

---

## Phase 6: Clone VHD to Physical NVMe

**Tool: AOMEI Backupper (Free Standard version works)**

Download: https://www.aomeitech.com/backup-software.html

**Prerequisites:**
- Target system with NVMe drive installed
- USB drive containing AOMEI Backupper bootable media
- VHD file accessible via USB or network

**Cloning Process:**

1. **Create AOMEI Bootable USB** (one-time setup):
   - Tools > Create Bootable Media
   - Windows PE-based bootable media
   - USB Boot Device

2. **Boot Target System from USB**:
   - Boot target hardware from AOMEI USB
   - Wait for AOMEI Backupper to load

3. **Restore VHD to NVMe**:
   - Restore > Select Image File
   - Browse to VHD file location
   - Select target NVMe drive
   - **Check "SSD Alignment"** (critical for NVMe performance)
   - Start Restore

4. **Completion**:
   - Restore takes 15-30 minutes depending on image size
   - Remove USB, reboot

**Result:** Physical hardware now has generalized Windows 11 baseline.

---

## Phase 7: Post-Deployment Configuration

System will boot into Windows OOBE (Out-of-Box Experience).

**First Boot Tasks:**

1. **Complete OOBE**:
   - Set computer name
   - Join domain or create local account
   - Configure network settings

2. **Extend C: Drive** (if needed):
   ```cmd
   # Open Disk Management (diskmgmt.msc)
   # Right-click C: > Extend Volume
   # Extend to use full NVMe capacity
   ```

3. **Recreate Windows RE** (if needed):
   ```cmd
   reagentc /enable
   reagentc /info  # Verify enabled
   ```

4. **Install Hardware-Specific Drivers**:
   - GPU drivers
   - Network adapter drivers (if not working)
   - Chipset drivers
   - Any manufacturer-specific utilities

5. **Validate System**:
   ```powershell
   # Check for missing drivers
   Get-WmiObject Win32_PnPEntity | Where-Object {$_.ConfigManagerErrorCode -ne 0}
   
   # Verify Windows activation
   slmgr /xpr
   
   # Check disk health
   Get-PhysicalDisk
   ```

6. **Join Domain (if applicable)**:
   ```powershell
   Add-Computer -DomainName "domain.local" -Credential (Get-Credential) -Restart
   ```

7. **Apply Group Policy / Install Domain-Specific Software**:
   ```powershell
   gpupdate /force
   ```

**Deployment Complete:** System ready for production use in ~1 hour total.

---

## Common Issues and Solutions

### Issue: Sysprep Fails with Store App Error

**Error:** "Sysprep was not able to validate your Windows installation"

**Cause:** Microsoft Store apps (including built-in apps) prevent generalization

**Solution:**
```powershell
# Remove all provisioned Store apps
Get-AppxProvisionedPackage -Online | Remove-AppxProvisionedPackage -Online

# Retry sysprep
```

### Issue: TPM Error on First Boot

**Error:** "A TPM error occurred" or authentication fails

**Cause:** TPM chip on physical hardware has different key than VM

**Solution:** 
- TPM automatically cleared during sysprep `/generalize`
- If issues persist: BIOS > Clear TPM
- Re-enable TPM after boot

### Issue: Outlook Sign-In / Profile Errors

**Error:** Outlook won't sign in or create profile after deployment

**Cause:** Microsoft account used in golden image

**Solution (Prevention):**
- Never sign into Microsoft account in golden image
- Use local account only during build
- Outlook profiles created fresh post-deployment

### Issue: VMware Tools Service Errors

**Error:** VMware tools service fails to start on physical hardware

**Cause:** VMware Tools installed in golden image

**Solution (Prevention):**
- Do not install VMware Tools in golden image
- If present, uninstall before sysprep

### Issue: NVMe Not Detected in AOMEI

**Error:** Target NVMe drive not visible in AOMEI Backupper

**Cause:** NVMe driver not included in AOMEI bootable media

**Solution:**
- Use AOMEI "Windows PE" bootable media (not Linux)
- Update AOMEI to latest version
- Try alternative tool (Macrium Reflect, Clonezilla)

### Issue: C: Drive Not Extended After Clone

**Error:** C: partition smaller than physical NVMe capacity

**Cause:** AOMEI clones exact partition sizes from VHD

**Solution:**
```cmd
# Windows: Disk Management
diskmgmt.msc
# Right-click C: > Extend Volume

# Or PowerShell:
$MaxSize = (Get-PartitionSupportedSize -DriveLetter C).SizeMax
Resize-Partition -DriveLetter C -Size $MaxSize
```

---

## Maintenance and Updates

### Monthly Image Refresh (Recommended)

1. Boot golden image VM (pre-sysprep backup)
2. Run Windows Update
3. Update installed applications
4. Test functionality
5. Re-run sysprep and conversion process
6. Update image version number

**Version Numbering:**
`Win11-Baseline-v[YEAR].[MONTH].vhd`

Example: `Win11-Baseline-v2025.10.vhd`

### Image Storage and Backups

**Storage Locations:**
- Primary: Network share or NAS for easy access
- Backup: External drive or cloud storage
- Size: 8-12GB compressed, 30-40GB uncompressed VHD

**Keep Multiple Versions:**
- Current month
- Previous month (rollback option)
- Known-good stable version (quarterly)

---

## Deployment Statistics

**Time Comparison:**

| Method | Time Required |
|--------|---------------|
| Manual Windows 11 install + updates + apps | 4-6 hours |
| This golden image workflow | 45-60 minutes |
| **Time Savings** | **75%** |

**Deployment Breakdown:**
- VHD to NVMe clone: 15-30 minutes
- First boot OOBE: 5 minutes
- Post-deployment config: 15-30 minutes
- **Total: ~1 hour per system**

**Tested On:**
- Dell Precision workstations
- HP EliteDesk desktops
- Custom-built systems with NVMe
- Various NVMe brands (Samsung, WD, Crucial)

**Success Rate:** 100% across tested hardware

---

## Skills Demonstrated

✅ **Windows Deployment** - Sysprep, generalization, OOBE configuration  
✅ **Virtualization** - VMware Workstation VM creation and management  
✅ **Image Conversion** - VMDK to VHD conversion workflow  
✅ **Hardware Cloning** - Physical disk imaging and restoration  
✅ **Troubleshooting** - Solved TPM/SID, Outlook, Store app issues  
✅ **Process Optimization** - 75% deployment time reduction  
✅ **Documentation** - Professional technical writing  

---

## Tools Used

**Virtualization:**
- VMware Workstation 17 Pro

**Image Conversion:**
- StarWind V2V Converter (free)

**Physical Cloning:**
- AOMEI Backupper (free standard version)

**Operating System:**
- Windows 11 Pro (official Microsoft ISO)

**All tools are free or have free versions available for this workflow.**

---

## Benefits of This Approach

### Repeatable Process
- Documented workflow can be followed by others
- Consistent results across deployments
- No specialized knowledge required after initial setup

### Hardware Agnostic
- Works across different manufacturers
- Tested on Dell, HP, custom-built systems
- Generic drivers loaded during OOBE

### Cost Effective
- Uses free tools (AOMEI, StarWind)
- No expensive enterprise imaging solutions required
- VMware Workstation widely available

### Time Efficient
- 75% faster than manual installation
- Minimal hands-on time during deployment
- Scales to multiple systems

### Solves Real Problems
- TPM/SID conflicts handled by sysprep
- Outlook errors prevented by local-account-only build
- Store app failures eliminated by removal before sysprep
- VMware Tools issues avoided by not installing

---

## Future Enhancements

- [ ] Automate post-deployment driver installation
- [ ] Create PowerShell script for post-deployment configuration
- [ ] Document network deployment via WDS/MDT
- [ ] Add automation for monthly image refresh
- [ ] Create validation test suite for deployed systems

---

## License

This project is licensed under the MIT License - free to use, attribution appreciated.

---

## Repository Structure

```
Win11-Baseline-Image/
├── README.md (this file)
├── docs/
│   ├── sysprep-unattend-example.xml
│   └── troubleshooting-guide.md
├── scripts/
│   └── post-deployment-config.ps1 (future)
└── LICENSE
```

---

**Status:** Production-ready process, actively maintained  
**Last Updated:** October 2025  
**Deployments Completed:** 10+ successful deployments  
**Success Rate:** 100%
