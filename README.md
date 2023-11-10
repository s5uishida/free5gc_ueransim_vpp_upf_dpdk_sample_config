# free5GC 5GC & UERANSIM UE / RAN Sample Configuration - VPP-UPF with DPDK
This describes a simple configuration for working free5GC and VPP-UPF with DPDK.
In particular, see [here](https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1) for VPP-UPF with DPDK configuration.

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Overview of free5GC 5GC Simulation Mobile Network](#overview)
- [Changes in configuration files of free5GC 5GC, VPP-UPF and UERANSIM UE / RAN](#changes)
  - [Changes in configuration files of free5GC 5GC C-Plane](#changes_cp)
  - [Changes in configuration files of VPP-UPF](#changes_up)
  - [Changes in configuration files of UERANSIM UE / RAN](#changes_ueransim)
    - [Changes in configuration files of RAN](#changes_ran)
    - [Changes in configuration files of UE (IMSI-001010000000000)](#changes_ue)
- [Network settings of free5GC 5GC, VPP-UPF and UERANSIM UE / RAN](#network_settings)
  - [Network settings of VPP-UPF and Data Network Gateway](#network_settings_up)
- [Build free5GC, VPP-UPF and UERANSIM](#build)
- [Run free5GC 5GC, VPP-UPF and UERANSIM UE / RAN](#run)
  - [Run VPP-UPF](#run_up)
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

This describes a simple configuration of C-Plane, VPP-UPF and Data Network Gateway for free5GC.
**Note that this configuration is implemented with Virtualbox VMs.**

The following minimum configuration was set as a condition.
- One UPF and Data Network Gateway
- One UE and one DNN

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The 5GC / VPP-UPF / UE / RAN used are as follows.
- 5GC - free5GC v3.3.0 (2023.11.10) - https://github.com/free5gc/free5gc
- VPP-UPF - UPG-VPP v1.10.0 (2023.11.10) - https://github.com/travelping/upg-vpp
- UE / RAN - UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Memory<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | free5GC 5GC C-Plane | 192.168.0.141/24 | Ubuntu 22.04 | 1 | 2GB | 20GB |
| VM-UP | UPG-VPP U-Plane | 192.168.0.151/24 | Ubuntu 20.04 | 2 | 8GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |
| VM2 | UERANSIM RAN (gNodeB) | 192.168.0.131/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |
| VM3 | UERANSIM UE | 192.168.0.132/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |

The network interfaces of each VM are as follows.
**Note. Do not enable(up) any devices that will be under the control of DPDK.
These devices will be enabled and set IP addresses in the `init.conf` file of VPP-UPF.**
| VM | Device | Network Adapter | IP address | Interface | Under DPDK |
| --- | --- | --- | --- | --- | --- |
| VM1 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.141/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.14.141/24 | N4 | -- |
| VM-UP | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.151/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.151/24 | N3 | x |
| | enp0s10 | NAT Network | 192.168.14.151/24 | N4 | x |
| | enp0s16 | NAT Network | 192.168.16.151/24 | N6 | x |
| VM-DN | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.152/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.16.152/24 | N6 | -- |
| VM2 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.131/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.131/24 | N3 | -- |
| VM3 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.132/24 | (Mgmt NW) | -- |

NAT networks of Virtualbox  are as follows.
| Network Name | Network CIDR |
| --- | --- |
| N3 | 192.168.13.0/24 |
| N4 | 192.168.14.0/24 |
| N6 | 192.168.16.0/24 |

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

## Changes in configuration files of free5GC 5GC, VPP-UPF and UERANSIM UE / RAN

Please refer to the following for building free5GC, VPP-UPF and UERANSIM respectively.
- free5GC v3.3.0 (2023.11.10) - https://free5gc.org/guide/
- UPG-VPP v1.10.0 (2023.11.10) - https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1
- UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM/wiki/Installation

<a id="changes_cp"></a>

### Changes in configuration files of free5GC 5GC C-Plane

The combination of DNN and S-NSSAI parameters can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- S-NSSAI

For the sake of simplicity, This time, only DNN will be changed. S-NSSAI of all UEs is fixed as `SST=1` and `SD=010203`.

- `free5gc/config/amfcfg.yaml`
```diff
--- amfcfg.yaml.orig    2023-11-10 23:13:08.456735624 +0900
+++ amfcfg.yaml 2023-11-10 23:26:10.862955424 +0900
@@ -5,7 +5,7 @@
 configuration:
   amfName: AMF # the name of this AMF
   ngapIpList:  # the IP list of N2 interfaces on this AMF
-    - 127.0.0.18
+    - 192.168.0.141
   ngapPort: 38412 # the SCTP port listened by NGAP
   sbi: # Service-based interface information
     scheme: http # the protocol for sbi (http or https)
@@ -24,18 +24,18 @@
   servedGuamiList: # Guami (Globally Unique AMF ID) list supported by this AMF
     # <GUAMI> = <MCC><MNC><AMF ID>
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       amfId: cafe00 # AMF identifier (3 bytes hex string, range: 000000~FFFFFF)
   supportTaiList:  # the TAI (Tracking Area Identifier) list supported by this AMF
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       tac: 000001 # Tracking Area Code (3 bytes hex string, range: 000000~FFFFFF)
   plmnSupportList: # the PLMNs (Public land mobile network) list supported by this AMF
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
--- ausfcfg.yaml.orig   2023-11-10 23:13:08.456735624 +0900
+++ ausfcfg.yaml        2023-11-10 23:26:26.862283097 +0900
@@ -15,8 +15,8 @@
     - nausf-auth # Nausf_UEAuthentication service
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   plmnSupportList: # the PLMNs (Public Land Mobile Network) list supported by this AUSF
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
     - mcc: 123 # Mobile Country Code (3 digits string, digit: 0~9)
       mnc: 45  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   groupId: ausfGroup001 # ID for the group of the AUSF
```
- `free5gc/config/nrfcfg.yaml`
```diff
--- nrfcfg.yaml.orig    2023-11-10 23:13:08.457735694 +0900
+++ nrfcfg.yaml 2023-11-10 23:26:39.252494730 +0900
@@ -14,8 +14,8 @@
       pem: cert/nrf.pem # NRF TLS Certificate
       key: cert/nrf.key # NRF TLS Private key
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
--- nssfcfg.yaml.orig   2023-11-10 23:13:08.457735694 +0900
+++ nssfcfg.yaml        2023-11-10 23:26:55.878733123 +0900
@@ -17,12 +17,12 @@
     - nnssf-nssaiavailability # Nnssf_NSSAIAvailability service
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
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
--- smfcfg.yaml.orig    2023-11-10 23:13:08.457735694 +0900
+++ smfcfg.yaml 2023-11-10 23:27:06.151859167 +0900
@@ -34,22 +34,22 @@
             ipv4: 8.8.8.8
             ipv6: 2001:4860:4860::8888
   plmnList: # the list of PLMN IDs that this SMF belongs to (optional, remove this key when unnecessary)
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   locality: area1 # Name of the location where a set of AMF, SMF, PCF and UPFs are located
   pfcp: # the IP address of N4 interface on this SMF (PFCP)
     # addr config is deprecated in smf config v1.0.3, please use the following config
-    nodeID: 127.0.0.1 # the Node ID of this SMF
-    listenAddr: 127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
-    externalAddr: 127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
+    nodeID: 192.168.14.141 # the Node ID of this SMF
+    listenAddr: 192.168.14.141 # the IP/FQDN of N4 interface on this SMF (PFCP)
+    externalAddr: 192.168.14.141 # the IP/FQDN of N4 interface on this SMF (PFCP)
   userplaneInformation: # list of userplane information
     upNodes: # information of userplane node (AN or UPF)
       gNB1: # the name of the node
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
@@ -72,7 +72,7 @@
         interfaces: # Interface list for this UPF
           - interfaceType: N3 # the type of the interface (N3 or N9)
             endpoints: # the IP address of this N3/N9 interface on this UPF
-              - 127.0.0.8
+              - 192.168.13.151
             networkInstances:  # Data Network Name (DNN)
               - internet
     links: # the topology graph of userplane, A and B represent the two nodes of each link
@@ -89,8 +89,10 @@
     expireTime: 16s   # default is 6 seconds
     maxRetryTimes: 3 # the max number of retransmission
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
-  #urrPeriod: 10 # default usage report period in seconds
-  #urrThreshold: 1000 # default usage report threshold in bytes
+  urrPeriod: 10 # default usage report period in seconds
+  urrThreshold: 1000 # default usage report threshold in bytes
+  ulcl: false
+  nwInstFqdnEncoding: true
 
 logger: # log output setting
   enable: true # true or false
```
This VPP-UPF requires `Network Instance` encoding in `PFCP Session Establishment Request`, so it is necessary to add the following line to the `smfcfg.yaml` of free5GC.
```yaml
...
configuration:
  ...
  nwInstFqdnEncoding: true <--
...
```
Please refer to the following 3GPP specifications for information on this matter.
```
3GPP TS 29.244 LTE;5G;Interface between the Control Plane and the User Plane nodes
- 8 Information Elements
  - 8.2 Information Elements
    - 8.2.4 Network Instance

3GPP TS 23.003 LTE;5G;Numbering, addressing and identification
- 9A Definition of Data Network Name
  - 9.1 Structure of APN

The encoding of the APN shall follow the Name Syntax defined in RFC 2181 [18], RFC 1035 [19] and RFC 1123 [20].
The APN consists of one or more labels. Each label is coded as a one octet length field followed by that number of
octets coded as 8 bit ASCII characters. Following RFC 1035 [19] the labels shall consist only of the alphabetic
characters (A-Z and a-z), digits (0-9) and the hyphen (-). Following RFC 1123 [20], the label shall begin and end with
either an alphabetic character or a digit. The case of alphabetic characters is not significant. The APN is not terminated
by a length byte of zero.
```

<a id="changes_up"></a>

### Changes in configuration files of VPP-UPF

See [here](https://github.com/s5uishida/install_vpp_upf_dpdk#changes_up) for the original files.

- `openair-upf/startup.conf`  
There is no change.

- `openair-upf/init.conf`  
There is no change.

<a id="changes_ueransim"></a>

### Changes in configuration files of UERANSIM UE / RAN

<a id="changes_ran"></a>

#### Changes in configuration files of RAN

- `UERANSIM/config/free5gc-gnb.yaml`
```diff
--- free5gc-gnb.yaml.orig       2021-02-12 09:47:56.000000000 +0900
+++ free5gc-gnb.yaml    2023-06-15 22:24:00.297158446 +0900
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
--- free5gc-ue.yaml.orig        2023-05-10 14:51:54.000000000 +0900
+++ free5gc-ue.yaml     2023-06-15 22:24:48.016816988 +0900
@@ -1,9 +1,9 @@
 # IMSI number of the UE. IMSI = [MCC|MNC|MSISDN] (In total 15 digits)
-supi: 'imsi-208930000000003'
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
@@ -28,7 +28,7 @@
 
 # List of gNB IP addresses for Radio Link Simulation
 gnbSearchList:
-  - 127.0.0.1
+  - 192.168.0.131
 
 # UAC Access Identities Configuration
 uacAic:
```

<a id="network_settings"></a>

## Network settings of free5GC 5GC, VPP-UPF and UERANSIM UE / RAN

<a id="network_settings_up"></a>

### Network settings of VPP-UPF and Data Network Gateway

See [this1](https://github.com/s5uishida/install_vpp_upf_dpdk#setup_up) and [this2](https://github.com/s5uishida/install_vpp_upf_dpdk#setup_dn).

<a id="build"></a>

## Build free5GC, VPP-UPF and UERANSIM

Please refer to the following for building free5GC, VPP-UPF and UERANSIM respectively.
- free5GC v3.3.0 (2023.11.10) - https://free5gc.org/guide/
- UPG-VPP v1.10.0 (2023.11.10) - https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1
- UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM/wiki/Installation

Install MongoDB on free5GC 5GC C-Plane machine.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

**Note. The installation guide also includes instructions on building the latest committed version.**

<a id="run"></a>

## Run free5GC 5GC, VPP-UPF and UERANSIM UE / RAN

First run VPP-UPF, then the 5GC and UERANSIM (UE & RAN implementation).

<a id="run_up"></a>

### Run VPP-UPF

See [this](https://github.com/s5uishida/install_vpp_upf_dpdk#run_upg_vpp).

<a id="run_cp"></a>

### Run free5GC 5GC C-Plane

Next, run free5GC 5GC C-Plane.
Create the following shell script and run it.
```bash
#!/usr/bin/env bash

PID_LIST=()

NF_LIST="nrf amf smf udr pcf udm nssf ausf chf"

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

The status of PFCP association between VPP-UPF and free5GC SMF is as follows.
```
vpp# show upf association 
Node: 192.168.14.141
  Recovery Time Stamp: 2023/11/10 23:42:15:000
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
UERANSIM v3.2.6
[2023-11-10 23:42:52.064] [sctp] [info] Trying to establish SCTP connection... (192.168.0.141:38412)
[2023-11-10 23:42:52.066] [sctp] [info] SCTP connection established (192.168.0.141:38412)
[2023-11-10 23:42:52.067] [sctp] [debug] SCTP association setup ascId[6]
[2023-11-10 23:42:52.067] [ngap] [debug] Sending NG Setup Request
[2023-11-10 23:42:52.069] [ngap] [debug] NG Setup Response received
[2023-11-10 23:42:52.069] [ngap] [info] NG Setup procedure is successful
```
The free5GC C-Plane log when executed is as follows.
```
2023-11-10T23:42:52.046448861+09:00 [INFO][AMF][Ngap] [AMF] SCTP Accept from: 192.168.0.131:57161
2023-11-10T23:42:52.047157586+09:00 [INFO][AMF][Ngap] Create a new NG connection for: 192.168.0.131:57161
2023-11-10T23:42:52.047853632+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:57161] Handle NGSetupRequest
2023-11-10T23:42:52.047888047+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:57161] Send NG-Setup response
```

<a id="start_ue"></a>

#### Start UE

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/free5gc-ue.yaml
UERANSIM v3.2.6
[2023-11-10 23:43:26.051] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2023-11-10 23:43:26.051] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2023-11-10 23:43:26.052] [nas] [info] Selected plmn[001/01]
[2023-11-10 23:43:26.052] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2023-11-10 23:43:26.052] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2023-11-10 23:43:26.052] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2023-11-10 23:43:26.052] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2023-11-10 23:43:26.054] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2023-11-10 23:43:26.054] [nas] [debug] Sending Initial Registration
[2023-11-10 23:43:26.054] [rrc] [debug] Sending RRC Setup Request
[2023-11-10 23:43:26.055] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2023-11-10 23:43:26.055] [rrc] [info] RRC connection established
[2023-11-10 23:43:26.055] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2023-11-10 23:43:26.055] [nas] [info] UE switches to state [CM-CONNECTED]
[2023-11-10 23:43:26.074] [nas] [debug] Authentication Request received
[2023-11-10 23:43:26.084] [nas] [debug] Security Mode Command received
[2023-11-10 23:43:26.084] [nas] [debug] Selected integrity[2] ciphering[0]
[2023-11-10 23:43:26.124] [nas] [debug] Registration accept received
[2023-11-10 23:43:26.124] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2023-11-10 23:43:26.124] [nas] [debug] Sending Registration Complete
[2023-11-10 23:43:26.124] [nas] [info] Initial Registration is successful
[2023-11-10 23:43:26.124] [nas] [debug] Sending PDU Session Establishment Request
[2023-11-10 23:43:26.124] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2023-11-10 23:43:26.333] [nas] [debug] Configuration Update Command received
[2023-11-10 23:43:26.437] [nas] [debug] PDU Session Establishment Accept received
[2023-11-10 23:43:26.441] [nas] [info] PDU Session establishment is successful PSI[1]
[2023-11-10 23:43:26.466] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
The free5GC C-Plane log when executed is as follows.
```
2023-11-10T23:43:26.053688221+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:57161] Handle InitialUEMessage
2023-11-10T23:43:26.053735269+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:57161] New RanUe [RanUeNgapID:1][AmfUeNgapID:1]
2023-11-10T23:43:26.053768931+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:57161] 5GSMobileIdentity ["SUCI":"suci-0-001-01-0000-0-0-0000000000", err: <nil>]
2023-11-10T23:43:26.054335025+09:00 [INFO][AMF][CTX] New AmfUe [supi:][guti:00101cafe0000000001]
2023-11-10T23:43:26.054369321+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Deregistered] to [Deregistered]
2023-11-10T23:43:26.054762063+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Registration Request
2023-11-10T23:43:26.054924666+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] RegistrationType: Initial Registration
2023-11-10T23:43:26.055096826+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] MobileIdentity5GS: SUCI[suci-0-001-01-0000-0-0-0000000000]
2023-11-10T23:43:26.055276742+09:00 [INFO][AMF][Gmm] Handle event[Start Authentication], transition from [Deregistered] to [Authentication]
2023-11-10T23:43:26.055435859+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Authentication procedure
2023-11-10T23:43:26.056164880+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.057229031+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=AUSF |
2023-11-10T23:43:26.058695664+09:00 [INFO][AUSF][UeAuth] HandleUeAuthPostRequest
2023-11-10T23:43:26.058888732+09:00 [INFO][AUSF][UeAuth] Serving network authorized
2023-11-10T23:43:26.059599964+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.060967304+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AUSF&service-names=nudm-ueau&target-nf-type=UDM |
2023-11-10T23:43:26.062040370+09:00 [INFO][UDM][UEAU] Handle GenerateAuthDataRequest
2023-11-10T23:43:26.062281776+09:00 [INFO][UDM][Suci] suciPart: [suci 0 001 01 0000 0 0 0000000000]
2023-11-10T23:43:26.062465576+09:00 [INFO][UDM][Suci] scheme 0
2023-11-10T23:43:26.062503457+09:00 [INFO][UDM][Suci] SUPI type is IMSI
http://127.0.0.10:8000
2023-11-10T23:43:26.063053925+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.063896927+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=UDM&target-nf-type=UDR |
2023-11-10T23:43:26.064797029+09:00 [INFO][UDR][DataRepo] Handle QueryAuthSubsData
2023-11-10T23:43:26.066357980+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2023-11-10T23:43:26.067219901+09:00 [INFO][UDM][UEAU] Nil Op
2023-11-10T23:43:26.067654036+09:00 [INFO][UDR][DataRepo] Handle ModifyAuthentication
2023-11-10T23:43:26.069546557+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PATCH   | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2023-11-10T23:43:26.069875338+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | POST    | /nudm-ueau/v1/suci-0-001-01-0000-0-0-0000000000/security-information/generate-auth-data |
2023-11-10T23:43:26.070189461+09:00 [INFO][AUSF][UeAuth] Add SuciSupiPair (suci-0-001-01-0000-0-0-0000000000, imsi-001010000000000) to map.
2023-11-10T23:43:26.070245061+09:00 [INFO][AUSF][UeAuth] Use 5G AKA auth method
2023-11-10T23:43:26.070282527+09:00 [INFO][AUSF][5gAka] XresStar = 6631376535613461363165333362653061653661336563643562363062613235
2023-11-10T23:43:26.070392629+09:00 [INFO][AUSF][GIN] | 201 |       127.0.0.1 | POST    | /nausf-auth/v1/ue-authentications |
2023-11-10T23:43:26.071028081+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Send Authentication Request
2023-11-10T23:43:26.071280620+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:57161] Send Downlink Nas Transport
2023-11-10T23:43:26.072056507+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Start T3560 timer
2023-11-10T23:43:26.073422667+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:57161] Handle UplinkNASTransport
2023-11-10T23:43:26.073773194+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:57161] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2023-11-10T23:43:26.074015181+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Authentication] to [Authentication]
2023-11-10T23:43:26.074172993+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Authentication Response
2023-11-10T23:43:26.074321583+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Stop T3560 timer
2023-11-10T23:43:26.075210410+09:00 [INFO][AUSF][5gAka] Auth5gAkaComfirmRequest
2023-11-10T23:43:26.075557081+09:00 [INFO][AUSF][5gAka] res*: 6631376535613461363165333362653061653661336563643562363062613235
Xres*: 6631376535613461363165333362653061653661336563643562363062613235
2023-11-10T23:43:26.076129310+09:00 [INFO][AUSF][5gAka] 5G AKA confirmation succeeded
2023-11-10T23:43:26.077203337+09:00 [INFO][UDM][UEAU] Handle ConfirmAuthDataRequest
2023-11-10T23:43:26.078023767+09:00 [INFO][UDR][DataRepo] Handle CreateAuthenticationStatus
2023-11-10T23:43:26.079325184+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-status |
2023-11-10T23:43:26.079759352+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-ueau/v1/imsi-001010000000000/auth-events |
2023-11-10T23:43:26.080237359+09:00 [INFO][AUSF][GIN] | 200 |       127.0.0.1 | PUT     | /nausf-auth/v1/ue-authentications/suci-0-001-01-0000-0-0-0000000000/5g-aka-confirmation |
2023-11-10T23:43:26.080630200+09:00 [INFO][AMF][Gmm] Handle event[Authentication Success], transition from [Authentication] to [SecurityMode]
2023-11-10T23:43:26.080840569+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Security Mode Command
2023-11-10T23:43:26.080901969+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:57161] Send Downlink Nas Transport
2023-11-10T23:43:26.081326971+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3560 timer
2023-11-10T23:43:26.082905500+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:57161] Handle UplinkNASTransport
2023-11-10T23:43:26.083164604+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:57161] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2023-11-10T23:43:26.083360126+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [SecurityMode] to [SecurityMode]
2023-11-10T23:43:26.083532686+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Security Mode Complete
2023-11-10T23:43:26.083680068+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3560 timer
2023-11-10T23:43:26.083875577+09:00 [INFO][AMF][Gmm] Handle event[SecurityMode Success], transition from [SecurityMode] to [ContextSetup]
2023-11-10T23:43:26.084031659+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle InitialRegistration
2023-11-10T23:43:26.084788289+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.086083814+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2023-11-10T23:43:26.087204053+09:00 [INFO][UDM][SDM] Handle GetNssai
2023-11-10T23:43:26.087905685+09:00 [INFO][UDR][DataRepo] Handle QueryAmData
2023-11-10T23:43:26.088757762+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data |
2023-11-10T23:43:26.089314761+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/nssai?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-11-10T23:43:26.089812802+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] RequestedNssai - ServingSnssai: &{Sst:1 Sd:010203}, HomeSnssai: <nil>
2023-11-10T23:43:26.091333488+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.094188679+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2023-11-10T23:43:26.095591667+09:00 [INFO][UDM][UECM] Handle RegistrationAmf3gppAccess
2023-11-10T23:43:26.095831064+09:00 [INFO][UDM][UECM] UEID: imsi-001010000000000
2023-11-10T23:43:26.096717477+09:00 [INFO][UDR][DataRepo] Handle CreateAmfContext3gpp
2023-11-10T23:43:26.098043922+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/amf-3gpp-access |
2023-11-10T23:43:26.098371549+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | PUT     | /nudm-uecm/v1/imsi-001010000000000/registrations/amf-3gpp-access |
2023-11-10T23:43:26.099143953+09:00 [INFO][UDM][SDM] Handle GetAmData
2023-11-10T23:43:26.099607570+09:00 [INFO][UDR][DataRepo] Handle QueryAmData
2023-11-10T23:43:26.100254801+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-11-10T23:43:26.100742228+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/am-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-11-10T23:43:26.101548569+09:00 [INFO][UDM][SDM] Handle GetSmfSelectData
2023-11-10T23:43:26.102157219+09:00 [INFO][UDR][DataRepo] Handle QuerySmfSelectData
2023-11-10T23:43:26.102795763+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/smf-selection-subscription-data |
2023-11-10T23:43:26.103164907+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/smf-select-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-11-10T23:43:26.103771703+09:00 [INFO][UDM][SDM] Handle GetUeContextInSmfData
2023-11-10T23:43:26.104156773+09:00 [INFO][UDR][DataRepo] Handle QuerySmfRegList
2023-11-10T23:43:26.104738049+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/smf-registrations |
2023-11-10T23:43:26.105195238+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/ue-context-in-smf-data |
2023-11-10T23:43:26.105995729+09:00 [INFO][UDM][SDM] Handle Subscribe
2023-11-10T23:43:26.106680561+09:00 [INFO][UDR][DataRepo] Handle CreateSdmSubscriptions
2023-11-10T23:43:26.106995925+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |
2023-11-10T23:43:26.107337333+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v1/imsi-001010000000000/sdm-subscriptions |
2023-11-10T23:43:26.108176914+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.109546914+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=PCF |
2023-11-10T23:43:26.111641444+09:00 [INFO][PCF][AmPol] Handle AM Policy Create Request
2023-11-10T23:43:26.112405867+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.113309865+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=UDR |
2023-11-10T23:43:26.114234514+09:00 [INFO][UDR][DataRepo] Handle PolicyDataUesUeIdAmDataGet
2023-11-10T23:43:26.114744675+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/policy-data/ues/imsi-001010000000000/am-data |
2023-11-10T23:43:26.115350116+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.116633768+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?guami=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22amfId%22%3A%22cafe00%22%7D&requester-nf-type=PCF&target-nf-type=AMF |
2023-11-10T23:43:26.117735493+09:00 [INFO][AMF][Comm] Handle AMF Status Change Subscribe Request
2023-11-10T23:43:26.117840098+09:00 [INFO][AMF][Comm] new AMF Status Subscription[1]
2023-11-10T23:43:26.117900282+09:00 [INFO][AMF][GIN] | 201 |       127.0.0.1 | POST    | /namf-comm/v1/subscriptions |
2023-11-10T23:43:26.118271904+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-am-policy-control/v1/policies |
2023-11-10T23:43:26.119122608+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Registration Accept
2023-11-10T23:43:26.119309269+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:57161] Send Initial Context Setup Request
2023-11-10T23:43:26.120650744+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3550 timer
2023-11-10T23:43:26.121465450+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:57161] Handle InitialContextSetupResponse
2023-11-10T23:43:26.121663712+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:57161] Handle InitialContextSetupResponse (RAN UE NGAP ID: 1)
2023-11-10T23:43:26.328225086+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:57161] Handle UplinkNASTransport
2023-11-10T23:43:26.328499353+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:57161] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2023-11-10T23:43:26.328724919+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [ContextSetup] to [ContextSetup]
2023-11-10T23:43:26.328999041+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Registration Complete
2023-11-10T23:43:26.329160682+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3550 timer
2023-11-10T23:43:26.329400340+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Configuration Update Command
2023-11-10T23:43:26.329601914+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:57161] Send Downlink Nas Transport
2023-11-10T23:43:26.330462602+09:00 [INFO][AMF][Gmm] Handle event[ContextSetup Success], transition from [ContextSetup] to [Registered]
2023-11-10T23:43:26.331662270+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:57161] Handle UplinkNASTransport
2023-11-10T23:43:26.331843162+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:57161] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2023-11-10T23:43:26.332056327+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Registered] to [Registered]
2023-11-10T23:43:26.332257357+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle UL NAS Transport
2023-11-10T23:43:26.332431563+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Transport 5GSM Message to SMF
2023-11-10T23:43:26.332657220+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Select SMF [snssai: {Sst:1 Sd:010203}, dnn: internet]
2023-11-10T23:43:26.334367842+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.336358228+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=NSSF |
2023-11-10T23:43:26.337551989+09:00 [INFO][NSSF][NsSel] Handle NSSelectionGet
2023-11-10T23:43:26.338085545+09:00 [INFO][NSSF][GIN] | 200 |       127.0.0.1 | GET     | /nnssf-nsselection/v1/network-slice-information?nf-id=daa13624-628f-4a4d-a77c-5fdd4360e204&nf-type=AMF&slice-info-request-for-pdu-session=%7B%22sNssai%22%3A%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%2C%22roamingIndication%22%3A%22NON_ROAMING%22%7D |
2023-11-10T23:43:26.339157679+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.340500036+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?dnn=internet&preferred-locality=area1&requester-nf-type=AMF&service-names=nsmf-pdusession&snssais=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&target-nf-type=SMF&target-plmn-list=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-11-10T23:43:26.341778063+09:00 [INFO][SMF][PduSess] Receive Create SM Context Request
2023-11-10T23:43:26.342618611+09:00 [INFO][SMF][PduSess] In HandlePDUSessionSMContextCreate
2023-11-10T23:43:26.342968504+09:00 [INFO][SMF][CTX] UrrPeriod: 10s
2023-11-10T23:43:26.343180823+09:00 [INFO][SMF][CTX] UrrThreshold: 1000
2023-11-10T23:43:26.343970705+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.345505909+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=UDM |
2023-11-10T23:43:26.347106821+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Send NF Discovery Serving UDM Successfully
2023-11-10T23:43:26.347857483+09:00 [INFO][UDM][SDM] Handle GetSmData
2023-11-10T23:43:26.347994213+09:00 [INFO][UDM][SDM] getSmDataProcedure: SUPI[imsi-001010000000000] PLMNID[00101] DNN[internet] SNssai[{"sst":1,"sd":"010203"}]
2023-11-10T23:43:26.348491001+09:00 [INFO][UDR][DataRepo] Handle QuerySmData
2023-11-10T23:43:26.349408741+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/sm-data?single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2023-11-10T23:43:26.349963205+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/sm-data?dnn=internet&plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D&single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2023-11-10T23:43:26.350661849+09:00 [INFO][SMF][GSM] In HandlePDUSessionEstablishmentRequest
2023-11-10T23:43:26+09:00 [INFO][NAS][Convert] ProtocolOrContainerList:  [0xc000308ac0 0xc000308ae0]
2023-11-10T23:43:26.351298289+09:00 [INFO][SMF][GSM] Protocol Configuration Options
2023-11-10T23:43:26.351536916+09:00 [INFO][SMF][GSM] &{[0xc000308ac0 0xc000308ae0]}
2023-11-10T23:43:26.351695808+09:00 [INFO][SMF][GSM] Didn't Implement container type IPAddressAllocationViaNASSignallingUL
2023-11-10T23:43:26.353090595+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.354382867+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-instance-id=daa13624-628f-4a4d-a77c-5fdd4360e204&target-nf-type=AMF |
2023-11-10T23:43:26.354859192+09:00 [INFO][SMF][Consumer] SendNFDiscoveryServingAMF ok
2023-11-10T23:43:26.355437780+09:00 [INFO][SMF][CTX] Allocated UE IP address: 10.60.0.1
2023-11-10T23:43:26.355650586+09:00 [INFO][SMF][CTX] Selected UPF: UPF
2023-11-10T23:43:26.355861519+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Allocated PDUAdress[10.60.0.1]
2023-11-10T23:43:26.356637331+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.357672280+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=SMF&target-nf-type=PCF |
2023-11-10T23:43:26.359373441+09:00 [INFO][PCF][SMpolicy] Handle CreateSmPolicy
2023-11-10T23:43:26.361023013+09:00 [INFO][UDR][DataRepo] Handle PolicyDataUesUeIdSmDataGet
2023-11-10T23:43:26.362281008+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/policy-data/ues/imsi-001010000000000/sm-data?dnn=internet&snssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2023-11-10T23:43:26.365153840+09:00 [INFO][UDR][DataRepo] Handle ApplicationDataInfluenceDataGet
2023-11-10T23:43:26.365529843+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/application-data/influenceData?dnns=internet&internal-Group-Ids=&snssais=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&supis=imsi-001010000000000 |
2023-11-10T23:43:26.366005198+09:00 [INFO][PCF][SMpolicy] Matched [0] trafficInfluDatas from UDR
2023-11-10T23:43:26.366551347+09:00 [INFO][UDR][DataRepo] Handle ApplicationDataInfluenceDataSubsToNotifyPost
2023-11-10T23:43:26.366626381+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v1/application-data/influenceData/subs-to-notify |
2023-11-10T23:43:26.367521955+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-11-10T23:43:26.368313630+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=BSF |
2023-11-10T23:43:26.369774939+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-smpolicycontrol/v1/sm-policies |
2023-11-10T23:43:26.371491150+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Has no pre-config route
2023-11-10T23:43:26.371743665+09:00 [WARN][SMF][PduSess] Create URR
2023-11-10T23:43:26.374990407+09:00 [WARN][SMF][PduSess] Create URR
2023-11-10T23:43:26.375196008+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Install PCCRule[PccRuleId-1]
2023-11-10T23:43:26.375392729+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] No srcTcData and tgtTcData. Nothing to do
2023-11-10T23:43:26.375709613+09:00 [INFO][SMF][GIN] | 201 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts |
2023-11-10T23:43:26.376537975+09:00 [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2023-11-10T23:43:26.377352092+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] create smContext[pduSessionID: 1] Success
2023-11-10T23:43:26.384782254+09:00 [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2023-11-10T23:43:26.387117958+09:00 [INFO][AMF][Producer] Handle N1N2 Message Transfer Request
2023-11-10T23:43:26.387358395+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:57161] Send PDU Session Resource Setup Request
2023-11-10T23:43:26.388245770+09:00 [INFO][AMF][GIN] | 200 |       127.0.0.1 | POST    | /namf-comm/v1/ue-contexts/imsi-001010000000000/n1-n2-messages |
2023-11-10T23:43:26.434339471+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:57161] Handle PDUSessionResourceSetupResponse
2023-11-10T23:43:26.434640362+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:57161] Handle PDUSessionResourceSetupResponse (RAN UE NGAP ID: 1)
2023-11-10T23:43:26.435553570+09:00 [INFO][SMF][PduSess] Receive Update SM Context Request
2023-11-10T23:43:26.446396287+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Accepted Response from AN UPF
2023-11-10T23:43:26.447026625+09:00 [INFO][SMF][GIN] | 200 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts/urn:uuid:29a06044-9258-416c-a1b0-da5d0497844e/modify |
```
The PDU session establishment status of VPP-UPF is as follows.
```
vpp# show upf session 
CP F-SEID: 0x0000000000000001 (1) @ 192.168.14.141
UP F-SEID: 0x0000000000000001 (1) @ 192.168.14.151
  PFCP Association: 0
  TEID assignment per choose ID
