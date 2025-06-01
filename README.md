# free5GC 5GC & UERANSIM UE / RAN Sample Configuration - UPG-VPP(VPP/DPDK UPF)
This describes a simple configuration for working free5GC and UPG-VPP.
In particular, see [here](https://github.com/s5uishida/install_vpp_upf_dpdk) for UPG-VPP configuration.

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Overview of free5GC 5GC Simulation Mobile Network](#overview)
- [Changes in configuration files of free5GC 5GC, UPG-VPP and UERANSIM UE / RAN](#changes)
  - [Changes in configuration files of free5GC 5GC C-Plane](#changes_cp)
  - [Changes in configuration files of UPG-VPP](#changes_up)
  - [Changes in configuration files of UERANSIM UE / RAN](#changes_ueransim)
    - [Changes in configuration files of RAN](#changes_ran)
    - [Changes in configuration files of UE (IMSI-001010000000000)](#changes_ue)
- [Network settings of free5GC 5GC, UPG-VPP and UERANSIM UE / RAN](#network_settings)
  - [Network settings of UPG-VPP and Data Network Gateway](#network_settings_up)
- [Build free5GC, UPG-VPP and UERANSIM](#build)
- [Run free5GC 5GC, UPG-VPP and UERANSIM UE / RAN](#run)
  - [Run UPG-VPP](#run_up)
  - [Run free5GC 5GC C-Plane](#run_cp)
  - [Run UERANSIM](#run_ueran)
    - [Start gNB](#start_gnb)
    - [Start UE](#start_ue)
- [Ping google.com](#ping)
  - [Case for going through DN 10.60.0.0/16](#ping_1)
- [Changelog (summary)](#changelog)

---

<a id="overview"></a>

## Overview of free5GC 5GC Simulation Mobile Network

This describes a simple configuration of C-Plane, UPG-VPP and Data Network Gateway for free5GC.
**Note that this configuration is implemented with Proxmox VE VMs.**

The following minimum configuration was set as a condition.
- One UPF and Data Network Gateway
- One UE and one DNN

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The 5GC / VPP/DPDK UPF / UE / RAN used are as follows.
- 5GC - free5GC v4.0.1 (2025.04.27) - https://github.com/free5gc/free5gc
- VPP/DPDK UPF - UPG-VPP v1.13.0 (2024.03.25) - https://github.com/travelping/upg-vpp
- UE / RAN - UERANSIM v3.2.7 (2025.04.28) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Mem<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | free5GC 5GC C-Plane | 192.168.0.141/24 | Ubuntu 24.04 | 1 | 2GB | 20GB |
| VM-UP | UPG-VPP U-Plane | 192.168.0.151/24 | Ubuntu **22.04** | 2 | 8GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |
| VM2 | UERANSIM RAN (gNodeB) | 192.168.0.131/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |
| VM3 | UERANSIM UE | 192.168.0.132/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |

The network interfaces of each VM are as follows.
**Note. Do not enable(up) any devices that will be under the control of DPDK.
These devices will be enabled and set IP addresses in the `init.conf` file of UPG-VPP.**
| VM | Device | Model | Linux Bridge | IP address | Interface | Under DPDK |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | ens18 | VirtIO | vmbr1 | 10.0.0.141/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.141/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr4 | 192.168.14.141/24 | N4 | -- |
| VM-UP | ~~ens18~~ | ~~VirtIO~~ | ~~vmbr1~~ | ~~10.0.0.151/24~~ | ~~(NAPT NW)~~ ***down*** | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.151/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.151/24 | N3 | x |
| | ens21 | VirtIO | vmbr4 | 192.168.14.151/24 | N4 | x |
| | ens22 | VirtIO | vmbr6 | 192.168.16.151/24 | N6 | x |
| VM-DN | ens18 | VirtIO | vmbr1 | 10.0.0.152/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.152/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr6 | 192.168.16.152/24 | N6 | -- |
| VM2 | ens18 | VirtIO | vmbr1 | 10.0.0.131/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.131/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.131/24 | N3 | -- |
| VM3 | ens18 | VirtIO | vmbr1 | 10.0.0.132/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.132/24 | (Mgmt NW) | -- |

Linux Bridges of Proxmox VE are as follows.
| Linux Bridge | Network CIDR | Interface |
| --- | --- | --- |
| vmbr1 | 10.0.0.0/24 | NAPT NW |
| mgbr0 | 192.168.0.0/24 | Mgmt NW |
| vmbr3 | 192.168.13.0/24 | N3 |
| vmbr4 | 192.168.14.0/24 | N4 |
| vmbr6 | 192.168.16.0/24 | N6 |

Set network instance to `internet`.
| Network Instance |
| --- |
| internet |

Subscriber Information (other information is the same) is as follows.  
**Note. Please select OP or OPc according to the setting of UERANSIM UE configuration file.**
| UE | IMSI | DNN | OP/OPc |
| --- | --- | --- | --- |
| UE | 001010000000000 | internet | OPc |

I registered these information with the free5GC WebUI.
In addition, [3GPP TS 35.208](https://www.3gpp.org/DynaReport/35208.htm) "4.3 Test Sets" is published by 3GPP as test data for the 3GPP authentication and key generation functions (MILENAGE).

The DN is as follows.
| DN | DNN | TUNnel interface of UE |
| --- | --- | --- |
| 10.60.0.0/16 | internet | uesimtun0 |

<a id="changes"></a>

## Changes in configuration files of free5GC 5GC, UPG-VPP and UERANSIM UE / RAN

Please refer to the following for building free5GC, UPG-VPP and UERANSIM respectively.
- free5GC v4.0.1 (2025.04.27) - https://free5gc.org/guide/
- UPG-VPP v1.13.0 (2024.03.25) - https://github.com/s5uishida/install_vpp_upf_dpdk
- UERANSIM v3.2.7 (2025.04.28) - https://github.com/aligungr/UERANSIM/wiki/Installation

<a id="changes_cp"></a>

### Changes in configuration files of free5GC 5GC C-Plane

The combination of DNN and S-NSSAI parameters can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- S-NSSAI

For the sake of simplicity, This time, only DNN will be changed. S-NSSAI of all UEs is fixed as `SST=1` and `SD=010203`.

- `free5gc/config/amfcfg.yaml`
```diff
--- amfcfg.yaml.orig    2024-10-13 05:09:24.000000000 +0900
+++ amfcfg.yaml 2025-05-04 19:42:17.462006265 +0900
@@ -5,7 +5,7 @@
 configuration:
   amfName: AMF # the name of this AMF
   ngapIpList:  # the IP list of N2 interfaces on this AMF
-    - 127.0.0.18
+    - 192.168.0.141
   ngapPort: 38412 # the SCTP port listened by NGAP
 
   # Service-based Interface (SBI) Configuration
@@ -30,22 +30,22 @@
   servedGuamiList:
     # <GUAMI> = <MCC><MNC><AMF ID>
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       amfId: cafe00 # AMF identifier (3 bytes hex string, range: 000000~FFFFFF)
 
   # the TAI (Tracking Area Identifier) list supported by this AMF
   supportTaiList:
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       tac: 000001 # Tracking Area Code (3 bytes hex string, range: 000000~FFFFFF)
 
   # the PLMNs (Public land mobile network) list supported by this AMF
   plmnSupportList:
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       snssaiList: # the S-NSSAI (Single Network Slice Selection Assistance Information) list supported by this AMF
         - sst: 1 # Slice/Service Type (uinteger, range: 0~255)
           sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
```
- `free5gc/config/ausfcfg.yaml`
```diff
--- ausfcfg.yaml.orig   2024-09-01 09:47:28.000000000 +0900
+++ ausfcfg.yaml        2024-09-01 09:55:54.000000000 +0900
@@ -16,10 +16,8 @@
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   nrfCertPem: cert/nrf.pem # NRF Certificate
   plmnSupportList: # the PLMNs (Public Land Mobile Network) list supported by this AUSF
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
-    - mcc: 123 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 45  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   groupId: ausfGroup001 # ID for the group of the AUSF
   eapAkaSupiImsiPrefix: false # including "imsi-" prefix or not when using the SUPI to do EAP-AKA' authentication

```
- `free5gc/config/nrfcfg.yaml`
```diff
--- nrfcfg.yaml.orig    2024-09-01 09:47:28.000000000 +0900
+++ nrfcfg.yaml 2024-09-01 09:56:10.000000000 +0900
@@ -18,8 +18,8 @@
       key: cert/root.key
     oauth: true
   DefaultPlmnId:
-    mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-    mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+    mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   serviceNameList: # the SBI services provided by this NRF, refer to TS 29.510
     - nnrf-nfm # Nnrf_NFManagement service
     - nnrf-disc # Nnrf_NFDiscovery service
```
- `free5gc/config/nssfcfg.yaml`
```diff
--- nssfcfg.yaml.orig   2024-09-01 09:47:28.000000000 +0900
+++ nssfcfg.yaml        2024-09-01 09:56:46.000000000 +0900
@@ -18,12 +18,12 @@
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   nrfCertPem: cert/nrf.pem # NRF Certificate
   supportedPlmnList: # the PLMNs (Public land mobile network) list supported by this NSSF
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   supportedNssaiInPlmnList: # Supported S-NSSAI List for each PLMN
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       supportedSnssaiList: # Supported S-NSSAIs of the PLMN
         - sst: 1 # Slice/Service Type (uinteger, range: 0~255)
           sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
```
- `free5gc/config/smfcfg.yaml`
```diff
--- smfcfg.yaml.orig    2024-10-13 05:09:24.000000000 +0900
+++ smfcfg.yaml 2025-05-04 19:44:21.139413362 +0900
@@ -42,16 +42,16 @@
 
   # Optional: PLMN IDs configuration.
   plmnList:
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   locality: area1 # Name of the location where a set of AMF, SMF, PCF and UPFs are located
 
   # PFCP (Packet Forwarding Control Protocol) configuration for N4 interface.
   pfcp:
     # addr config is deprecated in smf config v1.0.3, please use the following config
-    nodeID: 127.0.0.1 # the Node ID of this SMF
-    listenAddr: 127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
-    externalAddr: 127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
+    nodeID: 192.168.14.141 # the Node ID of this SMF
+    listenAddr: 192.168.14.141 # the IP/FQDN of N4 interface on this SMF (PFCP)
+    externalAddr: 192.168.14.141 # the IP/FQDN of N4 interface on this SMF (PFCP)
     assocFailAlertInterval: 10s
     assocFailRetryInterval: 30s
     heartbeatInterval: 10s
@@ -63,8 +63,8 @@
         type: AN # the type of the node (AN or UPF)
       UPF: # the name of the node
         type: UPF # the type of the node (AN or UPF)
-        nodeID: 127.0.0.8 # the Node ID of this UPF
-        addr: 127.0.0.8 # the IP/FQDN of N4 interface on this UPF (PFCP)
+        nodeID: 192.168.14.151 # the Node ID of this UPF
+        addr: 192.168.14.151 # the IP/FQDN of N4 interface on this UPF (PFCP)
         sNssaiUpfInfos: # S-NSSAI information list for this UPF
           - sNssai: # S-NSSAI (Single Network Slice Selection Assistance Information)
               sst: 1 # Slice/Service Type (uinteger, range: 0~255)
@@ -91,7 +91,7 @@
         interfaces: # Interface list for this UPF
           - interfaceType: N3 # the type of the interface (N3 or N9)
             endpoints: # the IP address of this N3/N9 interface on this UPF
-              - 127.0.0.8
+              - 192.168.13.151
             networkInstances: # Data Network Name (DNN)
               - internet
 
@@ -99,7 +99,8 @@
     links:
       - A: gNB1
         B: UPF
-
+  ulcl: false
+  nwInstFqdnEncoding: true
   # retransmission timer for PDU session modification command
   t3591:
     enable: true # true or false
```
This UPG-VPP requires `Network Instance` encoding in `PFCP Session Establishment Request`.
For more information, please see [here](https://github.com/s5uishida/enable_network_instance_encoding_free5gc_v3_3_0).

<a id="changes_up"></a>

### Changes in configuration files of UPG-VPP

See [here](https://github.com/s5uishida/install_vpp_upf_dpdk#conf) for the original files.

- `upg-vpp/startup.conf`  
There is no change.

- `upg-vpp/init.conf`  
There is no change.

<a id="changes_ueransim"></a>

### Changes in configuration files of UERANSIM UE / RAN

<a id="changes_ran"></a>

#### Changes in configuration files of RAN

- `UERANSIM/config/free5gc-gnb.yaml`
```diff
--- free5gc-gnb.yaml.orig       2024-12-11 20:31:30.000000000 +0900
+++ free5gc-gnb.yaml    2025-05-04 19:47:48.421731012 +0900
@@ -1,17 +1,17 @@
-mcc: '208'          # Mobile Country Code value
-mnc: '93'           # Mobile Network Code value (2 or 3 digits)
+mcc: '001'          # Mobile Country Code value
+mnc: '01'           # Mobile Network Code value (2 or 3 digits)
 
 nci: '0x000000010'  # NR Cell Identity (36-bit)
 idLength: 32        # NR gNB ID length in bits [22...32]
 tac: 1              # Tracking Area Code
 
-linkIp: 127.0.0.1   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
-ngapIp: 127.0.0.1   # gNB's local IP address for N2 Interface (Usually same with local IP)
-gtpIp: 127.0.0.1    # gNB's local IP address for N3 Interface (Usually same with local IP)
+linkIp: 192.168.0.131   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
+ngapIp: 192.168.0.131   # gNB's local IP address for N2 Interface (Usually same with local IP)
+gtpIp: 192.168.13.131    # gNB's local IP address for N3 Interface (Usually same with local IP)
 
 # List of AMF address information
 amfConfigs:
-  - address: 127.0.0.1
+  - address: 192.168.0.141
     port: 38412
 
 # List of supported S-NSSAIs by this gNB
```

<a id="changes_ue"></a>

#### Changes in configuration files of UE (IMSI-001010000000000)

- `UERANSIM/config/free5gc-ue.yaml`
```diff
--- free5gc-ue.yaml.orig        2025-03-16 15:49:12.000000000 +0900
+++ free5gc-ue.yaml     2025-05-04 19:49:27.180000029 +0900
@@ -1,9 +1,9 @@
 # IMSI number of the UE. IMSI = [MCC|MNC|MSISDN] (In total 15 digits)
-supi: 'imsi-208930000000001'
+supi: 'imsi-001010000000000'
 # Mobile Country Code value of HPLMN
-mcc: '208'
+mcc: '001'
 # Mobile Network Code value of HPLMN (2 or 3 digits)
-mnc: '93'
+mnc: '01'
 # SUCI Protection Scheme : 0 for Null-scheme, 1 for Profile A and 2 for Profile B
 protectionScheme: 0
 # Home Network Public Key for protecting with SUCI Profile A
@@ -31,7 +31,7 @@
 
 # List of gNB IP addresses for Radio Link Simulation
 gnbSearchList:
-  - 127.0.0.1
+  - 192.168.0.131
 
 # UAC Access Identities Configuration
 uacAic:
```

<a id="network_settings"></a>

## Network settings of free5GC 5GC, UPG-VPP and UERANSIM UE / RAN

<a id="network_settings_up"></a>

### Network settings of UPG-VPP and Data Network Gateway

See [this1](https://github.com/s5uishida/install_vpp_upf_dpdk#setup_up) and [this2](https://github.com/s5uishida/install_vpp_upf_dpdk#setup_dn).

<a id="build"></a>

## Build free5GC, UPG-VPP and UERANSIM

Please refer to the following for building free5GC, UPG-VPP and UERANSIM respectively.
- free5GC v4.0.1 (2025.04.27) - https://free5gc.org/guide/
- UPG-VPP v1.13.0 (2024.03.25) - https://github.com/s5uishida/install_vpp_upf_dpdk
- UERANSIM v3.2.7 (2025.04.28) - https://github.com/aligungr/UERANSIM/wiki/Installation

Install MongoDB on free5GC 5GC C-Plane machine.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

**Note. The installation guide also includes instructions on building the latest committed version.**

<a id="run"></a>

## Run free5GC 5GC, UPG-VPP and UERANSIM UE / RAN

First run UPG-VPP, then the 5GC and UERANSIM (UE & RAN implementation).

<a id="run_up"></a>

### Run UPG-VPP

See [this](https://github.com/s5uishida/install_vpp_upf_dpdk#run).

<a id="run_cp"></a>

### Run free5GC 5GC C-Plane

Next, run free5GC 5GC C-Plane.
Create the following shell script and run it.
```bash
#!/usr/bin/env bash

PID_LIST=()

NF_LIST="nrf amf smf udr pcf udm nssf ausf chf nef"

export GIN_MODE=release

for NF in ${NF_LIST}; do
    ./bin/${NF} &
    PID_LIST+=($!)
    sleep 1
done

function terminate()
{
    sudo kill -SIGTERM ${PID_LIST[${#PID_LIST[@]}-2]} ${PID_LIST[${#PID_LIST[@]}-1]}
    sleep 2
}

trap terminate SIGINT
wait ${PID_LIST}
```

The status of PFCP association between UPG-VPP and free5GC SMF is as follows.
```
vpp# show upf association 
Node: 192.168.14.141
  Recovery Time Stamp: 2025/05/04 20:19:47:000
  Sessions: 0
vpp# 
```

<a id="run_ueran"></a>

### Run UERANSIM

Here, the case of UE (IMSI-001010000000000) & RAN is described.
First, do an NG Setup between gNodeB and 5GC, then register the UE with 5GC and establish a PDU session.

Please refer to the following for usage of UERANSIM.

https://github.com/aligungr/UERANSIM/wiki/Usage

<a id="start_gnb"></a>

#### Start gNB

Start gNB as follows.
```
# ./nr-gnb -c ../config/free5gc-gnb.yaml
UERANSIM v3.2.7
[2025-05-04 20:20:23.444] [sctp] [info] Trying to establish SCTP connection... (192.168.0.141:38412)
[2025-05-04 20:20:23.447] [sctp] [info] SCTP connection established (192.168.0.141:38412)
[2025-05-04 20:20:23.447] [sctp] [debug] SCTP association setup ascId[7]
[2025-05-04 20:20:23.448] [ngap] [debug] Sending NG Setup Request
[2025-05-04 20:20:23.449] [ngap] [debug] NG Setup Response received
[2025-05-04 20:20:23.449] [ngap] [info] NG Setup procedure is successful
```
The free5GC C-Plane log when executed is as follows.
```
2025-05-04T20:20:23.484210294+09:00 [INFO][AMF][Ngap] [AMF] SCTP Accept from: 192.168.0.131:52590
2025-05-04T20:20:23.484978649+09:00 [INFO][AMF][Ngap] Create a new NG connection for: 192.168.0.131:52590
2025-05-04T20:20:23.485350846+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:52590] Handle NGSetupRequest
2025-05-04T20:20:23.485529381+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:52590] Send NG-Setup response
```

<a id="start_ue"></a>

#### Start UE

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/free5gc-ue.yaml
UERANSIM v3.2.7
[2025-05-04 20:21:01.304] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2025-05-04 20:21:01.304] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2025-05-04 20:21:01.305] [nas] [info] Selected plmn[001/01]
[2025-05-04 20:21:01.305] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2025-05-04 20:21:01.305] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2025-05-04 20:21:01.305] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2025-05-04 20:21:01.305] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2025-05-04 20:21:01.306] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2025-05-04 20:21:01.306] [nas] [debug] Sending Initial Registration
[2025-05-04 20:21:01.306] [rrc] [debug] Sending RRC Setup Request
[2025-05-04 20:21:01.306] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2025-05-04 20:21:01.307] [rrc] [info] RRC connection established
[2025-05-04 20:21:01.307] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2025-05-04 20:21:01.307] [nas] [info] UE switches to state [CM-CONNECTED]
[2025-05-04 20:21:01.360] [nas] [debug] Authentication Request received
[2025-05-04 20:21:01.360] [nas] [debug] Received SQN [00000000002A]
[2025-05-04 20:21:01.360] [nas] [debug] SQN-MS [000000000000]
[2025-05-04 20:21:01.384] [nas] [debug] Security Mode Command received
[2025-05-04 20:21:01.384] [nas] [debug] Selected integrity[2] ciphering[0]
[2025-05-04 20:21:01.542] [nas] [debug] Registration accept received
[2025-05-04 20:21:01.542] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2025-05-04 20:21:01.542] [nas] [debug] Sending Registration Complete
[2025-05-04 20:21:01.542] [nas] [info] Initial Registration is successful
[2025-05-04 20:21:01.542] [nas] [debug] Sending PDU Session Establishment Request
[2025-05-04 20:21:01.542] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2025-05-04 20:21:01.747] [nas] [debug] Configuration Update Command received
[2025-05-04 20:21:01.896] [nas] [debug] PDU Session Establishment Accept received
[2025-05-04 20:21:01.896] [nas] [info] PDU Session establishment is successful PSI[1]
[2025-05-04 20:21:01.916] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
The free5GC C-Plane log when executed is as follows.
```
2025-05-04T20:21:01.337030782+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:52590] Handle InitialUEMessage
2025-05-04T20:21:01.337086225+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:52590] New RanUe [RanUeNgapID:1][AmfUeNgapID:1]
2025-05-04T20:21:01.337118780+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:52590] 5GSMobileIdentity ["SUCI":"suci-0-001-01-0000-0-0-0000000000", err: <nil>]
2025-05-04T20:21:01.337188806+09:00 [INFO][AMF][CTX] New AmfUe [supi:][guti:00101cafe0000000001]
2025-05-04T20:21:01.337268888+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Deregistered] to [Deregistered]
2025-05-04T20:21:01.337276075+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Registration Request
2025-05-04T20:21:01.337282955+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] RegistrationType: Initial Registration
2025-05-04T20:21:01.337289812+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] MobileIdentity5GS: SUCI[suci-0-001-01-0000-0-0-0000000000]
2025-05-04T20:21:01.337299530+09:00 [INFO][AMF][Gmm] Handle event[Start Authentication], transition from [Deregistered] to [Authentication]
2025-05-04T20:21:01.337305873+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Authentication procedure
2025-05-04T20:21:01.338267569+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.339826218+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.342167154+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.343063195+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.344405574+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=AUSF |
2025-05-04T20:21:01.345439211+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.347018898+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.349750379+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.351734910+09:00 [INFO][AUSF][UeAuth] HandleUeAuthPostRequest
2025-05-04T20:21:01.351839880+09:00 [INFO][AUSF][UeAuth] Serving network authorized
2025-05-04T20:21:01.352596901+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.353486715+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.355634119+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.356545034+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.358012701+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AUSF&service-names=nudm-ueau&target-nf-type=UDM |
2025-05-04T20:21:01.360867944+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.361731395+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.364938604+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.366800366+09:00 [INFO][UDM][UEAU] Handle GenerateAuthDataRequest
2025-05-04T20:21:01.367510903+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.368874378+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.371553765+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.372050327+09:00 [INFO][UDM][Suci] scheme 0
2025-05-04T20:21:01.372259480+09:00 [INFO][UDM][Suci] SUPI type is IMSI
2025-05-04T20:21:01.372694950+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.373920558+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.375899675+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.376822964+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.377744214+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=UDM&target-nf-type=UDR |
2025-05-04T20:21:01.383628055+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2025-05-04T20:21:01.384381683+09:00 [INFO][UDM][Proc] ModifyAuthenticationSubscriptionRequest:  [{replace /sequenceNumber  { 00000000002b map[] 0 }}]
2025-05-04T20:21:01.386477327+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PATCH   | /nudr-dr/v2/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2025-05-04T20:21:01.386953178+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | POST    | /nudm-ueau/v1/suci-0-001-01-0000-0-0-0000000000/security-information/generate-auth-data |
2025-05-04T20:21:01.387526369+09:00 [INFO][AUSF][UeAuth] Add SuciSupiPair (suci-0-001-01-0000-0-0-0000000000, imsi-001010000000000) to map.
2025-05-04T20:21:01.387700171+09:00 [INFO][AUSF][UeAuth] Use 5G AKA auth method
2025-05-04T20:21:01.387926966+09:00 [INFO][AUSF][5gAka] XresStar = 3962386264663230636536306534333032373830376133336230303837633837
2025-05-04T20:21:01.388402172+09:00 [INFO][AUSF][GIN] | 201 |       127.0.0.1 | POST    | /nausf-auth/v1/ue-authentications |
2025-05-04T20:21:01.389153522+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Send Authentication Request
2025-05-04T20:21:01.389358125+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:52590] Send Downlink Nas Transport
2025-05-04T20:21:01.389507409+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Start T3560 timer
2025-05-04T20:21:01.390538207+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:52590] Handle UplinkNASTransport
2025-05-04T20:21:01.390710750+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:52590] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2025-05-04T20:21:01.390831850+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Authentication] to [Authentication]
2025-05-04T20:21:01.390989568+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Authentication Response
2025-05-04T20:21:01.391075303+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Stop T3560 timer
2025-05-04T20:21:01.391994149+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.393453645+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.396084553+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.397644128+09:00 [INFO][AUSF][5gAka] Auth5gAkaComfirmRequest
2025-05-04T20:21:01.397808993+09:00 [INFO][AUSF][5gAka] res*: 3962386264663230636536306534333032373830376133336230303837633837
Xres*: 3962386264663230636536306534333032373830376133336230303837633837
2025-05-04T20:21:01.397851289+09:00 [INFO][AUSF][5gAka] 5G AKA confirmation succeeded
2025-05-04T20:21:01.398638954+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.399524246+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.402527456+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.403998636+09:00 [INFO][UDM][UEAU] Handle ConfirmAuthDataRequest
2025-05-04T20:21:01.404706056+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.405909356+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.408531168+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.410927905+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v2/subscription-data/imsi-001010000000000/authentication-data/authentication-status |
2025-05-04T20:21:01.411285860+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-ueau/v1/imsi-001010000000000/auth-events |
2025-05-04T20:21:01.411829091+09:00 [INFO][AUSF][GIN] | 200 |       127.0.0.1 | PUT     | /nausf-auth/v1/ue-authentications/suci-0-001-01-0000-0-0-0000000000/5g-aka-confirmation |
2025-05-04T20:21:01.412291574+09:00 [INFO][AMF][Gmm] Handle event[Authentication Success], transition from [Authentication] to [SecurityMode]
2025-05-04T20:21:01.412663006+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Security Mode Command
2025-05-04T20:21:01.412847499+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:52590] Send Downlink Nas Transport
2025-05-04T20:21:01.412979785+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3560 timer
2025-05-04T20:21:01.414114192+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:52590] Handle UplinkNASTransport
2025-05-04T20:21:01.414234524+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:52590] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2025-05-04T20:21:01.414377416+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [SecurityMode] to [SecurityMode]
2025-05-04T20:21:01.414408132+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Security Mode Complete
2025-05-04T20:21:01.414438559+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3560 timer
2025-05-04T20:21:01.414567134+09:00 [INFO][AMF][Gmm] Handle event[SecurityMode Success], transition from [SecurityMode] to [ContextSetup]
2025-05-04T20:21:01.414644492+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle InitialRegistration
2025-05-04T20:21:01.415445012+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.417045420+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.419251183+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.420032628+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.421038544+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2025-05-04T20:21:01.421629598+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.423066438+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.427691849+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.429900633+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 nssai
2025-05-04T20:21:01.430024448+09:00 [INFO][UDM][SDM] Handle GetNssai
2025-05-04T20:21:01.431091977+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.432302511+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.434979626+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.436441249+09:00 [INFO][UDR][DataRepo] QueryAmDataProcedure: ueId: imsi-001010000000000, servingPlmnId: 00101
2025-05-04T20:21:01.437103851+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features= |
2025-05-04T20:21:01.438410952+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/nssai?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2025-05-04T20:21:01.438967642+09:00 [INFO][AMF][Gmm] RequestedNssai: &{Iei:47 Len:5 Buffer:[4 1 1 2 3]}
2025-05-04T20:21:01.439044547+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] RequestedNssai - ServingSnssai: &{Sst:1 Sd:010203}, HomeSnssai: <nil>
2025-05-04T20:21:01.439813486+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.441111762+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.443242899+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.443956912+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.445127218+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2025-05-04T20:21:01.445720473+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.446946208+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.449749575+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.451459055+09:00 [INFO][UDM][UECM] Handle RegistrationAmf3gppAccess
2025-05-04T20:21:01.451511557+09:00 [INFO][UDM][UECM] UEID: imsi-001010000000000
2025-05-04T20:21:01.452207715+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.453330824+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.455863174+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.458338480+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/amf-3gpp-access |
2025-05-04T20:21:01.458657878+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | PUT     | /nudm-uecm/v1/imsi-001010000000000/registrations/amf-3gpp-access |
2025-05-04T20:21:01.459721958+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.461322800+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.464073392+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.465322715+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 am-data
2025-05-04T20:21:01.465598798+09:00 [INFO][UDM][SDM] Handle GetAmData
2025-05-04T20:21:01.466339870+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.467578490+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.470280705+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.471883857+09:00 [INFO][UDR][DataRepo] QueryAmDataProcedure: ueId: imsi-001010000000000, servingPlmnId: 00101
2025-05-04T20:21:01.472855009+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2025-05-04T20:21:01.474012796+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/am-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2025-05-04T20:21:01.477041630+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.479180624+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.482384675+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.485740039+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 smf-select-data
2025-05-04T20:21:01.485986338+09:00 [INFO][UDM][SDM] Handle GetSmfSelectData
2025-05-04T20:21:01.487249124+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.488538013+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.491022367+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.492935541+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/smf-selection-subscription-data?supported-features= |
2025-05-04T20:21:01.493438939+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/smf-select-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2025-05-04T20:21:01.494696555+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.496206464+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.499150779+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.500338865+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 ue-context-in-smf-data
2025-05-04T20:21:01.500486849+09:00 [INFO][UDM][SDM] Handle GetUeContextInSmfData
2025-05-04T20:21:01.501358271+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.502531074+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.505011047+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.506708924+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/smf-registrations?supported-features= |
2025-05-04T20:21:01.507161106+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/ue-context-in-smf-data |
2025-05-04T20:21:01.508204602+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.509755416+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.512791283+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.516831987+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 sdm-subscriptions
2025-05-04T20:21:01.518766130+09:00 [INFO][UDM][SDM] Handle Subscribe
2025-05-04T20:21:01.519480514+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.520698482+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.523336360+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.526520716+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |
2025-05-04T20:21:01.526901455+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v2/imsi-001010000000000/sdm-subscriptions |
2025-05-04T20:21:01.527922735+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.529584636+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.531564152+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.532382949+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.533566136+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=PCF |
2025-05-04T20:21:01.534178658+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.535485900+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.538845536+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.543862241+09:00 [INFO][PCF][AmPol] Handle AM Policy Create Request
2025-05-04T20:21:01.544798989+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.545972399+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.548032533+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.548792635+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.549572074+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=UDR |
2025-05-04T20:21:01.550680173+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.551836070+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.554483693+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.556189000+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/policy-data/ues/imsi-001010000000000/am-data |
2025-05-04T20:21:01.557184160+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.558447058+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.560492440+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.561173783+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.562333325+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?guami=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22amfId%22%3A%22cafe00%22%7D&requester-nf-type=PCF&target-nf-type=AMF |
2025-05-04T20:21:01.562993817+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.564233272+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.567200279+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.568893139+09:00 [INFO][AMF][Comm] Handle AMF Status Change Subscribe Request
2025-05-04T20:21:01.569048232+09:00 [INFO][AMF][Comm] new AMF Status Subscription[1]
2025-05-04T20:21:01.569263827+09:00 [INFO][AMF][GIN] | 201 |       127.0.0.1 | POST    | /namf-comm/v1/subscriptions |
2025-05-04T20:21:01.569776714+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-am-policy-control/v1/policies |
2025-05-04T20:21:01.570396323+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Registration Accept
2025-05-04T20:21:01.570628003+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:52590] Send Initial Context Setup Request
2025-05-04T20:21:01.570934143+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3550 timer
2025-05-04T20:21:01.571516903+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:52590] Handle InitialContextSetupResponse
2025-05-04T20:21:01.571735649+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:52590] Handle InitialContextSetupResponse (RAN UE NGAP ID: 1)
2025-05-04T20:21:01.776469426+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:52590] Handle UplinkNASTransport
2025-05-04T20:21:01.776488841+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:52590] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2025-05-04T20:21:01.776529068+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [ContextSetup] to [ContextSetup]
2025-05-04T20:21:01.776537174+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Registration Complete
2025-05-04T20:21:01.776542861+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3550 timer
2025-05-04T20:21:01.776562460+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Configuration Update Command
2025-05-04T20:21:01.776569762+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:52590] Send Downlink Nas Transport
2025-05-04T20:21:01.776633048+09:00 [INFO][AMF][Gmm] Handle event[ContextSetup Success], transition from [ContextSetup] to [Registered]
2025-05-04T20:21:01.776696075+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:52590] Handle UplinkNASTransport
2025-05-04T20:21:01.776704650+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:52590] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2025-05-04T20:21:01.776731952+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Registered] to [Registered]
2025-05-04T20:21:01.776750621+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle UL NAS Transport
2025-05-04T20:21:01.776756305+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Transport 5GSM Message to SMF
2025-05-04T20:21:01.776766025+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Select SMF [snssai: {Sst:1 Sd:010203}, dnn: internet]
2025-05-04T20:21:01.777583675+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.778979232+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.781115935+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.781896107+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.782706934+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=NSSF |
2025-05-04T20:21:01.783353666+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.784687471+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.787254381+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.788882313+09:00 [INFO][NSSF][NsSel] Handle NSSelectionGet
2025-05-04T20:21:01.789226601+09:00 [WARN][NSSF][Util] No TA {"plmnId":{"mcc":"001","mnc":"01"},"tac":"000001"} in NSSF configuration
2025-05-04T20:21:01.789495366+09:00 [INFO][NSSF][GIN] | 200 |       127.0.0.1 | GET     | /nnssf-nsselection/v2/network-slice-information?nf-id=b09ace16-da74-41c8-bf4c-b83c3fdf9fcd&nf-type=AMF&slice-info-request-for-pdu-session=%7B%22sNssai%22%3A%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%2C%22roamingIndication%22%3A%22NON_ROAMING%22%7D&tai=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22tac%22%3A%22000001%22%7D |
2025-05-04T20:21:01.790028009+09:00 [WARN][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] nsiInformation is still nil, use default NRF[http://127.0.0.10:8000]
2025-05-04T20:21:01.790753678+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.792177449+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.794342122+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.795198142+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.796429025+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?dnn=internet&preferred-locality=area1&requester-nf-type=AMF&service-names=nsmf-pdusession&snssais=%5B%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%5D&target-nf-type=SMF&target-plmn-list=%5B%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%5D |
2025-05-04T20:21:01.797078396+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.799636728+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.804039321+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.806503393+09:00 [INFO][SMF][PduSess] Receive Create SM Context Request
2025-05-04T20:21:01.808236048+09:00 [INFO][SMF][PduSess] In HandlePDUSessionSMContextCreate
2025-05-04T20:21:01.808428432+09:00 [INFO][SMF][CTX] UrrPeriod: 30s
2025-05-04T20:21:01.808570966+09:00 [INFO][SMF][CTX] UrrThreshold: 500000
2025-05-04T20:21:01.810531487+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.811987551+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.814671142+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.815451199+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.816488372+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=UDM |
2025-05-04T20:21:01.817259444+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Send NF Discovery Serving UDM Successfully
2025-05-04T20:21:01.817679760+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.818836220+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.821720978+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.822961670+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 sm-data
2025-05-04T20:21:01.823289786+09:00 [INFO][UDM][SDM] Handle GetSmData
2025-05-04T20:21:01.824011882+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.825318948+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.827738088+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.828104830+09:00 [INFO][UDM][SDM] getSmDataProcedure: SUPI[imsi-001010000000000] PLMNID[00101] DNN[internet] SNssai[{"sst":1,"sd":"010203"}]
2025-05-04T20:21:01.829833810+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/sm-data?single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2025-05-04T20:21:01.830355421+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/sm-data?dnn=internet&plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D&single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2025-05-04T20:21:01.831308196+09:00 [INFO][SMF][GSM] In HandlePDUSessionEstablishmentRequest
2025-05-04T20:21:01+09:00 [INFO][NAS][Convert] ProtocolOrContainerList:  [0xc0002556a0 0xc0002556c0]
2025-05-04T20:21:01.831549355+09:00 [INFO][SMF][GSM] Protocol Configuration Options
2025-05-04T20:21:01.831670279+09:00 [INFO][SMF][GSM] &{[0xc0002556a0 0xc0002556c0]}
2025-05-04T20:21:01.831743459+09:00 [INFO][SMF][GSM] Didn't Implement container type IPAddressAllocationViaNASSignallingUL
2025-05-04T20:21:01.832595862+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.833844799+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.835984959+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.836741736+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.837896860+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-instance-id=b09ace16-da74-41c8-bf4c-b83c3fdf9fcd&target-nf-type=AMF |
2025-05-04T20:21:01.838412801+09:00 [INFO][SMF][Consumer] SendNFDiscoveryServingAMF ok
2025-05-04T20:21:01.838606886+09:00 [INFO][SMF][CTX] Allocated UE IP address: 10.60.0.1
2025-05-04T20:21:01.838698407+09:00 [INFO][SMF][CTX] Selected UPF: UPF
2025-05-04T20:21:01.838758240+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Allocated PDUAdress[10.60.0.1]
2025-05-04T20:21:01.839046753+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.840120843+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.842218657+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.842965186+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.844048180+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=SMF&target-nf-type=PCF |
2025-05-04T20:21:01.844716473+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.845813511+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.848812232+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.850606886+09:00 [INFO][PCF][SMpolicy] Handle CreateSmPolicy
2025-05-04T20:21:01.851399239+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.852694984+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.855226731+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.857470983+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/policy-data/ues/imsi-001010000000000/sm-data?dnn=internet&snssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2025-05-04T20:21:01.861033562+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.863850858+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.867598110+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.870204800+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/application-data/influenceData?dnns=internet&snssais=%5B%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%5D&supis=imsi-001010000000000 |
2025-05-04T20:21:01.870593543+09:00 [INFO][PCF][SMpolicy] Matched [0] trafficInfluDatas from UDR
2025-05-04T20:21:01.871365222+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.872603777+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.874931935+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.876082055+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v2/application-data/influenceData/subs-to-notify |
2025-05-04T20:21:01.877180440+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.878637544+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.880562253+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.881275853+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.881702712+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=BSF |
2025-05-04T20:21:01.882799356+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-smpolicycontrol/v1/sm-policies |
2025-05-04T20:21:01.884122461+09:00 [INFO][SMF][PduSess] CHF Selection for SMContext SUPI[imsi-001010000000000] PDUSessionID[1]
2025-05-04T20:21:01.885036288+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.886007216+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.887891096+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.888623435+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2025-05-04T20:21:01.889315283+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=CHF |
2025-05-04T20:21:01.889609758+09:00 [INFO][SMF][Charging] Handle SendConvergedChargingRequest
2025-05-04T20:21:01.889897856+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.890933632+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.893283164+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.898779003+09:00 [INFO][CHF][ChargingPost] HandleChargingdataInitial
2025-05-04T20:21:01.898996399+09:00 [INFO][CHF][ChargingPost] SMF charging event
2025-05-04T20:21:01.899312749+09:00 [ERRO][CHF][ChargingPost] Charging gateway fail to send CDR to billing domain dial tcp 127.0.0.1:2121: connect: connection refused
2025-05-04T20:21:01.899409996+09:00 [INFO][CHF][ChargingPost] Open CDR for UE imsi-001010000000000
2025-05-04T20:21:01.899435698+09:00 [INFO][CHF][ChargingPost] NewChfUe imsi-001010000000000
2025-05-04T20:21:01.899742955+09:00 [INFO][CHF][GIN] | 201 |       127.0.0.1 | POST    | /nchf-convergedcharging/v3/chargingdata |
2025-05-04T20:21:01.900419092+09:00 [INFO][SMF][Charging] Send Charging Data Request[Init] successfully
2025-05-04T20:21:01.900638941+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Install PCCRule[PccRuleId-1]
2025-05-04T20:21:01.900780173+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] No srcTcData and tgtTcData. Nothing to do
2025-05-04T20:21:01.900870738+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Has default path
2025-05-04T20:21:01.902559336+09:00 [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2025-05-04T20:21:01.904413685+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 sdm-subscriptions
2025-05-04T20:21:01.904629175+09:00 [INFO][UDM][SDM] Handle Subscribe
2025-05-04T20:21:01.906031241+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.908316920+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.909099084+09:00 [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2025-05-04T20:21:01.913346675+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.914522587+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.915885699+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |
2025-05-04T20:21:01.916437587+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v2/imsi-001010000000000/sdm-subscriptions |
2025-05-04T20:21:01.917002341+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.917176395+09:00 [INFO][SMF][PduSess] SDM Subscription Successful UE: imsi-001010000000000 SubscriptionId: 2
2025-05-04T20:21:01.917519638+09:00 [INFO][SMF][GIN] | 201 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts |
2025-05-04T20:21:01.918565712+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] create smContext[pduSessionID: 1] Success
2025-05-04T20:21:01.921171215+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.922985959+09:00 [INFO][AMF][Producer] Handle N1N2 Message Transfer Request
2025-05-04T20:21:01.923073032+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:52590] Send PDU Session Resource Setup Request
2025-05-04T20:21:01.923410848+09:00 [INFO][AMF][GIN] | 200 |       127.0.0.1 | POST    | /namf-comm/v1/ue-contexts/imsi-001010000000000/n1-n2-messages |
2025-05-04T20:21:01.925208586+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:52590] Handle PDUSessionResourceSetupResponse
2025-05-04T20:21:01.925357481+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:52590] Not comprehended IE ID 0x0079 (criticality: ignore)
2025-05-04T20:21:01.925385651+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:52590] Handle PDUSessionResourceSetupResponse (RAN UE NGAP ID: 1)
2025-05-04T20:21:01.926164778+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2025-05-04T20:21:01.927712094+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2025-05-04T20:21:01.930608970+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2025-05-04T20:21:01.932093794+09:00 [INFO][SMF][PduSess] Receive Update SM Context Request
2025-05-04T20:21:01.937900014+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Accepted Response from AN UPF
2025-05-04T20:21:01.938254287+09:00 [INFO][SMF][GIN] | 200 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts/urn:uuid:98897ae0-4465-46c5-948d-9d6002f06930/modify |
```
The PDU session establishment status of UPG-VPP is as follows.
```
vpp# show upf session 
CP F-SEID: 0x0000000000000001 (1) @ 192.168.14.141
UP F-SEID: 0x0000000000000001 (1) @ 192.168.14.151 (192.168.14.141 ::)
  PFCP Association: 0
  TEID assignment per choose ID
PDR: 1 @ 0x7f31ebdc51d8
  Precedence: 255
  PDI:
    Fields: 0000000d
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 2 (0x00000002)
            IPv4: 192.168.13.151
    UE IP address (source):
      IPv4 address: 10.60.0.1
    SDF Filter [1]:
      permit out ip from any to assigned 
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 1
  URR Ids: [1,2,7] @ 0x7f31ecc8c318
  QER Ids: [2,1] @ 0x7f31ecc8c338
PDR: 2 @ 0x7f31ebdc5258
  Precedence: 255
  PDI:
    Fields: 0000000c
    Source Interface: Core
    Network Instance: internet
    UE IP address (destination):
      IPv4 address: 10.60.0.1
    SDF Filter [1]:
      permit out ip from any to assigned 
  Outer Header Removal: no
  FAR Id: 2
  URR Ids: [1,2,7] @ 0x7f31ecc8c358
  QER Ids: [2,1] @ 0x7f31ecc8c378
FAR: 1
  Apply Action: 00000002 == [FORWARD]
  Forward:
    Network Instance: internet
    Destination Interface: 1
FAR: 2
  Apply Action: 00000002 == [FORWARD]
  Forward:
    Network Instance: internet
    Destination Interface: 0
    Outer Header Creation: [GTP-U/UDP/IPv4],TEID:00000001,IP:192.168.13.131
URR: 1
  Measurement Method: 0002 == [VOLUME]
  Reporting Triggers: 0003 == [PERIODIC REPORTING,VOLUME THRESHOLD]
  Status: 0 == []
  Start Time: 2025/05/04 20:23:31:885
  vTime of First Usage:       0.0000 
  vTime of Last Usage:        0.0000 
  Volume
    Up:    Measured:                    0, Theshold:               500000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Down:  Measured:                    0, Theshold:               500000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Total: Measured:                    0, Theshold:                    0, Pkts:          0
           Consumed:                    0, Quota:                       0
  Measurement Period:                   30 secs @ 2025/05/04 20:24:01:885, in     9.803 secs, handle 0x00000056
URR: 2
  Measurement Method: 0002 == [VOLUME]
  Reporting Triggers: 0003 == [PERIODIC REPORTING,VOLUME THRESHOLD]
  Status: 0 == []
  Start Time: 2025/05/04 20:23:31:885
  vTime of First Usage:       0.0000 
  vTime of Last Usage:        0.0000 
  Volume
    Up:    Measured:                    0, Theshold:               500000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Down:  Measured:                    0, Theshold:               500000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Total: Measured:                    0, Theshold:                    0, Pkts:          0
           Consumed:                    0, Quota:                       0
  Measurement Period:                   30 secs @ 2025/05/04 20:24:01:885, in     9.803 secs, handle 0x00000057
URR: 7
  Measurement Method: 0002 == [VOLUME]
  Reporting Triggers: 0002 == [VOLUME THRESHOLD]
  Status: 0 == []
  Start Time: 2025/05/04 20:21:01:885
  vTime of First Usage:       0.0000 
  vTime of Last Usage:        0.0000 
  Volume
    Up:    Measured:                    0, Theshold:               500000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Down:  Measured:                    0, Theshold:               500000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Total: Measured:                    0, Theshold:                    0, Pkts:          0
           Consumed:                    0, Quota:                       0
vpp# 
```
Looking at the console log of the `nr-ue` command, UE has been assigned the IP address `10.60.0.1` from free5GC 5GC.
```
[2025-05-04 20:21:01.916] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
8: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.60.0.1/24 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::15a2:ed7a:dcaf:1d51/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
...
```

<a id="ping"></a>

## Ping google.com

Specify the UE's TUNnel interface and try ping.

Please refer to the following for usage of TUNnel interface.

https://github.com/aligungr/UERANSIM/wiki/Usage

<a id="ping_1"></a>

### Case for going through DN 10.60.0.0/16

Run `tcpdump` on VM-DN and check that the packet goes through N6 (ens20).
- `ping google.com` on VM3 (UE)
```
# ping google.com -I uesimtun0 -n
PING google.com (172.217.161.78) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 172.217.161.78: icmp_seq=1 ttl=110 time=17.8 ms
64 bytes from 172.217.161.78: icmp_seq=2 ttl=110 time=18.3 ms
64 bytes from 172.217.161.78: icmp_seq=3 ttl=110 time=17.8 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i ens20 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens20, link-type EN10MB (Ethernet), snapshot length 262144 bytes
20:25:17.180932 IP 10.60.0.1 > 172.217.161.78: ICMP echo request, id 1347, seq 1, length 64
20:25:17.197743 IP 172.217.161.78 > 10.60.0.1: ICMP echo reply, id 1347, seq 1, length 64
20:25:18.182696 IP 10.60.0.1 > 172.217.161.78: ICMP echo request, id 1347, seq 2, length 64
20:25:18.200186 IP 172.217.161.78 > 10.60.0.1: ICMP echo reply, id 1347, seq 2, length 64
20:25:19.184123 IP 10.60.0.1 > 172.217.161.78: ICMP echo request, id 1347, seq 3, length 64
20:25:19.200985 IP 172.217.161.78 > 10.60.0.1: ICMP echo reply, id 1347, seq 3, length 64
```

You could specify the IP address assigned to the TUNnel interface to run almost any applications (iperf3 etc.) as in the following example using `nr-binder` tool.

- `curl google.com` on VM3 (UE)
```
# sh nr-binder 10.60.0.1 curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
- Run `tcpdump` on VM-DN
```
20:26:35.778047 IP 10.60.0.1.59979 > 172.217.161.78.80: Flags [S], seq 3265131590, win 65280, options [mss 1360,sackOK,TS val 4253256943 ecr 0,nop,wscale 7], length 0
20:26:35.793604 IP 172.217.161.78.80 > 10.60.0.1.59979: Flags [S.], seq 3662333825, ack 3265131591, win 65535, options [mss 1412,sackOK,TS val 2832494843 ecr 4253256943,nop,wscale 8], length 0
20:26:35.794403 IP 10.60.0.1.59979 > 172.217.161.78.80: Flags [.], ack 1, win 510, options [nop,nop,TS val 4253256960 ecr 2832494843], length 0
20:26:35.794403 IP 10.60.0.1.59979 > 172.217.161.78.80: Flags [P.], seq 1:74, ack 1, win 510, options [nop,nop,TS val 4253256960 ecr 2832494843], length 73: HTTP: GET / HTTP/1.1
20:26:35.810624 IP 172.217.161.78.80 > 10.60.0.1.59979: Flags [.], ack 74, win 1050, options [nop,nop,TS val 2832494860 ecr 4253256960], length 0
20:26:35.904918 IP 172.217.161.78.80 > 10.60.0.1.59979: Flags [P.], seq 1:774, ack 74, win 1050, options [nop,nop,TS val 2832494954 ecr 4253256960], length 773: HTTP: HTTP/1.1 301 Moved Permanently
20:26:35.905728 IP 10.60.0.1.59979 > 172.217.161.78.80: Flags [.], ack 774, win 504, options [nop,nop,TS val 4253257071 ecr 2832494954], length 0
20:26:35.906850 IP 10.60.0.1.59979 > 172.217.161.78.80: Flags [F.], seq 74, ack 774, win 504, options [nop,nop,TS val 4253257072 ecr 2832494954], length 0
20:26:35.922495 IP 172.217.161.78.80 > 10.60.0.1.59979: Flags [F.], seq 774, ack 75, win 1050, options [nop,nop,TS val 2832494972 ecr 4253257072], length 0
20:26:35.923216 IP 10.60.0.1.59979 > 172.217.161.78.80: Flags [.], ack 775, win 504, options [nop,nop,TS val 4253257089 ecr 2832494972], length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using UPG-VPP.

---

Now you could work free5GC with UPG-VPP.
I would like to thank the excellent developers and all the contributors of free5GC, UPG-VPP, VPP, DPDK and UERANSIM.

<a id="changelog"></a>

## Changelog (summary)

- [2025.05.04] Updated to UPG-VPP `v1.13.0`, free5GC `v4.0.1 (2025.04.27)` and UERANSIM `v3.2.7 (2025.04.28)`. Changed the VM environment from Virtualbox to Proxmox VE.
- [2024.03.24] Updated to UPG-VPP `v1.12.0`.
- [2023.11.10] Changed from [oai-cn5g-upf-vpp](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp) to its base [travelping/upg-vpp](https://github.com/travelping/upg-vpp).
- [2023.07.15] Enabled URR in `smfcfg.yaml`.
- [2023.06.27] The SMF fixed on 2023.06.27 now works with oai-cn5g-upf-vpp `v1.5.1`, so I updated to free5GC `v3.3.0`.
- [2023.06.15] Initial release.
