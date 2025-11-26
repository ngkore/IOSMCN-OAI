# IOS-MCN O1 Adapter Deployment Guide

**Component:** O1 Adapter for OAI gNB
**Version:** `v0.3.0` (Agartala release)
**Modes:** Docker or source or makefile build

## 1. Repository Preparation

```bash
git clone https://github.com/ios-mcn/ios-mcn-releases.git
cd ios-mcn-releases/Agartala/v0.3.0/RAN/source-code/
tar -xzf ios-mcn-ran-0.3.0.iosmcn.ran.tar.gz
mv ios-mcn-ran-0.3.0.iosmcn.ran ran
mv ran/ ~/
```

Navigate to the O1 adapter:

```bash
cd ~/ran/o1-adapter/
```

## 2. Update Configuration

Update configuration before building as this is common for deployment methods — **docker**, **source**, or **Makefile**.

### For docker or source-based builds:

Modify `docker/config/config.json`:

1. **Set host IP address**
2. **Set telnet server IP address** (use host IP in simulated environments)
3. **Align telnet ports**
   - `docker/scripts/servertest.py` uses **9090**
   - `config.json` defaults to **9091**
   - Change `config.json` to **9090**
4. Update the `ves.url` parameter to `http://<HOST_IP>:8080/eventListener/v7` to connect with the SMO VES Collector.
5. Update the `docker/scripts/servertest.py` to include the missing O1 configuration from CU. Either copy it from `docker/scripts/cu_servertest.py` or use this [servertest.py](servertest.py).

### For Makefile-based builds:

1. Add missing `"gnb-cu-id": 0` under the `info` section of `integration/config/config.json.template`:

   ```json
   "info": {
           "gnb-du-id": 0,
           "gnb-cu-id": 0, // this line to be added
           "cell-local-id": 0,
           "node-id": "gNB-Eurecom-5GNRBox-00001",
           "location-name": "MountPoint 05, Rack 234-17, Room 234, 2nd Floor, Körnerstraße 7, 10785 Berlin, Germany, Europe, Earth, Solar-System, Universe",
           "managed-by": "ManagementSystem=O-RAN-SC-ONAP-based-SMO",
           "managed-element-type": "NodeB",
           "model": "nr-softmodem",
           "unit-type": "gNB"
   }
   ```

2. Update the `integration/.env` file as the values got further propagated to `integration/config/config.json`.

   - Rename the incorrect variable from `OAI_OAI_TELNET_HOST` to `OAI_TELNET_HOST`
   - Add a new variable `VES_PORT` and set a default value of `8080`.
     - `VES_PORT=8080`
   - Change `VES_COLLECTOR_URL` value from `https://${VES_FQDN}/eventListener/v7` to `http://${VES_IP}:${VES_PORT}/eventListener/v7`

3. Change the `ports` parameter to `9090` in `integration/docker-compose-telnet.yaml`.

## 3. Deployment Method 1: Docker-Based

### 3.1 Build the Adapter Container

Supported flags for `build-adapter.sh`:

| Flag         | Description                                                              |
| ------------ | ------------------------------------------------------------------------ |
| `--dev`      | Build development container (debug tools included; not production-ready) |
| `--adapter`  | Build production adapter container                                       |
| `--no-cache` | Disable docker build cache                                               |

Build the adapter:

```bash
./build-adapter.sh --adapter
```

> [!TIP]
> Rebuild after config changes using the same command.

### 3.2 Start Telnet Server (Separate Terminal)

The adapter uses a telnet-based interface to communicate with the RAN.
Install dependencies:

```bash
sudo apt update
sudo apt install python3-pip
pip3 install telnetlib3
```

Run the telnet test server:

```bash
cd ran/o1-adapter/docker/scripts/
python3 servertest.py
```

### 3.3 Start the gNB Adapter

```bash
./start-adapter.sh --adapter
```

At this point, if the configuration is correct, the adapter will be listening for NETCONF clients and ready to communicate with the gNB’s telnet server.

## 4. Deployment Method 2: Source-Based Build

### 4.1 System Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y tzdata build-essential git cmake pkg-config unzip wget \
  libpcre2-dev zlib1g-dev libssl-dev autoconf libtool python3-pip

sudo apt install -y --no-install-recommends psmisc unzip wget openssl \
  openssh-client vsftpd openssh-server