PDR: 1 @ 0x7f56710e3948
  Precedence: 255
  PDI:
    Fields: 0000000d
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 1 (0x00000001)
            IPv4: 192.168.13.151
    UE IP address (source):
      IPv4 address: 10.60.0.1
    SDF Filter [1]:
      permit out ip from any to assigned 
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 1
  URR Ids: [1,2] @ 0x7f56710f0b98
  QER Ids: [2,1] @ 0x7f56710d5e58
PDR: 2 @ 0x7f56710e39c8
  Precedence: 255
  PDI:
    Fields: 0000000c
    Source Interface: SGi-LAN
    Network Instance: internet
    UE IP address (destination):
      IPv4 address: 10.60.0.1
    SDF Filter [1]:
      permit out ip from any to assigned 
  Outer Header Removal: no
  FAR Id: 2
  URR Ids: [1,2] @ 0x7f56710d62d8
  QER Ids: [2,1] @ 0x7f56710f4d58
PDR: 3 @ 0x7f56710e3a48
  Precedence: 128
  PDI:
    Fields: 0000000d
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 1 (0x00000001)
            IPv4: 192.168.13.151
    UE IP address (source):
      IPv4 address: 10.60.0.1
    SDF Filter [1]:
      permit out ip from 10.60.0.0/16 to any 
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 3
  URR Ids: [1,2] @ 0x7f5671fdb618
  QER Ids: [1,3] @ 0x7f56710f48d8
