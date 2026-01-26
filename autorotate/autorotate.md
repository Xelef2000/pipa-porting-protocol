
````markdown
# Xiaomi Pad 6 PMOS Auto-Rotate Diagnostic Summary (Updated)

## Objective
Diagnose and document the current state of auto-rotate and sensor handling on Xiaomi Pad 6 running postmarketOS (PMOS).

## Sensor Handling Overview
- On Xiaomi Pad 6, the accelerometer, gyroscope, and other sensors are **managed entirely by the Qualcomm DSP (Hexagon)**.
- The Linux kernel does **not have a direct IIO driver** for the IMU; no `/sys/bus/iio/devices/iio:deviceX/in_accel_*_raw` or `/in_anglvel_*_raw` exists.
- User-space sensor access is provided via **HexagonRPC** and then exposed to iio-sensor-proxy.

## Commands and Findings

### 1. IIO devices
```bash
ls /sys/bus/iio/devices/
````

```
iio:device0  iio:device1  iio:device2
```

```bash
for d in /sys/bus/iio/devices/iio:device*; do
    echo "$d: $(cat $d/name 2>/dev/null)"
done
```

```
/sys/bus/iio/devices/iio:device0: c440000.spmi:pmic@0:adc@3100
/sys/bus/iio/devices/iio:device1: c440000.spmi:pmic@2:adc@3100
/sys/bus/iio/devices/iio:device2: c440000.spmi:pmic@4:adc@3100
```

**Observation:** Only PMIC ADCs present; no accelerometer or gyroscope.

### 2. I2C Devices

```bash
for d in /sys/bus/i2c/devices/*; do
    if [ -f "$d/name" ]; then
        echo "$d: $(cat $d/name)"
    fi
done
```

```
/sys/bus/i2c/devices/1-0036: aw88261
/sys/bus/i2c/devices/1-0037: aw88261
/sys/bus/i2c/devices/11-0011: ktz8866
/sys/bus/i2c/devices/15-0042: fsa4480
/sys/bus/i2c/devices/16-0066: bq2597x-standalone
/sys/bus/i2c/devices/2-004c: 803
/sys/bus/i2c/devices/21-0010: ov13b10
/sys/bus/i2c/devices/3-0034: aw88261
/sys/bus/i2c/devices/3-0035: aw88261
/sys/bus/i2c/devices/8-0041: rx1665
/sys/bus/i2c/devices/i2c-1: Geni-I2C
/sys/bus/i2c/devices/i2c-11: Geni-I2C
/sys/bus/i2c/devices/i2c-15: Geni-I2C
/sys/bus/i2c/devices/i2c-16: Geni-I2C
/sys/bus/i2c/devices/i2c-2: Geni-I2C
/sys/bus/i2c/devices/i2c-20: dpu_dp_aux
/sys/bus/i2c/devices/i2c-21: Qualcomm-CCI
/sys/bus/i2c/devices/i2c-22: Qualcomm-CCI
/sys/bus/i2c/devices/i2c-3: Geni-I2C
/sys/bus/i2c/devices/i2c-8: Geni-I2C
```

**Observation:** No IMU device detected; all I2C devices are audio, PMIC, display, or generic buses.

### 3. Kernel Configuration

```bash
zcat /proc/config.gz | grep ICM426
```

```
# CONFIG_INV_ICM42600_I2C is not set
# CONFIG_INV_ICM42600_SPI is not set
```

```bash
lsmod | grep icm
```

*(no output)*

```bash
find /lib/modules/$(uname -r)/ -type f | grep icm426
```

*(no output)*
**Observation:** Kernel has no IMU driver; no module exists.

### 4. Manual Steps Tried (Unreliable)
I got auto rotate to work once using these steps but it was very unreliable.
I these steps are also incomplete/wrong since i didn't get it working since. (Bad documentation during the test)

1. Installed Qualcomm IPC package:

```bash
sudo apk add qrtr
```

2. Added DSP firmware to initramfs:

```bash
sudo tee /etc/mkinitfs/files/10-xiaomi-pipa-firmware <<EOF
/lib/firmware/qcom/sm8250/xiaomi/pipa/adsp.mbn
/lib/firmware/qcom/sm8250/xiaomi/pipa/slpi.mbn
/lib/firmware/qcom/sm8250/xiaomi/pipa/cdsp.mbn
EOF
sudo mkinitfs
```

3. **HexagonRPC daemon (`hexagonrpcd`) manual service creation:**

**Service file used:**

```ini
[Unit]
Description=Hexagon RPC Daemon (Sensors DSP)
After=local-fs.target
Wants=local-fs.target

[Service]
Type=simple
ExecStartPre=/bin/sh -c 'until [ -e /dev/fastrpc-sdsp ]; do sleep 0.2; done'
ExecStart=/usr/bin/hexagonrpcd -f /dev/fastrpc-sdsp -R /usr/share/qcom/sm8250/Xiaomi/pipa/dsp/sdsp/
Restart=always
RestartSec=1
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

* Initially tried `.device` dependency (`After=dev-fastrpc-sdsp.device`) which failed with timeout.
* Pre-start loop waits for `/dev/fastrpc-sdsp` and allows the daemon to start reliably.
* Logs indicate `Starting /usr/bin/hexagonrpcd (INIT_ATTACH) on /dev/fastrpc-sdsp`, confirming attachment to DSP.
* Can run manually with verbose flags for debugging DSP issues.

4. iio-sensor-proxy service configured to start after HexagonRPC, before display manager, restart on failure.

### 5. Current Systemd Status

```bash
systemctl status hexagonrpcd
```

```
Active: active (running)
ExecStart: /usr/bin/hexagonrpcd -f /dev/fastrpc-sdsp -R /usr/share/qcom/sm8250/Xiaomi/pipa/dsp/sdsp/
```

```bash
systemctl status iio-sensor-proxy2
```

```
Active: active (running)
No sensors detected:
=== No accelerometer
=== No ambient light sensor
=== No proximity sensor
=== No compass
```

**Observation:** HexagonRPC runs and attaches to DSP but iio-sensor-proxy sees no sensors. Logs show `g_hash_table_lookup: assertion 'hash_table != NULL' failed`, indicating the proxy did not initialize properly because the DSP did not provide sensor data.

### 6. Commands for Monitoring

```bash
monitor-sensor
```

Output:

```
Waiting for iio-sensor-proxy to appear
+++ iio-sensor-proxy appeared
=== No accelerometer
=== No ambient light sensor
=== No proximity sensor
=== No compass
```

### 7. DMA errors on for DSP


The SLPI remote processor boots successfully but immediately crashes with watchdog resets. Analysis revealed:

```
qcom,fastrpc: no reserved DMA memory for FASTRPC
```

**Diagnosis:** Missing reserved DMA memory carveout for FastRPC communication between Linux userspace and the SLPI processor.

#### Memory Requirements

The SLPI subsystem needs **two** separate memory regions:

| Region | Purpose | Size | Type |
|--------|---------|------|------|
| `slpi_mem` | SLPI firmware code and runtime data | ~47 MB | `no-map` (exclusive to SLPI) |
| `slpi_fastrpc_mem` | Shared DMA buffers for IPC | 16 MB | `reusable` (can be reclaimed) |


#### Device Tree Changes

##### 1. Add FastRPC Reserved Memory Region

**Location:** `reserved-memory` block in `sm8250-xiaomi-pipa.dts`

**Change:**
```dts
reserved-memory {
    /* Existing region - already present */
    slpi_mem: slpi@88c00000 {
        reg = <0x0 0x88c00000 0x0 0x2f00000>;
        no-map;
    };

    /* NEW: FastRPC DMA pool */
    slpi_fastrpc_mem: slpi-fastrpc {
        compatible = "shared-dma-pool";
        size = <0x0 0x1000000>;        /* 16 MB */
        alignment = <0x0 0x400000>;    /* 4 MB alignment */
        reusable;
    };
    
    /* Rest of regions... */
};
```

**Rationale:**
- Uses **dynamic allocation** - kernel finds free memory automatically
- `reusable` flag allows kernel to use memory when SLPI isn't active
- 16 MB is standard size for SLPI FastRPC pools on SM8250
- 4 MB alignment required by IOMMU

##### 2. Link Memory Regions to SLPI Node

**Location:** `&slpi` override section in `sm8250-xiaomi-pipa.dts`

**Change:**
```dts
&slpi {
    firmware-name = "qcom/sm8250/xiaomi/pipa/slpi.mbn";
    status = "okay";
    
    /* ADDED: Link both memory regions */
    memory-region = <&slpi_mem>, <&slpi_fastrpc_mem>;
    
    /* ADDED: Configure FastRPC subnode */
    glink-edge {
        fastrpc {
            compatible = "qcom,fastrpc";
            memory-region = <&slpi_fastrpc_mem>;
            label = "sdsp";
        };
    };
};
```


#### Before Fix
```
[   18.003965] qcom,fastrpc: no reserved DMA memory for FASTRPC
[   18.113055] remoteproc remoteproc0: crash detected in slpi: type watchdog
[   18.142299] remoteproc remoteproc0: handling crash #3 in slpi
```

#### After Fix
```
[    0.000000] OF: reserved mem: initialized node slpi-fastrpc, compatible id shared-dma-pool
[   12.450123] remoteproc remoteproc0: remote processor slpi is now up
[   12.567890] qcom,fastrpc: FastRPC initialized on /dev/fastrpc-sdsp
```

#### Other Remote Processors

The same pattern was applies to other DSPs on SM8250:

| Processor | Purpose | May Need Similar Fix |
|-----------|---------|---------------------|
| ADSP | Audio processing | If audio DSP features fail |
| CDSP | Compute (camera, CV) | If camera/AI features fail |
| SLPI | Sensors (this fix) | ✅ Primary focus |


### **Summary**

* Sensors are **entirely managed by Qualcomm DSP**; kernel IIO driver is missing.
* HexagonRPC (`hexagonrpcd`) must start and attach to `/dev/fastrpc-sdsp` to provide sensors.
* iio-sensor-proxy relies on HexagonRPC; currently runs but sees no sensors.
* Previous systemd `.device` dependency caused timeouts; pre-start loop works to start HexagonRPC reliably.
* Remaining issue: **DSP is running, but either firmware or HexagonRPC cannot expose sensors**, preventing iio-sensor-proxy from detecting them.

```
```