pip3 install telnetlib3
```

### 4.2 NETCONF User Preparation

Create a dedicated system user:

```bash
sudo adduser --system netconf
echo "netconf:netconf!" | sudo chpasswd
```

SSH directory:

```bash
sudo mkdir -p /home/netconf/.ssh
sudo chmod 700 /home/netconf/.ssh
```

### 4.3 Configure OpenSSH for netconf User

Edit `/etc/ssh/sshd_config`:

```bash
sudo vim /etc/ssh/sshd_config
```

Append:

```
Match User netconf
       ChrootDirectory /
       X11Forwarding no
       AllowTcpForwarding no
       ForceCommand internal-sftp -d /ftp
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

### 4.4 Configure vsftpd

Prepare directories:

```bash
sudo mkdir -p /ftp
sudo chown -R netconf:nogroup /ftp
sudo mkdir -p /var/run/vsftpd/empty
sudo mkdir -p /run/sshd
```

Edit `/etc/vsftpd.conf`:

```bash
sudo vim /etc/vsftpd.conf
```

Ensure the following settings are present:

```
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
chroot_local_user=YES
allow_writeable_chroot=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
ssl_enable=NO
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO
```

Add `netconf` to `/etc/vsftpd.userlist`:

```bash
sudo vim /etc/vsftpd.userlist
```

Insert:

```
netconf
```

### 4.5 Prepare O1 Adapter Source Tree

The adapter expects configuration inside `src/`:

```bash
cp -r docker/config/ src/
```

### 4.6 Install NETCONF Dependencies

```bash
cd ran/o1-adapter/docker/scripts/
sudo ./netconf_dep_install.sh
sudo ldconfig
```

Run hostkey merge utilities:

```bash
sudo /usr/local/share/netopeer2/merge_hostkey.sh
sudo /usr/local/share/netopeer2/merge_config.sh
```

Fetch and install YANG models:

```bash
mkdir -p yang
./get-yangs.sh
sudo ./install-yangs.sh
```

### 4.7 Build the gNB Adapter (Source)

```bash
cd ../../src/
./build.sh
```

Rebuild after config changes:

```bash
rm gnb-adapter
./build.sh
```

Set environment variables:

```bash
export TERM=xterm-256color
```

### 4.8 Start Supporting Processes

#### Terminal 1: NETCONF Server

```bash
sudo netopeer2-server -d -v3 -t 60
```

#### Terminal 2: Telnet Test Server

```bash
cd ran/o1-adapter/docker/scripts/
python3 servertest.py
# Optional CU test:
# python3 cu_servertest.py
```

#### Terminal 3: gNB Adapter

```bash
cd ran/o1-adapter/src/
sudo ./gnb-adapter
```

## 5. Deployment Method 3: Makefile-Based Deployment

Through `Makefile`, we can deploy the **four different combinations** depending on what you want to integrate with:

1. **Adapter alone** (no gNB, no SMO)
2. **Adapter + telnet test server** (fake gNB)
3. **Adapter + SMO** (real O-RAN SMO integration, but still needs a telnet server or real gNB)
4. **Adapter + SMO + telnet test server** (full O-RAN environment with fake gNB)

### 5.1 Launch Adapter Using Makefile

#### Standalone Adapter

```bash
make run-o1-oai-adapter
```

#### Adapter + Telnet Test Server

```bash
make run-o1-adapter-telnet
```

> [!IMPORTANT]
> The below methods for SMO are not tested and still expected to fail. See [iosmcn-smo.md](iosmcn-smo.md) for more details on SMO deployment and integration.

#### Adapter + SMO

```bash
make run-o1-oai-adapter-smo
```

#### Adapter + SMO + Telnet Server

```bash
make run-o1-oai-adapter-smo-telnet
```

### 5.2 Teardown

```bash
make teardown
```

## 6. Testing the gNB Adapter with OAI gNB

### 6.1 Update `config.json` for simulated integration

- Set telnet server IP to `127.0.0.1`. Keep the host IP as is.
- Set port to `9090`
- Rebuild gnb-adapter (docker or source)

### 6.2 Start NETCONF server (only for source-based builds)

```bash
sudo netopeer2-server -d -v3 -t 60
```

Skip if using docker-based adapter.

### 6.3 Start gNB Adapter

Docker:

```bash
./start-adapter.sh --adapter
```

Source:

```bash
cd src/
sudo ./gnb-adapter
```

### 6.4 Start OAI gNB (RFsim + O1 Telnet)

```bash
sudo ./nr-softmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf \
  -E \
  --rfsim \
  --rfsimulator.serveraddr server \
  --gNBs.[0].min_rxtxtime 6 \
  --telnetsrv \
  --telnetsrv.listenport 9090 \
  --telnetsrv.shrmod o1
```
