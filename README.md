# hackintosh-asus-4xx

[English](README.md) | [Русский](README.ru.md)

> Disclaimer: Translated by Google Translate

---

### OpenCore configuration for ASUS 4XX motherboards

This guide was written after installing macOS 26 Tahoe on an ASUS PRIME H470-PLUS with an i3-10100F. It is also relevant for H470, B460, Z490, and others, but depending on your hardware configuration, there may be different issues or solutions.  
This repository contains verified kexts, ACPI tables, and `config.plist` settings that solve possible issues.

---

### 1. MAT memory bug in some ASUS BIOS versions
* The macOS kernel hard-locks at `#[EB.LD.LKC|R.1]<"BootKernelExtensions.kc">`. In certain ASUS BIOS versions, the Memory Attributes Table is corrupted, and default settings from Dortania lead to freezes. In `Booter --> Quirks` you need to set:
  ```xml
  <key>EnableWriteUnprotector</key>
  <true/>
  <key>RebuildAppleMemoryMap</key>
  <false/>
  <key>SyncRuntimePermissions</key>
  <false/>
  <key>SetupVirtualMap</key>
  <false/>
  ```

---

### 2. BIOS dropping into safe mode upon shutdown
* On shutdown or reboot, the motherboard passes POST in safe mode. The AppleRTC driver overwrites checksums in CMOS. The BIOS perceives this as a power/battery failure. In `Kernel --> Quirks`, enable:
  ```xml
  <key>DisableRtcChecksum</key>
  <true/>
  ```

---

### 3. Broken RealtekRTL8111.kext v3
* This is mostly relevant for ASUS PRIME H470. The latest version RealtekRTL8111.kext v3.0.0 hangs on link negotiation on chip revision 15 without AppleVTD. Rolling back to RealtekRTL8111.kext v2.4.2 (included in the Kexts catalog) fixes this.

---

### 4. No sound in macOS 26 Tahoe (AppleHDA)
* Apple completely removed AppleHDA.kext from macOS Tahoe. The `AppleALC` kext cannot work without the base driver. You need to add **`SimpleHDA.kext`** (by laobamac) to `EFI/OC/Kexts/`. Order in `Kernel --> Add` strictly: `Lilu.kext` --> `AppleALC.kext` --> `SimpleHDA.kext`.
  1. Enable `Kernel --> Quirks --> DisableLinkeditJettison = True`
  2. For 3-jack boards (for example, on ALC887-VD), use `alcid=11` in boot-args and remove the forced layout-id from DeviceProperties.

---

### What kexts are in the Kexts directory and why they are needed

* **`Lilu.kext`** -- in the `Kernel --> Add` list it **always** must be in 1st place. Provides an API for hooking and modifying kernel functions on the fly, mandatory for all other kexts.
* **`VirtualSMC.kext`** -- emulation of the Apple SMC hardware chip. Bypasses authenticity key checks and sensors.
* **`AppleALC.kext`** -- audio driver for Realtek built-in codecs (including ALC887-VD). Dynamically modifies audio and board jack configurations for the required profile.
* **`SimpleHDA.kext`** -- must be placed strictly after AppleALC.kext in config. Only for macOS 26 Tahoe: Apple completely cut out the system AppleHDA.kext, so AppleALC had nothing to patch. SimpleHDA returns the AppleHDA stack on the fly.
* **`RealtekRTL8111.kext` (version 2.4.2)** -- wired Ethernet network card. Exactly version **2.4.2** is placed here, as the fresh version 3.0.0 hangs during speed negotiation on chip revision 15 without AppleVTD.
* **`USBToolBox.kext` + `UTBDefault.kext`** -- pair for USB ports. Allows you to immediately run keyboard, mouse, and flash drives on any motherboard ports without having to manually map ports.
* **`RestrictEvents.kext`** -- universal system patcher from Acidanthera. Needed for Tahoe so the OS does not block booting on unsupported processors and SMBIOS (via the `revpatch=sbvmm` argument), and also blocks unnecessary system warnings.

---

### Tables in the ACPI (.aml) directory

* **`SSDT-AWAC.aml`** -- fixes the real-time clock (RTC). On Intel 400-series chipsets, the AWAC timer incompatible with macOS is enabled by default. This patch disables AWAC and activates the classic RTC.
* **`SSDT-PLUG-DRTNIA.aml`** -- enables native CPU Power Management (`plugin-type=1`), so the CPU properly downclocks at idle.
* **`SSDT-EC-USBX-DESKTOP.aml`** -- creates a dummy embedded controller device and sets USB power parameters.

---

### Mandatory boot arguments (boot-args)
In `NVRAM --> Add --> 7C436110-AB2A-4BBB-A880-FE41995C9F82 --> boot-args`:
```text
-lilubetaall -vsmcbeta -revbeta revpatch=sbvmm alcid=11
```
* `-lilubetaall -vsmcbeta -revbeta` -- force unlock Lilu and plugins on the new Darwin 25/26 kernel (macOS Tahoe).
* `revpatch=sbvmm` -- masks boot for installer and updates via RestrictEvents.
* `alcid=11` -- sets correct layout for 3-jack Realtek ALC887-VD audio codec on ASUS board.
