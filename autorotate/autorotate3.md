# Xiaomi Pad 6 (pipa) Sensor & Auto-Rotate Investigation Summary

## System Environment
* **Device:** Xiaomi Pad 6 (Codename: `xiaomi-pipa`)
* **OS:** postmarketOS (systemd variant)
* **Architecture:** aarch64
* **Sensor Interface:** Hexagon DSP (SDSP) accessed via `fastrpc` kernel driver.
* **Key Services:** `iio-sensor-proxy.service`, `hexagonrpcd-sdsp.service`

## Verified Configuration
1.  **Mount Matrix:**
    * File: `/usr/lib/udev/rules.d/81-libssc-xiaomi-pipa.rules`
    * Content: `ENV{ACCEL_MOUNT_MATRIX}+="-1, 0, 0; 0, -1, 0; 0, 0, -1"`
2.  **Udev Rules (Sensor Types):**
    * File: `/usr/lib/udev/rules.d/80-iio-sensor-proxy-libssc.rules`
    * Content: Assigns `ssc-accel` to `fastrpc-adsp*` and `fastrpc-sdsp*` devices.
    * Modification: The "hack" to remove `ssc-light`, `ssc-proximity`, and `ssc-compass` was applied via `sed`.
3.  **Service Override:**
    * File: `/etc/systemd/system/iio-sensor-proxy.service.d/override.conf` created and edited.

## Command Execution & Results

### 1. Boot Behavior
* **Command:** `sudo monitor-sensor` (immediately after reboot)
* **Result:** Application hangs at "Waiting for iio-sensor-proxy to appear" or appears but reports no sensors.

### 2. Manual Service Sequences (Crucial Findings)

#### Sequence A (Successful)
* **Commands:**
    1.  `sudo systemctl stop hexagonrpcd-sdsp`
    2.  `sudo systemctl restart iio-sensor-proxy`
    3.  `sudo systemctl start hexagonrpcd-sdsp`
    4.  `sudo monitor-sensor`
* **Result:**
    * `+++ iio-sensor-proxy appeared`
    * `=== Has accelerometer`
    * Orientation changes detected correctly (e.g., `right-up`, `bottom-up`).

### 3. Modifications
* **Udev Rule Edit:** Executed `sed` to remove unsupported sensors. `cat` output confirms only `ssc-accel` is assigned to fastrpc nodes.
* **Systemd Edit:** Installed `nano` and successfully created the override file for `iio-sensor-proxy`.

## Confirmed Facts
* **Operational State:** The accelerometer hardware and drivers are functional; they correctly report orientation data when the software stack is in a specific state.
* **Start Order Sensitivity:** The system **succeeds** when `hexagonrpcd-sdsp` is started *after* `iio-sensor-proxy` is already running/restarted. The system **fails** to detect the accelerometer if `iio-sensor-proxy` is restarted while `hexagonrpcd-sdsp` is already running.
* **Persistence:** The working state does not persist across reboots.
* **Sensor Identification:** `monitor-sensor` identifies the device as having an accelerometer but reports "No ambient light sensor," "No proximity sensor," and "No compass" (consistent with the applied udev hack).
