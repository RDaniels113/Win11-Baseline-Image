# Windows 11 Baseline Image Deployment (VMware → NVMe)

---
[← Back to README](../README.md) | [Tools](tools.md)
---

## Overview
Create a Windows 11 golden image in VMware Workstation, then deploy it to physical NVMe drives. This workflow avoids USB install blocks, TPM/SID pitfalls, Outlook errors, and WinRE partition traps. The result is a clean, sysprep-ready, repeatable image.

## Problem Statement
- Windows 11 installer blocks installs to USB/NVMe enclosures
- Registry hacks and diskpart workarounds are unreliable
- Need a clean, deployable golden image for multiple machines

## Solution: VMware VM Workflow

### Phase 1: VM Creation & OOBE Bypass

#### 1. Create Windows 11 VM in VMware Workstation
- Windows 11 Pro recommended
- 4 GB+ RAM, 60 GB+ disk
- **Disable network adapter** at creation

#### 2. Install Windows 11
- Boot from ISO
- At "Let's get you connected" press **Shift + F10**
- Run the bypass command:
```cmd
oobe\bypassnro
```
- Restart → choose "I don't have internet"
- Create a local administrator account

#### 3. Re-enable Network
- VM Settings → enable "Connect at power on"
- Connect to internet (stay on local account; do not sign into Microsoft)

---

## Phase 2: Golden Image Configuration

### ⚠️ Do not install VMware Tools
VMware Tools drivers/services can cause errors after sysprep when the image is cloned to physical hardware. If you installed it for convenience, uninstall it before proceeding.

#### 4. Install Updates & Software
- Fully patch via Windows Update
- Install Microsoft 365 Apps (**do not sign in**)
- Install other baseline/enterprise apps
- Configure system settings (power, taskbar, File Explorer, etc.)

#### 5. Important: No User Sign-ins or Store Apps
- No Microsoft account sign-ins
- No Outlook configuration
- No Microsoft Store downloads or installs (breaks sysprep)
- No personal apps (ChatGPT, Spotify, etc.)

---

## Phase 3: Sysprep Preparation

#### 6. Remove Problematic Apps (if needed)
If any Store apps were installed by accident, remove them before sysprep:

```powershell
# Run PowerShell as Administrator
Get-AppxPackage -Name "*OpenAI.ChatGPT*" | Remove-AppxPackage
# Repeat for other Store apps as needed
```

#### 7. Disable WinRE (prevents C: growth from being blocked)
WinRE is typically placed at the **end of the OS partition**, which can block extending C: after cloning.

```cmd
reagentc /info
reagentc /disable
```

**Optional but recommended:** Delete the Recovery partition now so C: can extend cleanly later:

```cmd
diskpart
list disk
select disk 0
list partition
select partition <#>
rem Replace <#> with the Recovery partition number
delete partition override
exit
```

#### 8. Run Sysprep
```cmd
C:\Windows\System32\sysprep\sysprep.exe /generalize /oobe /shutdown
```
- Generalizes the system
- Prepares for hardware detection
- VM shuts down when complete

---

## Phase 4: VM to Physical Deployment

#### 9. Convert VMDK to VHD
- Use **StarWind V2V Converter**
- Source: main `.vmdk` file (not split parts)
- Destination: Microsoft VHD (pre-allocated)

#### 10. Clone to Physical NVMe
- Mount `.vhd` in Disk Management (Action → Attach VHD)
- Use **AOMEI Backupper Standard** → Disk Clone: VHD → NVMe
- Choose manual partition adjustment

#### 11. Partition Handling
- If WinRE was **disabled/deleted** before sysprep, extend C: now without issue
- If a Recovery partition still sits after C:, delete it, extend C:, then recreate WinRE
- Extend C: in Disk Management to fill available space

#### 12. Recreate and Re-enable WinRE (after extending C:)

**Simple approach (suitable for small shops):**
```cmd
mkdir R:\Recovery\WindowsRE
copy C:\Windows\System32\Recovery\winre.wim R:\Recovery\WindowsRE\
reagentc /setreimage /path R:\Recovery\WindowsRE
reagentc /enable
```

**If you need to create a dedicated 1 GB Recovery partition with proper GPT attributes:**
```cmd
diskpart
list disk
select disk 0
create partition primary size=1024
format quick fs=ntfs label="Recovery"
assign letter=R
set id=de94bba4-06d1-4d40-a16a-bfd50179d6ac
gpt attributes=0x8000000000000001
exit

mkdir R:\Recovery\WindowsRE
copy C:\Windows\System32\Recovery\winre.wim R:\Recovery\WindowsRE\
reagentc /setreimage /path R:\Recovery\WindowsRE
reagentc /enable
```

**If you temporarily assigned drive letter R:, you can remove it afterward:**
```cmd
diskpart
select volume R
remove letter=R
exit
```

---

## Phase 5: Testing & Validation

#### 13. Test Golden Image
- Boot the cloned NVMe in a test machine
- Complete OOBE
- Validate installed applications
- Verify Windows activation
- Confirm Outlook profile/sign-in works normally

---

## Troubleshooting Notes

### USB Installation Issues
- Windows 11 aggressively blocks USB installs
- Registry hacks like `StorageDevicePolicies` often fail
- VM approach bypasses these restrictions

### Sysprep Validation Errors
- Usually caused by Microsoft Store apps
- Remove with PowerShell before sysprep
- Check logs: `C:\Windows\System32\Sysprep\Panther\setuperr.log`

### Hardware Detection
- One-time hardware detection messages after first boot are normal
- The sysprepped image adapts across different hardware

---

## Post-Deployment User Setup

1. Complete OOBE → users create their own accounts
2. Join on-prem domain or Azure AD
3. Office activation with corporate licensing
4. Users install personal apps and configure OneDrive/settings

---

## Benefits

- Clean golden image with no user artifacts
- Proper hardware detection via sysprep
- Avoids TPM/SID mismatches and Outlook errors
- Scalable NVMe deployment for SMB and enterprise

---

## Lessons Learned

- **Never install Microsoft Store apps** in the golden image. They break sysprep.
- **Do not install VMware Tools** in the VM. It causes driver/service errors after sysprep on physical hardware.
- TPM/SID mismatches caused most Outlook failures.
- Repeatability beats quick fixes over time.
- Test across different NVMe controllers.
- **Disable WinRE and remove the Recovery partition before sysprep** to avoid C: resize headaches. Recreate and re-enable WinRE after deployment.

---

## Tools Required

- [VMware Workstation](https://www.vmware.com/products/workstation-pro.html)
- [StarWind V2V Converter](https://www.starwindsoftware.com/starwind-v2v-converter) (Free)
- [AOMEI Backupper Standard](https://www.aomeitech.com/ab/standard.html) (Free)
- Windows 11 Pro ISO

---

## Final Notes

This workflow produces a **clean, replicable Windows 11 image** for NVMe deployment. It avoids USB blocks, TPM/SID mismatches, Outlook profile errors, and WinRE partition issues, giving users a stable baseline they can customize post-deployment.
