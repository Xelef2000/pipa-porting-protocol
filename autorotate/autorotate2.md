# Xiaomi Pad 6 Auto-Rotate Debug Session Summary

## Objective
Debug the sensor stack by running hexagonrpcd, iio-sensor-proxy, and monitor-sensor manually to identify where the sensor data flow breaks down.

---

## Test 1: Stop Services and Verify DSP Device

### Commands:
```bash
sudo systemctl stop iio-sensor-proxy
sudo systemctl stop hexagonrpcd
ls -la /dev/fastrpc-sdsp
```

### Result:
```
crw-------    1 fastrpc  fastrpc    10, 265 Nov 22  1971 /dev/fastrpc-sdsp
```

**Finding:** DSP device node exists.

---

## Test 2: Run hexagonrpcd Without Sensor Flag

### Command:
```bash
sudo /usr/bin/hexagonrpcd -f /dev/fastrpc-sdsp -R /usr/share/qcom/sm8250/Xiaomi/pipa/dsp/sdsp/
```

### Result:
```
Starting /usr/bin/hexagonrpcd (INIT_ATTACH) on /dev/fastrpc-sdsp
Could not register ADSP default listener
```

**Finding:** hexagonrpcd starts but fails to register default listener. On second run (after stopping service), no error but also no output.

---

## Test 3: Check hexagonrpcd Process and Device Nodes

### Commands:
```bash
ps aux | grep hexagonrpcd
ls -la /dev/fastrpc-*
sudo dmesg | tail -50
```

### Results:
```
 2882 root      0:00 /usr/bin/hexagonrpcd -f /dev/fastrpc-sdsp -R /usr/share/qcom/sm8250/Xiaomi/pipa/dsp/sdsp/
 2929 felix     0:00 grep hexagonrpcd
```

```
crw-------    1 fastrpc  fastrpc    10, 266 Nov 22  1971 /dev/fastrpc-adsp
crw-------    1 fastrpc  fastrpc    10, 265 Nov 22  1971 /dev/fastrpc-sdsp
```

**dmesg output:** Only audio (aw88261) device messages, no FastRPC or sensor-related errors.

**Finding:** hexagonrpcd is running but not producing any sensor-related output.

---

## Test 4: Run iio-sensor-proxy Manually

### Command:
```bash
sudo /usr/libexec/iio-sensor-proxy
```

### Result:
```
** (iio-sensor-proxy:2964): WARNING **: 19:41:13.016: Not a switch [/sys/devices/platform/soc@0/c440000.spmi/spmi-0/0-00/c440000.spmi:pmic@0:adc@3100/iio:device0/../capabilities/sw]

** (iio-sensor-proxy:2964): WARNING **: 19:41:13.017: Invalid bitmask entry for /sys/devices/platform/soc@0/9c0000.geniqup/988000.i2c/i2c-2/2-004c/input/input3/event3

(iio-sensor-proxy:2964): GLib-CRITICAL **: 19:41:13.046: g_hash_table_lookup: assertion 'hash_table != NULL' failed

(iio-sensor-proxy:2964): GLib-CRITICAL **: 19:41:13.046: g_hash_table_insert_internal: assertion 'hash_table != NULL' failed

(iio-sensor-proxy:2964): GLib-CRITICAL **: 19:41:31.120: g_hash_table_lookup: assertion 'hash_table != NULL' failed

(iio-sensor-proxy:2964): GLib-CRITICAL **: 19:41:31.121: g_hash_table_insert_internal: assertion 'hash_table != NULL' failed

(iio-sensor-proxy:2964): GLib-CRITICAL **: 19:41:31.137: g_hash_table_lookup: assertion 'hash_table != NULL' failed

(iio-sensor-proxy:2964): GLib-CRITICAL **: 19:41:31.137: g_hash_table_insert_internal: assertion 'hash_table != NULL' failed

(iio-sensor-proxy:2964): GLib-CRITICAL **: 19:41:31.153: g_hash_table_lookup: assertion 'hash_table != NULL' failed

(iio-sensor-proxy:2964): GLib-CRITICAL **: 19:41:31.153: g_hash_table_insert_internal: assertion 'hash_table != NULL' failed

(iio-sensor-proxy:2964): GLib-CRITICAL **: 19:41:31.170: g_hash_table_lookup: assertion 'hash_table != NULL' failed

(iio-sensor-proxy:2964): GLib-CRITICAL **: 19:41:31.170: g_hash_table_insert_internal: assertion 'hash_table != NULL' failed
```

