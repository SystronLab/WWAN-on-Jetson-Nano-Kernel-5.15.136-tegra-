# Enabling WWAN Support on Jetson Nano (Kernel 5.15.136-tegra)

This guide enables MBIM based 5G modem support on a Jetson Nano running:

- JetPack 6
- L4T R36.3
- Kernel `5.15.136-tegra`

Tested with:

- Quectel RMU500-EK

The stock NVIDIA kernel did not include:

- `cdc_mbim`
- `cdc_ncm`
- `cdc_wdm`
- `qmi_wwan`

---

# 1. Verify Jetson Version

Check Jetson Linux release:

```bash
cat /etc/nv_tegra_release
```

Expected:

```text
# R36 (release), REVISION: 3.0
```

Check kernel:

```bash
uname -r
```

Expected:

```text
5.15.136-tegra
```

---

# 2. Install Kernel Headers

```bash
sudo apt update
sudo apt install nvidia-l4t-kernel-headers
```

---

# 3. Download NVIDIA Kernel Sources

Create workspace:

```bash
mkdir -p ~/nvidia-kernel-src
cd ~/nvidia-kernel-src
```

Download sources:

```bash
wget https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/sources/public_sources.tbz2
```

Extract:

```bash
tar xf public_sources.tbz2
cd Linux_for_Tegra/source
tar xf kernel_src.tbz2
```

Enter kernel source tree:

```bash
cd kernel/kernel-jammy-src
```

Verify:

```bash
ls
```

Expected folders:

```text
arch
drivers
include
net
scripts
```

---

# 4. Import Current Kernel Configuration

Jetson may not have `/boot/config-*`.

Use:

```bash
zcat /proc/config.gz > .config
```

---

# 5. Enable Required WWAN Modules

Enable MBIM support:

```bash
scripts/config --module USB_WDM
scripts/config --module USB_NET_CDC_NCM
scripts/config --module USB_NET_CDC_MBIM
```

Verify:

```bash
grep -E "USB_WDM|CDC_NCM|CDC_MBIM" .config
```

Expected:

```text
CONFIG_USB_WDM=m
CONFIG_USB_NET_CDC_NCM=m
CONFIG_USB_NET_CDC_MBIM=m
```

Optional QMI support:

```bash
scripts/config --module USB_NET_QMI_WWAN
```

---

# 6. Build Kernel Modules

Prepare build environment:

```bash
make olddefconfig
make modules_prepare
```

Build required modules:

```bash
make M=drivers/usb/class modules
make M=drivers/net/usb modules
```

---

# 7. Install Modules

Create module directories:

```bash
sudo mkdir -p /lib/modules/$(uname -r)/kernel/drivers/usb/class
sudo mkdir -p /lib/modules/$(uname -r)/kernel/drivers/net/usb
```

Copy modules:

```bash
sudo cp drivers/usb/class/cdc-wdm.ko \
/lib/modules/$(uname -r)/kernel/drivers/usb/class/

sudo cp drivers/net/usb/cdc_ncm.ko \
/lib/modules/$(uname -r)/kernel/drivers/net/usb/

sudo cp drivers/net/usb/cdc_mbim.ko \
/lib/modules/$(uname -r)/kernel/drivers/net/usb/
```

Optional:

```bash
sudo cp drivers/net/usb/qmi_wwan.ko \
/lib/modules/$(uname -r)/kernel/drivers/net/usb/
```

Refresh module database:

```bash
sudo depmod -a
```

---

# 8. Load Modules

```bash
sudo modprobe cdc_wdm
sudo modprobe cdc_ncm
sudo modprobe cdc_mbim
```

Optional:

```bash
sudo modprobe qmi_wwan
```

Verify:

```bash
lsmod | grep -E "mbim|cdc|qmi"
```

---

# 9. Make Modules Persistent Across Reboots

```bash
echo cdc_wdm | sudo tee -a /etc/modules
echo cdc_ncm | sudo tee -a /etc/modules
echo cdc_mbim | sudo tee -a /etc/modules
```

Optional:

```bash
echo qmi_wwan | sudo tee -a /etc/modules
```

---

# 10. Connect the Modem

Plug in the modem and verify detection:

```bash
lsusb
```

Expected:

```text
2c7c:0800 Quectel Wireless Solutions RMU500-EK
```

Check driver binding:

```bash
lsusb -t
```

Expected:

```text
Driver=cdc_mbim
```

Check WWAN device:

```bash
ls /dev/cdc-wdm*
```

Expected:

```text
/dev/cdc-wdm0
```

Check network interface:

```bash
ip a
```

Expected:

```text
wwan0
```

---

# Final Result

Working components:

- Jetson Nano
- Linux kernel `5.15.136-tegra`
- MBIM WWAN support enabled
- `/dev/cdc-wdm0`
- `wwan0`
- Quectel RMU500-EK connectivity working through MBIM mode