PDR: 4 @ 0x7f56710e3ac8
  Precedence: 128
  PDI:
    Fields: 0000000c
    Source Interface: SGi-LAN
    Network Instance: internet
    UE IP address (destination):
      IPv4 address: 10.60.0.1
    SDF Filter [1]:
      permit out ip from any to 10.60.0.0/16 
  Outer Header Removal: no
  FAR Id: 4
  URR Ids: [1,2] @ 0x7f56710bc318
  QER Ids: [1,3] @ 0x7f56710f1fd8
FAR: 1
  Apply Action: 00000002 == [FORWARD]
  Forward:
    Network Instance: internet
    Destination Interface: 2
FAR: 2
  Apply Action: 00000002 == [FORWARD]
  Forward:
    Network Instance: internet
    Destination Interface: 0
    Outer Header Creation: [GTP-U/UDP/IPv4],TEID:00000001,IP:192.168.13.131
FAR: 3
  Apply Action: 00000002 == [FORWARD]
  Forward:
    Network Instance: internet
    Destination Interface: 2
FAR: 4
  Apply Action: 00000002 == [FORWARD]
  Forward:
    Network Instance: internet
    Destination Interface: 0
    Outer Header Creation: [GTP-U/UDP/IPv4],TEID:00000001,IP:192.168.13.131