**Finding:** iio-sensor-proxy starts but hash table is NULL, meaning no sensors were discovered during initialization.

---

## Test 5: Run monitor-sensor

### Command:
```bash
sudo monitor-sensor
```

### Result:
```
    Waiting for iio-sensor-proxy to appear
+++ iio-sensor-proxy appeared
=== No accelerometer
=== No ambient light sensor
=== No proximity sensor
=== No compass
```

**Finding:** monitor-sensor can communicate with iio-sensor-proxy but confirms no sensors detected.

---

## Test 6: Examine Sensor Configuration Files

### Commands:
```bash
cat /usr/share/qcom/sm8250/Xiaomi/pipa/sensors/config/lsm6dso_0.json
```

### Result:
```json
{
  "config":
  {
    "hw_platform": ["QRD", "MTP", "Dragon", "Surf", "HDK", "PIPA"],
    "soc_id": ["336", "339", "341", "321", "339", "356"]
  },
  "lsm6dso_0":{
    "owner": "lsm6dso",
    ".accel":{
      "owner": "lsm6dso",
      ".config":{
        "owner": "lsm6dso",
        "is_dri":{ "type": "int", "ver": "0", "data": "1" },
        "hw_id":{ "type": "int", "ver": "0", "data": "0" },
        "res_idx":{ "type": "int", "ver": "0", "data": "2" },
        "sync_stream":{ "type": "int", "ver": "0", "data": "0" }
      }
    },
    ".gyro":{
      "owner": "lsm6dso",
      ".config":{
        "owner": "lsm6dso",
        "is_dri":{ "type": "int", "ver": "0", "data": "1" },
        "hw_id":{ "type": "int", "ver": "0", "data": "0" },
        "res_idx":{ "type": "int", "ver": "0", "data": "3" },
        "sync_stream":{ "type": "int", "ver": "0", "data": "0" }
      }
    },
    ".md":{
      "owner": "lsm6dso",
      ".config":{
        "owner": "lsm6dso",
        "is_dri":{ "type": "int", "ver": "0", "data": "1" },
        "hw_id":{ "type": "int", "ver": "0", "data": "0" },
        "res_idx":{ "type": "int", "ver": "0", "data": "0" },
        "sync_stream":{ "type": "int", "ver": "0", "data": "0" }
      }
    },
    ".temp":{
      "owner": "lsm6dso",
      ".config":{
        "owner": "lsm6dso",
        "is_dri":{ "type": "int", "ver": "0", "data": "0" },
        "hw_id":{ "type": "int", "ver": "0", "data": "0" },
        "res_idx":{ "type": "int", "ver": "0", "data": "2" },
        "sync_stream":{ "type": "int", "ver": "0", "data": "0" }
      }
    }
  }
}
```

### Commands:
```bash
ls -la /usr/share/qcom/sm8250/Xiaomi/pipa/sensors/registry/sensors_registry
```

### Result:
```
-rw-r--r--    1 root     root             0 Jan 15 19:27 /usr/share/qcom/sm8250/Xiaomi/pipa/sensors/registry/sensors_registry
```

**Finding:** LSM6DSO sensor configuration exists and includes "PIPA" in supported platforms. The sensors_registry file exists but is empty (0 bytes).

---

## Test 7: Check hexagonrpcd Usage for Verbose Options

### Command:
```bash
sudo /usr/bin/hexagonrpcd -v -v -v -f /dev/fastrpc-sdsp -R /usr/share/qcom/sm8250/Xiaomi/pipa/dsp/sdsp/
```

### Result:
```
/usr/bin/hexagonrpcd: unrecognized option: v
Usage: /usr/bin/hexagonrpcd [options] -f DEVICE
Server for FastRPC remote procedure calls from Qualcomm DSPs
Options:
	-c SHELL		Create a new pd running the specified ELF
	-d DSP		DSP name (default: )
	-f DEVICE	FastRPC device node to attach to
	-p PROGRAM	Run client program with shared file descriptor
	-R DIR		Root directory of served files (default: /usr/share/qcom/)
	-s		Attach to sensorspd
```

