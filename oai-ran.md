# OAI RAN Deployment Guide

**Components:** gNB + NR UE (RFsim mode)
**Repository:** `openairinterface5g` (branch: `develop`)
**Host OS:** Ubuntu 22.04

## 1. Install Required Dependencies

```bash
sudo apt update
sudo apt install -y autoconf automake build-essential ccache cmake cpufrequtils \
  doxygen ethtool g++ git inetutils-tools libboost-all-dev libncurses-dev \
  libusb-1.0-0 libusb-1.0-0-dev libusb-dev python3-dev python3-mako \
  python3-numpy python3-requests python3-scipy python3-setuptools python3-ruamel.yaml
```

Also install FLTK (required for some OAI tools):

```bash
sudo apt install -y libforms-dev libforms-bin
```

## 2. Build and install `yaml-cpp` from source

```bash
cd
git clone https://github.com/jbeder/yaml-cpp.git
cd yaml-cpp
mkdir build && cd build
cmake .. -DYAML_BUILD_SHARED_LIBS=ON
make -j$(nproc)
sudo make install
cd
sudo rm -r yaml-cpp/
```

## 3. Clone and Prepare OAI RAN Source

```bash
cd
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git
cd openairinterface5g
git checkout develop
```

Load the OAI build environment:

```bash
source oaienv
```

Initialize dependencies:

```bash
./cmake_targets/build_oai -I
```

## 4. Configuration Changes

Apply these changes for consistent user profile across 5G Core and RAN:

```patch
diff --git a/targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf b/targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf
index 7f750d7259..4ae4fddca3 100644
--- a/targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf
+++ b/targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf
@@ -10,8 +10,8 @@ gNBs =
     gNB_name  =  "gNB-OAI";

     // Tracking area code, 0x0000 and 0xfffe are reserved values
-    tracking_area_code  =  1;
-    plmn_list = ({ mcc = 001; mnc = 01; mnc_length = 2; snssaiList = ({ sst = 1; }) });
+    tracking_area_code  =  0xa000;
+    plmn_list = ({ mcc = 208; mnc = 95; mnc_length = 2; snssaiList = ({ sst = 1; sd = 0xffffff; }) });

     nr_cellid = 12345678L;

diff --git a/targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf b/targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf
index 4cea93ea46..a6e099c89f 100644
--- a/targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf
+++ b/targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf
@@ -1,10 +1,10 @@
 uicc0 = {
-imsi = "2089900007487";
-key = "fec86ba6eb707ed08905757b1bb44b8f";
-opc= "C42449363BBAD02B66D16BC975D77CC1";
+imsi = "208950000000031";
+key = "0C0A34601D4F07677303652C0462535B";
+opc= "63bfa50ee6523365ff14c1f45f88737d";
 dnn= "oai";
 nssai_sst=1;
-nssai_sd=1;
+nssai_sd=0xffffff;
 }

 position0 = {
```

Or simply apply the patch file:

```bash
git apply patch-files/openairinterface5g.patch
```

## 5. Build the OAI RAN Components

```bash
cd cmake_targets/
./build_oai --ninja --nrUE --gNB --build-lib "nrscope"

# Optional - telnet management server support
./build_oai --build-lib telnetsrv
```

## 6. Run gNB (RF Simulator Mode)

Navigate to the build directory:

```bash
cd ran_build/build
```

Run the gNB:

```bash
sudo ./nr-softmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf \
  -E \
  --rfsim \
  --rfsimulator.serveraddr server \
  --gNBs.[0].min_rxtxtime 6
```

Append the following flags to add telnet server support:

```
--telnetsrv \
--telnetsrv.listenport 9090 \
--telnetsrv.shrmod o1
```

## 7. Run NR UE (RFsim Mode)

```bash
sudo ./nr-uesoftmodem \
  -r 106 \
  --numerology 1 \
  --band 78 \
  -C 3619200000 \
  --rfsim \
  --rfsimulator.serveraddr 127.0.0.1 \
  -E \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf
```