URR: 1
  Measurement Method: 0002 == [VOLUME]
  Reporting Triggers: 0003 == [PERIODIC REPORTING,VOLUME THRESHOLD]
  Status: 0 == []
  Start Time: 2023/11/10 23:44:56:580
  vTime of First Usage:       0.0000 
  vTime of Last Usage:        0.0000 
  Volume
    Up:    Measured:                    0, Theshold:                 1000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Down:  Measured:                    0, Theshold:                 1000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Total: Measured:                    0, Theshold:                    0, Pkts:          0
           Consumed:                    0, Quota:                       0
  Measurement Period:                   10 secs @ 2023/11/10 23:45:06:580, in     4.181 secs, handle 0x0000003d
URR: 2
  Measurement Method: 0002 == [VOLUME]
  Reporting Triggers: 0003 == [PERIODIC REPORTING,VOLUME THRESHOLD]
  Status: 0 == []
  Start Time: 2023/11/10 23:44:56:580
  vTime of First Usage:       0.0000 
  vTime of Last Usage:        0.0000 
  Volume
    Up:    Measured:                    0, Theshold:                 1000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Down:  Measured:                    0, Theshold:                 1000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Total: Measured:                    0, Theshold:                    0, Pkts:          0
           Consumed:                    0, Quota:                       0
  Measurement Period:                   10 secs @ 2023/11/10 23:45:06:580, in     4.181 secs, handle 0x0000003e
