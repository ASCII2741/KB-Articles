# Guide: Flashing Dell R720xd (H710P Mini + H810) to IT Mode & Fixing Boot Conflicts

**Author:** Bikesh Mahato
**Platform:** Dell PowerEdge R720xd (LFF)  
**Target OS:** TrueNAS Scale  

This guide documents the process of converting a Dell PowerEdge R720xd with **both** an internal H710P Mini and an external H810 to IT Mode (HBA). It specifically addresses the critical "Ghost Card" issue where the server refuses to boot from internal drives when the external card is installed, and provides best practices for a smooth installation.

---

## ⚠️ Critical Warnings (Read Before Starting)

### 1. Disconnect Old Drives Before Booting

**Do not** attempt to boot into TrueNAS (or the installer) with old, unwiped SSDs/HDDs attached if they were part of a previous ZFS pool or RAID array.

* **The Issue:** TrueNAS will attempt to import or scan these "zombie" pools during boot. This can cause the system to hang or lag for **hours** while it times out on every disk.
* **The Fix:** Physically disconnect all data drives. Only connect the target boot drive(s) initially, or ensure all drives are securely wiped (using `blkdiscard` or similar) before starting the installation.

### 2. Do Not Mirror Boot Drives During Install

If you plan to use a mirrored boot pool (RAID1) for the OS:

* **Do not** select both drives during the TrueNAS installation wizard.
* **The Strategy:** Install TrueNAS to **one single SSD** first. Once the system is up and running, log into the Web UI and attach the second SSD as a mirror.
* **Why?** Creating the mirror during installation often complicates the UEFI boot loader setup on Dell servers, leading to "No Boot Device" errors. Adding the mirror later is safer and easier.

---

## The Hardware Setup

* **Server:** Dell PowerEdge R720xd
* **Internal Card:** Dell PERC H710P Mini (Integrated Slot) – *For OS/Boot Drives*
* **External Card:** Dell PERC H810 (PCIe Slot 5) – *For MD1220 Disk Shelf / Mass Storage*

## The Problem: "The Ghost Card"

After flashing both cards to IT Mode, the server would boot fine with *only* the internal H710P. However, as soon as the external H810 was inserted, the BIOS would fail with:
> *"No boot device available."*

**Root Cause:**
The Dell BIOS scans PCIe slots in a specific order. When the H810 (Slot 5) is installed, it shifts the PCI address map. If the H810 is flashed to IT Mode but left "brainless" (no Boot ROM/BIOS Image), the Dell BIOS treats it as an "Unknown/Broken Device," hangs on the identification step, and stops scanning before it ever reaches the internal H710P.

---

## The Solution: "The Traffic Cop" Configuration

To fix this, we must ensure **BOTH** cards have a valid Boot ROM so the BIOS can identify them, but then manually configure the BIOS to ignore the external card.

### Part 1: Preparation

1. Follow [Fohdeesha's PERC Flashing Guide](https://fohdeesha.com/docs/perc.html) to flash the main IT Mode firmware to your cards.
2. Boot into the Linux flashing environment (`sudo su -`).

### Part 2: Flash the Boot ROM to BOTH Cards

You must flash the UEFI Boot ROM (`x64sas2.rom`) to **both** controllers. Do not skip the external card!

1. **Identify your cards:**

    ```bash
    ./sas2flash -listall
    ```

    * *Note:* The H710P Mini is usually on **Bus 2** (Controller 0). The H810 is usually on **Bus 3** (Controller 1).

2. **Flash Internal Card (Controller 0):**

    ```bash
    ./sas2flash -c 0 -o -b /root/Bootloaders/x64sas2.rom
    ```

3. **Flash External Card (Controller 1):**

    ```bash
    ./sas2flash -c 1 -o -b /root/Bootloaders/x64sas2.rom
    ```

4. **Verify:** Run `./sas2flash -listall`. Both cards should now show a version number (or "Flash Successful").

### Part 3: Configure BIOS "Traffic Rules"

Now that both cards have brains, we need to tell the server which one to boot from.

1. **Reboot** and press **F2** to enter **System Setup**.
2. Go to **Device Settings**.
    * *Success Check:* You should see **two** "LSI SAS2" (or "Avago") entries listed.

**Step A: Disable External H810 (Slot 5)**

1. Select the **Slot 5** card.
2. Go to **Controller Management** -> **Change Controller Properties**.
3. Set **Boot Support** to **"None"** (or "OS Only").
4. **Apply Changes.**
    * *Why:* This keeps the card visible to the OS (TrueNAS) but prevents the BIOS from trying to boot from it.

**Step B: Enable Internal H710P (Integrated)**

1. Select the **Integrated Mini** card.
2. Go to **Controller Management** -> **Change Controller Properties**.
3. Set **Boot Support** to **"Enabled BIOS & OS"**.
4. **Apply Changes.**

### Part 4: Create the Final Boot Option

1. Go back to **System BIOS** -> **Boot Settings** -> **UEFI Boot Settings**.
2. **Delete** any old/broken boot options (like "TrueNAS-0").
3. Click **Add Boot Option**.
4. **File System List:**
    * Click the list. You will see multiple `PciRoot...` entries.
    * Select the one that corresponds to your Internal SSD. (Trial and error: select one, click "File Name", and check if you can browse to an `EFI` folder).
5. **Select the File:** Browse to `EFI` -> `BOOT` -> `BOOTX64.EFI`.
6. **Name It:** "TrueNAS Final".
7. **Move to Top:** Ensure it is the first priority.
8. **Save, Exit, and Reboot.**

---

## Summary of Commands

| Action | Command |
| :--- | :--- |
| **List All Cards** | `./sas2flash -listall` |
| **Flash Boot ROM (Internal)** | `./sas2flash -c 0 -o -b x64sas2.rom` |
| **Flash Boot ROM (External)** | `./sas2flash -c 1 -o -b x64sas2.rom` |
| **Erase Boot ROM (Only if needed)** | `./sas2flash -c X -o -e 5` |
| **Set SAS Address (If 0s)** | `./sas2flash -c X -o -sasadd 500xxxxxxxxxxxxx` |

---

**Result:** The server will pass the POST check for the H810 without hanging, skip it for booting, find the H710P, and launch TrueNAS immediately.