**Finding:** No verbose flag exists. Discovered `-s` flag for "Attach to sensorspd" which is likely needed.

---

## Test 8: Run hexagonrpcd with Sensors Flag

### Command:
```bash
sudo /usr/bin/hexagonrpcd -s -f /dev/fastrpc-sdsp -R /usr/share/qcom/sm8250/Xiaomi/pipa/dsp/sdsp/
```

### Result:
```
Starting /usr/bin/hexagonrpcd (INIT_ATTACH_SNS) on /dev/fastrpc-sdsp
Could not open /sys/devices/soc0/hw_platform: No such file or directory
Could not open /sys/devices/soc0/hw_platform: No such file or directory
Could not open /sys/devices/soc0/hw_platform: No such file or directory
Could not open /sys/devices/soc0/hw_platform: No such file or directory
Could not open /sys/devices/soc0/hw_platform: No such file or directory
Could not open /sys/devices/soc0/hw_platform: No such file or directory
Could not open /sys/devices/soc0/hw_platform: No such file or directory
Could not open /sys/devices/soc0/hw_platform: No such file or directory
Could not open /sys/devices/soc0/hw_platform: No such file or directory
Could not open /sys/devices/soc0/hw_platform: No such file or directory
Could not open /sys/devices/soc0/platform_subtype: No such file or directory
Could not open /sys/devices/soc0/platform_subtype: No such file or directory
Could not open /sys/devices/soc0/platform_subtype: No such file or directory
Could not open /sys/devices/soc0/platform_subtype: No such file or directory
Could not open /sys/devices/soc0/platform_subtype: No such file or directory
Could not open /sys/devices/soc0/platform_subtype: No such file or directory
Could not open /sys/devices/soc0/platform_subtype: No such file or directory
Could not open /sys/devices/soc0/platform_subtype: No such file or directory
Could not open /sys/devices/soc0/platform_subtype: No such file or directory
Could not open /sys/devices/soc0/platform_subtype: No such file or directory
Could not open /sys/devices/soc0/platform_subtype_id: No such file or directory
Could not open /sys/devices/soc0/platform_subtype_id: No such file or directory
Could not open /sys/devices/soc0/platform_subtype_id: No such file or directory
Could not open /sys/devices/soc0/platform_subtype_id: No such file or directory
Could not open /sys/devices/soc0/platform_subtype_id: No such file or directory
Could not open /sys/devices/soc0/platform_subtype_id: No such file or directory
Could not open /sys/devices/soc0/platform_subtype_id: No such file or directory
Could not open /sys/devices/soc0/platform_subtype_id: No such file or directory
Could not open /sys/devices/soc0/platform_subtype_id: No such file or directory
Could not open /sys/devices/soc0/platform_subtype_id: No such file or directory
Could not open /sys/devices/soc0/platform_version: No such file or directory
Could not open /sys/devices/soc0/platform_version: No such file or directory
Could not open /sys/devices/soc0/platform_version: No such file or directory
Could not open /sys/devices/soc0/platform_version: No such file or directory
Could not open /sys/devices/soc0/platform_version: No such file or directory
Could not open /sys/devices/soc0/platform_version: No such file or directory
Could not open /sys/devices/soc0/platform_version: No such file or directory
Could not open /sys/devices/soc0/platform_version: No such file or directory
Could not open /sys/devices/soc0/platform_version: No such file or directory
Could not open /sys/devices/soc0/platform_version: No such file or directory
Could not open /sys/devices/soc0/soc_id: No such file or directory
Could not open /sys/devices/soc0/soc_id: No such file or directory
Could not open /sys/devices/soc0/soc_id: No such file or directory
Could not open /sys/devices/soc0/soc_id: No such file or directory
Could not open /sys/devices/soc0/soc_id: No such file or directory
Could not open /sys/devices/soc0/soc_id: No such file or directory
Could not open /sys/devices/soc0/soc_id: No such file or directory
Could not open /sys/devices/soc0/soc_id: No such file or directory
Could not open /sys/devices/soc0/soc_id: No such file or directory
Could not open /sys/devices/soc0/soc_id: No such file or directory
Could not open /sys/devices/soc0/revision: No such file or directory
Could not open /sys/devices/soc0/revision: No such file or directory
Could not open /sys/devices/soc0/revision: No such file or directory
Could not open /sys/devices/soc0/revision: No such file or directory
Could not open /sys/devices/soc0/revision: No such file or directory
Could not open /sys/devices/soc0/revision: No such file or directory
Could not open /sys/devices/soc0/revision: No such file or directory
Could not open /sys/devices/soc0/revision: No such file or directory
Could not open /sys/devices/soc0/revision: No such file or directory
Could not open /sys/devices/soc0/revision: No such file or directory
Could not open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version: No such file or directory
Could not open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version: No such file or directory
Could not open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version: No such file or directory
Could not open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version: No such file or directory
Could not open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version: No such file or directory
Could not open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version: No such file or directory
Could not open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version: No such file or directory
Could not open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version: No such file or directory
Could not open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version: No such file or directory
Could not open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version: No such file or directory
Tried to open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version for writing
Tried to open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version for writing
Tried to open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version for writing
Tried to open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version for writing
Tried to open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version for writing
Tried to open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version for writing
Tried to open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version for writing
Tried to open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version for writing
Tried to open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version for writing
Tried to open /mnt/vendor/persist/sensors/registry/registry/../sns_reg_version for writing
Could not find local interface sns_registry
Unsupported handle: 4294967295
```

