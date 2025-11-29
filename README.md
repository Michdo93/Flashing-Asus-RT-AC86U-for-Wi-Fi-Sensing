# Flashing Asus RT-AC86U for WiFiSensing

## Installing ASUSWRT-Merlin

First, we need to install the ASUSWRT Merlin firmware for our RT-AC86U router. You can download it [here](https://sourceforge.net/projects/asuswrt-merlin/files/RT-AC86U/Release/RT-AC86U_386.14_2.zip/download). After that you have to unzip it.

We need it primarily to install Entware, later to permanently replace the `dhd.ko` file, and also to install programs such as ftp/sftp and scp, which we will need later.

First, go to your router's web interface and log in (e.g., http://192.168.1.1). Under Advanced Settings, click on Administration. Then click on Firmware Upgrade. 

Under certain circumstances, you can also make a backup beforehand. You should see something like “Manual Update” and “Upload” behind it. Click on it and then select the firmware file. After you confirm, a firmware update takes about 5 minutes and the router should then restart normally.

## Install Entware and any other necessary software

We now need a USB stick. Entware requires an external EXT2/3/4 partition.

The AC86U cannot install Entware on the internal memory (/jffs). This is how Merlin designed it → you need a USB stick.

No USB stick → no Entware → no tcpdump → no CSI logging.

### Prepare USB stick (EXT4)

You will need: Any USB stick (2–16 GB is sufficient)

Best to format on a PC → EXT4

Linux:

```
sudo mkfs.ext4 /dev/sdX
```

Windows:

Windows cannot format ext4 → Use a tool such as miniTool Partition Wizard, Paragon Linux File System Driver or USB on Linux Liveboot.

### Plug the USB stick into the router.

The AC86U automatically mounts the stick under:

```
/tmp/mnt/<NAME>
```

Check:

```
df -h
ls /tmp/mnt
```

You should see something like this:

```
/tmp/mnt/sda1
```

This is your ext4 stick!

### Entware installation

```
cd /jffs
chmod +x entware-setup.sh
./entware-setup.sh
```

When asked:

```
Do you wish to install the 64-bit version?
```

```
→ y
```

The following should then appear:

```
Info: /tmp/mnt/sda1 will be used for Entware
Installing...
```

If this completes → Success.

### Test whether Entware is running and install software

```
opkg update
opkg install tcpdump
```

Now you can install:

* tcpdump
* nc / ncat
* bash
* python3 (installable)

Everything you need to read CSI.

## How to patch RT-AC86U with Nexmon CSI

### Setting up your work environment

Use a Linux system (e.g., Ubuntu). Install dependencies:

```
sudo apt-get install git gawk qpdf flex bison build-essential unzip
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install libc6:i386 libncurses5-dev:i386 libstdc++6:i386 python2 python2-dev
```

Nexmon requires Python 2 and i386 libraries in some cases.

### Clone Nexmon Framework

In the next step you clone the Nexmon Framework and build it:

```
git clone https://github.com/seemoo-lab/nexmon.git
cd nexmon
source setup_env.sh
make
```
This extracts Broadcom firmware tools & basic tools. This extracts the original uCode, template RAM, etc.

### Clonse Nexmon_CSI

The relevant chip for the RT-AC86U is: BCM4366C0 (10.10.122.20 firmware)

```
cd ~/nexmon/patches/bcm4366c0/10_10_122_20/
git clone https://github.com/seemoo-lab/nexmon_csi.git
```

### Download ARM64 cross-toolchain

The AC86U uses ARM64 / aarch64, so you need the ASUS/ASUSWRT toolchain.

```
cd ~
git clone https://github.com/RMerl/am-toolchains.git
```

Relevant toolchain:

```
am-toolchains/brcm-arm-hnd/crosstools-aarch64-gcc-5.3-linux-4.1-glibc-2.22-binutils-2.25/
```

### Set toolchain environment variables

```
export TOOLCHAIN=~/am-toolchains/brcm-arm-hnd/crosstools-aarch64-gcc-5.3-linux-4.1-glibc-2.22-binutils-2.25
export PATH=$TOOLCHAIN/usr/bin:$PATH
export CC=aarch64-buildroot-linux-gnu-gcc
export AMCC=/home/ubuntu/am-toolchains/brcm-arm-hnd/crosstools-aarch64-gcc-5.3-linux-4.1-glibc-2.22-binutils-2.25/usr/bin/aarch64-buildroot-linux-gnu-
export LD_LIBRARY_PATH=/home/ubuntu/am-toolchains/brcm-arm-hnd/crosstools-aarch64-gcc-5.3-linux-4.1-glibc-2.22-binutils-2.25/usr/lib
```

This will build everything for ARM64.

### Compiling Nexutil (ARM64 version)

#### Deactivate the Android build

At first we have to do some changes.

```
mv ~/nexmon/utilities/nexutil/Makefile ~/nexmon/utilities/nexutil/Makefile.bak
nano ~/nexmon/utilities/nexutil/Makefile
```

```
GIT_VERSION := $(shell git describe --abbrev=4 --dirty --always --tags)
USE_VENDOR_CMD := 0

# -------------------------------------------
# Router/Native Build immer erzwingen
# (Android/NDK-Build komplett deaktivieren)
# -------------------------------------------
all: nexutil

# -------------------------------------------
# Native ARM64 / Router build
# -------------------------------------------
nexutil: nexutil.c ../libnexio/libnexio.a
ifeq ($(USE_VENDOR_CMD),1)
	$(CC) -static -o nexutil nexutil.c bcmutils.c bcmwifi_channels.c b64-encode.c b64-decode.c \
        -DUSE_VENDOR_CMD -DBUILD_ON_RPI -DVERSION=\"$(GIT_VERSION)\" \
        -I. -I./include -I../../patches/include -I../libnexio -L../libnexio/ -lnexio \
        $(shell pkg-config --cflags --libs libnl-3.0 libnl-genl-3.0) -lpthread
else
	$(CC) -static -o nexutil nexutil.c bcmutils.c bcmwifi_channels.c b64-encode.c b64-decode.c \
        -DBUILD_ON_RPI -DVERSION=\"$(GIT_VERSION)\" -DUSE_NETLINK \
        -I. -I./include -I../../patches/include -I../libnexio -L../libnexio/ -lnexio
endif

# -------------------------------------------
# Build libnexio native only (NO Android!)
# -------------------------------------------
../libnexio/libnexio.a: FORCE
	cd ../libnexio && make NDK_BUILD_DISABLE=1

# -------------------------------------------
# Install for router
# -------------------------------------------
install: nexutil
	cp $< /usr/bin/

# -------------------------------------------
clean:
	rm -Rf libs
	rm -Rf local
	rm -f nexutil

FORCE:

```

This will deactivate the Android build. It's not needed for your router.

#### Build libnexio manually in native mode (without Android)

```
cd ~/nexmon/utilities/libnexio
${AMCC}gcc -c libnexio.c -o libnexio.o -DBUILD_ON_RPI
${AMCC}ar rcs libnexio.a libnexio.o
```

```
cd ~/nexmon/utilities/nexutil
make
```

It generates nexutil as an aarch64 binary.

### Build CSI patch

```
cd ~/nexmon/patches/bcm4366c0/10_10_122_20/nexmon_csi
make


cd ~/nexmon/utilities/nexutil

echo "typedef uint32_t uint;" > types.h
sed -i 's/argp-extern/argp/' nexutil.c

${AMCC}gcc -static -o nexutil nexutil.c \
    bcmutils.c \
    bcmwifi_channels.c \
    b64-encode.c b64-decode.c \
    -DBUILD_ON_RPI -DVERSION=0 \
    -I. \
    -Iinclude \
    -I../../patches/include \
    -I../libnexio \
    -I../../utilities/dhdutil/include \
    -L../libnexio -lnexio
```

This compiles ucode patches, the dhd module, and CSI functions.

### Install firmware on the router

Connect the router to the PC via LAN. Activate SSH in the web interface (Administration → System). Install firmware:

```
make install-firmware REMOTEADDR=<router-ip>
```

```
make install-firmware REMOTEADDR=192.168.1.1
```

This loads the patched dhd.ko onto the router and activates it.

### Copy Nexutil to the router

```
scp ~/nexmon/utilities/nexutil/nexutil admin@<router-ip>:/jffs/
ssh admin@<router-ip> "chmod +x /jffs/nexutil"
```

### Activate CSI

Check channel parameters:

```
ssh admin@<router-ip>
/jffs/nexutil -k
```

Enable monitor mode:

```
/jffs/nexutil -m1
/jffs/nexutil -c1
/jffs/nexutil -k1
```

Router in monitor mode + CSI output:

```
wl -i eth6 monitor 1
wl -i eth6 chanspec 36/80
```

Configuring CSI on the host PC:

```
cd ~/nexmon/patches/bcm4366c0/10_10_122_20/nexmon_csi
./makecsiparams -c 1/80 -C 0x1 -N 1 -m 0x1 > csi_params.bin
```

Copy to router:

```
scp csi_params.bin admin@<router-ip>:/jffs/
```

Load into the router:

```
/jffs/nexutil -I eth6 -s500 -b csi_params.bin
```

eth6 is usually the 5 GHz interface. (At 2.4 GHz, it is usually eth5.)

### Read CSI (pcap)

On the host:

```
ssh admin@<router-ip> "tcpdump -i eth6 -s 0 -w /tmp/csi.pcap"
```

or locally via ssh forwarding:

```
ssh admin@<router-ip> "tcpdump -i eth6 -s 0 -U -w -" > csi.pcap
```

### Decoding CSI data

Deploying Nexmon CSI Tools:

```
cd ~/nexmon/patches/bcm4366c0/10_10_122_20/nexmon_csi/scripts
python2 read_csi.py csi.pcap
```

### Analyzing CSI files (Python)

With the Nexmon CSI Python Parser:

```
git clone https://github.com/seemoo-lab/nexmon_csi.git
cd nexmon_csi/python
python3 plot.py ../../examples/example.csi
```

## Python Codes

### Reading multiple routers in Python

Each router sends, for example, via UDP/LAN CSI to the VM:

Router:

```
tcpdump … | nc 192.168.1.100 5001   (Router 1)
tcpdump … | nc 192.168.1.100 5002   (Router 2)
tcpdump … | nc 192.168.1.100 5003   (Router 3)
```

Python receives and stores them separately:

```
import socket
import threading

def listen(port, buffer):
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.bind(("0.0.0.0", port))
    while True:
        data, _ = s.recvfrom(4096)
        buffer.append(data)

buf1, buf2, buf3 = [], [], []

threading.Thread(target=listen, args=(5001, buf1)).start()
threading.Thread(target=listen, args=(5002, buf2)).start()
threading.Thread(target=listen, args=(5003, buf3)).start()
```

### Sample code for Matplotlib (3 routers simultaneously)

#### Main Code

```
import matplotlib.pyplot as plt
import numpy as np

# Beispiel: 256 Subcarrier, 1 Snapshot pro Router
csi_router1 = np.random.rand(256)
csi_router2 = np.random.rand(256)
csi_router3 = np.random.rand(256)

plt.figure(figsize=(12, 6))

plt.plot(csi_router1, label="Router 1")
plt.plot(csi_router2, label="Router 2")
plt.plot(csi_router3, label="Router 3")

plt.title("CSI Amplituden – Vergleich der Router")
plt.xlabel("Subcarrier Index")
plt.ylabel("Amplitude")
plt.legend()
plt.grid(True)

plt.show()
```

Of course, you can also:

* use three separate diagrams
* one heat map per router
* update all routers in real time

#### Example of heat map per router

```
fig, axs = plt.subplots(3, 1, figsize=(12, 10))

axs[0].imshow(heatmap_router1, aspect='auto')
axs[1].imshow(heatmap_router2, aspect='auto')
axs[2].imshow(heatmap_router3, aspect='auto')

axs[0].set_title("Router 1 — CSI Heatmap")
axs[1].set_title("Router 2 — CSI Heatmap")
axs[2].set_title("Router 3 — CSI Heatmap")

plt.show()
```