vpp# 
```
Looking at the console log of the `nr-ue` command, UE has been assigned the IP address `10.60.0.1` from free5GC 5GC.
```
[2023-11-10 23:43:26.466] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
7: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.60.0.1/32 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::c804:3e08:b8f0:ba20/64 scope link stable-privacy 
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

Run `tcpdump` on VM-DN and check that the packet goes through N6 (enp0s9).
- `ping google.com` on VM3 (UE)
```
# ping google.com -I uesimtun0 -n
PING google.com (172.217.26.238) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 172.217.26.238: icmp_seq=2 ttl=59 time=20.4 ms
64 bytes from 172.217.26.238: icmp_seq=3 ttl=59 time=24.6 ms
64 bytes from 172.217.26.238: icmp_seq=4 ttl=59 time=19.9 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i enp0s9 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), snapshot length 262144 bytes
23:47:05.860786 IP 10.60.0.1 > 172.217.26.238: ICMP echo request, id 5, seq 2, length 64
23:47:05.880448 IP 172.217.26.238 > 10.60.0.1: ICMP echo reply, id 5, seq 2, length 64
23:47:06.861754 IP 10.60.0.1 > 172.217.26.238: ICMP echo request, id 5, seq 3, length 64
23:47:06.885482 IP 172.217.26.238 > 10.60.0.1: ICMP echo reply, id 5, seq 3, length 64
23:47:07.862713 IP 10.60.0.1 > 172.217.26.238: ICMP echo request, id 5, seq 4, length 64
23:47:07.881436 IP 172.217.26.238 > 10.60.0.1: ICMP echo reply, id 5, seq 4, length 64
```