**Finding:** With `-s` flag, hexagonrpcd attempts sensor initialization (INIT_ATTACH_SNS) but fails due to:
1. Missing hardware platform identification files in `/sys/devices/soc0/`
2. Missing Android sensor registry path `/mnt/vendor/persist/sensors/registry/`
3. Final error: "Could not find local interface sns_registry" with invalid handle

---

## Test 9: Check Available SOC Information

### Commands:
```bash
ls -la /sys/devices/soc0/
find /sys/devices -name "*soc*" -type d 2>/dev/null
```

### Results:
```
total 0
drwxr-xr-x    3 root     root             0 Jan  1  1970 .
drwxr-xr-x   35 root     root             0 Jan  1  1970 ..
-r--r--r--    1 root     root          4096 Jan 26 19:47 family
-r--r--r--    1 root     root          4096 Jan 26 19:47 machine
drwxr-xr-x    2 root     root             0 Jan 26 19:47 power
-r--r--r--    1 root     root          4096 Jan 26 19:47 revision
-r--r--r--    1 root     root          4096 Jan 26 19:47 serial_number
-r--r--r--    1 root     root          4096 Jan 26 19:47 soc_id
lrwxrwxrwx    1 root     root             0 Jan  1  1970 subsystem -> ../../bus/soc
-rw-r--r--    1 root     root          4096 Jan  1  1970 uevent
```

**Finding:** `/sys/devices/soc0/` exists and has `soc_id` and `revision` files, but missing `hw_platform`, `platform_subtype`, `platform_subtype_id`, and `platform_version` that hexagonrpcd expects.

---

## Test 10: Check Android Sensor Registry Paths

### Commands:
```bash
ls -la /mnt/vendor/persist/sensors/ 2>/dev/null || echo "Path doesn't exist"
mount | grep persist
ls -la /usr/share/qcom/sm8250/Xiaomi/pipa/sensors/registry/
```

### Results:
```
Path doesn't exist
```

(No persist mount found)

Registry directory exists at `/usr/share/qcom/sm8250/Xiaomi/pipa/sensors/registry/` with multiple sensor configuration files including LSM6DSO.

**Finding:** The Android path `/mnt/vendor/persist/sensors/registry/` does not exist. No persist partition is mounted. The actual sensor registry files are located at `/usr/share/qcom/sm8250/Xiaomi/pipa/sensors/registry/`.

---

## Summary of Findings

1. **hexagonrpcd requires the `-s` flag** to attach to sensorspd (sensor protection domain)
2. **Missing hardware platform files** in `/sys/devices/soc0/`: `hw_platform`, `platform_subtype`, `platform_subtype_id`, `platform_version`
3. **Path mismatch**: hexagonrpcd expects Android paths (`/mnt/vendor/persist/sensors/`) but sensor configs are at `/usr/share/qcom/sm8250/Xiaomi/pipa/sensors/`
4. **Critical failure**: "Could not find local interface sns_registry" - hexagonrpcd cannot communicate with DSP sensor registry
5. **LSM6DSO sensor configuration exists** and correctly lists "PIPA" as supported platform
6. **iio-sensor-proxy hash table is NULL** - no sensors discovered during initialization