You could specify the IP address assigned to the TUNnel interface to run almost any applications as in the following example using `nr-binder` tool.

- Run `curl google.com` on VM3 (UE)
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
23:48:06.978092 IP 10.60.0.1.48437 > 172.217.26.238.80: Flags [S], seq 3147857, win 65280, options [mss 1360,sackOK,TS val 1398062012 ecr 0,nop,wscale 7], length 0
23:48:07.015043 IP 172.217.26.238.80 > 10.60.0.1.48437: Flags [S.], seq 2880001, ack 3147858, win 65535, options [mss 1460], length 0
23:48:07.015748 IP 10.60.0.1.48437 > 172.217.26.238.80: Flags [.], ack 1, win 65280, length 0
23:48:07.015857 IP 10.60.0.1.48437 > 172.217.26.238.80: Flags [P.], seq 1:75, ack 1, win 65280, length 74: HTTP: GET / HTTP/1.1
23:48:07.015943 IP 172.217.26.238.80 > 10.60.0.1.48437: Flags [.], ack 75, win 65535, length 0
23:48:07.121898 IP 172.217.26.238.80 > 10.60.0.1.48437: Flags [P.], seq 1:774, ack 75, win 65535, length 773: HTTP: HTTP/1.1 301 Moved Permanently
23:48:07.122764 IP 10.60.0.1.48437 > 172.217.26.238.80: Flags [.], ack 774, win 64507, length 0
23:48:07.124672 IP 10.60.0.1.48437 > 172.217.26.238.80: Flags [F.], seq 75, ack 774, win 64507, length 0
23:48:07.124753 IP 172.217.26.238.80 > 10.60.0.1.48437: Flags [.], ack 76, win 65535, length 0
23:48:07.169834 IP 172.217.26.238.80 > 10.60.0.1.48437: Flags [F.], seq 774, ack 76, win 65535, length 0
23:48:07.170455 IP 10.60.0.1.48437 > 172.217.26.238.80: Flags [.], ack 775, win 64507, length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using VPP-UPF with DPDK.

---

Now you could work free5GC with VPP-UPF.
I would like to thank the excellent developers and all the contributors of free5GC, OpenAir CN 5G for UPF, UPG-VPP, DPDK and UERANSIM.

<a id="changelog"></a>

## Changelog (summary)

- [2023.11.10] Changed VPP-UPF from [oai-cn5g-upf-vpp](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp) to its base [travelping/upg-vpp](https://github.com/travelping/upg-vpp).
- [2023.07.15] Enabled URR in `smfcfg.yaml`.
- [2023.06.27] The SMF fixed on 2023.06.27 now works with oai-cn5g-upf-vpp v1.5.1, so I updated to free5GC v3.3.0.
- [2023.06.15] Initial release.
