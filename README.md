# free5GC 5GC & UERANSIM UE / RAN Sample Configuration - VPP-UPF with DPDK
This describes a simple configuration for working free5GC and VPP-UPF with DPDK.
In particular, see [here](https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1) for VPP-UPF with DPDK configuration.

**If UPG-VPP built with [this instruction](https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1) does not work well, please try OAI-CN5G-UPF-VPP built with [this instruction](https://github.com/s5uishida/install_vpp_upf_dpdk#build).**

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
- 5GC - free5GC v3.4.0 (2024.02.17) - https://github.com/free5gc/free5gc
- VPP-UPF - UPG-VPP v1.12.0 (2024.01.25) - https://github.com/travelping/upg-vpp
- UE / RAN - UERANSIM v3.2.6 (2024.03.08) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Memory<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | free5GC 5GC C-Plane | 192.168.0.141/24 | Ubuntu 22.04 | 1 | 2GB | 20GB |
| VM-UP | UPG-VPP U-Plane | 192.168.0.151/24 | Ubuntu **20.04** | 2 | 8GB | 20GB |
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
- free5GC v3.4.0 (2024.02.17) - https://free5gc.org/guide/
- UPG-VPP v1.12.0 (2024.01.25) - https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1
- UERANSIM v3.2.6 (2024.03.08) - https://github.com/aligungr/UERANSIM/wiki/Installation

<a id="changes_cp"></a>

### Changes in configuration files of free5GC 5GC C-Plane

The combination of DNN and S-NSSAI parameters can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- S-NSSAI

For the sake of simplicity, This time, only DNN will be changed. S-NSSAI of all UEs is fixed as `SST=1` and `SD=010203`.

- `free5gc/config/amfcfg.yaml`
```diff
--- amfcfg.yaml.orig    2024-03-24 17:44:26.586613478 +0900
+++ amfcfg.yaml 2024-03-24 17:54:43.098963522 +0900
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
--- ausfcfg.yaml.orig   2024-03-24 17:44:26.586613478 +0900
+++ ausfcfg.yaml        2024-03-24 17:54:54.683442170 +0900
@@ -16,8 +16,8 @@
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   nrfCertPem: cert/nrf.pem # NRF Certificate
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
--- nrfcfg.yaml.orig    2024-03-24 17:44:26.586613478 +0900
+++ nrfcfg.yaml 2024-03-24 17:55:10.259073431 +0900
@@ -15,8 +15,8 @@
       key: cert/nrf.key # NRF TLS Private key
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
--- nssfcfg.yaml.orig   2024-03-24 17:44:26.586613478 +0900
+++ nssfcfg.yaml        2024-03-24 17:55:25.660684149 +0900
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
--- smfcfg.yaml.orig    2024-03-24 17:44:26.586613478 +0900
+++ smfcfg.yaml 2024-03-24 17:56:35.702302968 +0900
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
@@ -90,8 +90,10 @@
     maxRetryTimes: 3 # the max number of retransmission
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   nrfCertPem: cert/nrf.pem # NRF Certificate
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
--- free5gc-gnb.yaml.orig       2023-12-02 06:14:20.000000000 +0900
+++ free5gc-gnb.yaml    2024-03-24 16:59:41.400899800 +0900
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
--- free5gc-ue.yaml.orig        2024-03-02 20:20:59.000000000 +0900
+++ free5gc-ue.yaml     2024-03-24 17:02:26.271746257 +0900
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
- free5GC v3.4.0 (2024.02.17) - https://free5gc.org/guide/
- UPG-VPP v1.12.0 (2024.01.25) - https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1
- UERANSIM v3.2.6 (2024.03.08) - https://github.com/aligungr/UERANSIM/wiki/Installation

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
  Recovery Time Stamp: 2024/03/24 18:47:07:000
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
[2024-03-24 18:47:36.925] [sctp] [info] Trying to establish SCTP connection... (192.168.0.141:38412)
[2024-03-24 18:47:36.928] [sctp] [info] SCTP connection established (192.168.0.141:38412)
[2024-03-24 18:47:36.929] [sctp] [debug] SCTP association setup ascId[11]
[2024-03-24 18:47:36.929] [ngap] [debug] Sending NG Setup Request
[2024-03-24 18:47:36.931] [ngap] [debug] NG Setup Response received
[2024-03-24 18:47:36.931] [ngap] [info] NG Setup procedure is successful
```
The free5GC C-Plane log when executed is as follows.
```
2024-03-24T18:47:36.947645674+09:00 [INFO][AMF][Ngap] [AMF] SCTP Accept from: 192.168.0.131:38737
2024-03-24T18:47:36.948228708+09:00 [INFO][AMF][Ngap] Create a new NG connection for: 192.168.0.131:38737
2024-03-24T18:47:36.948943407+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:38737] Handle NGSetupRequest
2024-03-24T18:47:36.948975576+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:38737] Send NG-Setup response
```

<a id="start_ue"></a>

#### Start UE

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/free5gc-ue.yaml
UERANSIM v3.2.6
[2024-03-24 18:48:05.181] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2024-03-24 18:48:05.181] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2024-03-24 18:48:05.182] [nas] [info] Selected plmn[001/01]
[2024-03-24 18:48:05.182] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2024-03-24 18:48:05.182] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2024-03-24 18:48:05.182] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2024-03-24 18:48:05.182] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2024-03-24 18:48:05.182] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2024-03-24 18:48:05.183] [nas] [debug] Sending Initial Registration
[2024-03-24 18:48:05.183] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2024-03-24 18:48:05.183] [rrc] [debug] Sending RRC Setup Request
[2024-03-24 18:48:05.183] [rrc] [info] RRC connection established
[2024-03-24 18:48:05.184] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2024-03-24 18:48:05.184] [nas] [info] UE switches to state [CM-CONNECTED]
[2024-03-24 18:48:05.243] [nas] [debug] Authentication Request received
[2024-03-24 18:48:05.243] [nas] [debug] Received SQN [00000000007E]
[2024-03-24 18:48:05.243] [nas] [debug] SQN-MS [000000000000]
[2024-03-24 18:48:05.269] [nas] [debug] Security Mode Command received
[2024-03-24 18:48:05.270] [nas] [debug] Selected integrity[2] ciphering[0]
[2024-03-24 18:48:05.430] [nas] [debug] Registration accept received
[2024-03-24 18:48:05.430] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2024-03-24 18:48:05.430] [nas] [debug] Sending Registration Complete
[2024-03-24 18:48:05.430] [nas] [info] Initial Registration is successful
[2024-03-24 18:48:05.431] [nas] [debug] Sending PDU Session Establishment Request
[2024-03-24 18:48:05.433] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2024-03-24 18:48:05.634] [nas] [debug] Configuration Update Command received
[2024-03-24 18:48:05.801] [nas] [debug] PDU Session Establishment Accept received
[2024-03-24 18:48:05.806] [nas] [info] PDU Session establishment is successful PSI[1]
[2024-03-24 18:48:05.830] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
The free5GC C-Plane log when executed is as follows.
```
2024-03-24T18:48:05.086207247+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:38737] Handle InitialUEMessage
2024-03-24T18:48:05.086248314+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:38737] New RanUe [RanUeNgapID:1][AmfUeNgapID:1]
2024-03-24T18:48:05.086297575+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:38737] 5GSMobileIdentity ["SUCI":"suci-0-001-01-0000-0-0-0000000000", err: <nil>]
2024-03-24T18:48:05.086781270+09:00 [INFO][AMF][CTX] New AmfUe [supi:][guti:00101cafe0000000001]
2024-03-24T18:48:05.086804483+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Deregistered] to [Deregistered]
2024-03-24T18:48:05.086812370+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Registration Request
2024-03-24T18:48:05.086819665+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] RegistrationType: Initial Registration
2024-03-24T18:48:05.086827570+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] MobileIdentity5GS: SUCI[suci-0-001-01-0000-0-0-0000000000]
2024-03-24T18:48:05.086842507+09:00 [INFO][AMF][Gmm] Handle event[Start Authentication], transition from [Deregistered] to [Authentication]
2024-03-24T18:48:05.086849857+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Authentication procedure
2024-03-24T18:48:05.087718778+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.088026752+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.088186391+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.089576190+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.092225560+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.095756401+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.096702222+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=AUSF |
2024-03-24T18:48:05.097263931+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.097541510+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.097816068+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.099025239+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.102179276+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.103894555+09:00 [INFO][AUSF][UeAuth] HandleUeAuthPostRequest
2024-03-24T18:48:05.104265200+09:00 [INFO][AUSF][UeAuth] Serving network authorized
2024-03-24T18:48:05.104936610+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.105222016+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.105383354+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.106213269+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.108999417+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.110026365+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.111258591+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AUSF&service-names=nudm-ueau&target-nf-type=UDM |
2024-03-24T18:48:05.111784811+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.112049281+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.112247540+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.113045018+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.116646267+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.119573094+09:00 [INFO][UDM][UEAU] Handle GenerateAuthDataRequest
2024-03-24T18:48:05.120767828+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.121194201+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.121557129+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.122666214+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.125779024+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.126227952+09:00 [INFO][UDM][Suci] suciPart: [suci 0 001 01 0000 0 0 0000000000]
2024-03-24T18:48:05.126410090+09:00 [INFO][UDM][Suci] scheme 0
2024-03-24T18:48:05.126619263+09:00 [INFO][UDM][Suci] SUPI type is IMSI
http://127.0.0.10:8000
2024-03-24T18:48:05.127148081+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.127365303+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.127567205+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.128689722+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.131227303+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.132337535+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.133250743+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=UDM&target-nf-type=UDR |
2024-03-24T18:48:05.135045272+09:00 [INFO][UDR][DataRepo] Handle QueryAuthSubsData
2024-03-24T18:48:05.136568453+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2024-03-24T18:48:05.137522360+09:00 [INFO][UDM][UEAU] Nil Op
2024-03-24T18:48:05.138798126+09:00 [INFO][UDR][DataRepo] Handle ModifyAuthentication
2024-03-24T18:48:05.140699874+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PATCH   | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2024-03-24T18:48:05.141242065+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | POST    | /nudm-ueau/v1/suci-0-001-01-0000-0-0-0000000000/security-information/generate-auth-data |
2024-03-24T18:48:05.141744321+09:00 [INFO][AUSF][UeAuth] Add SuciSupiPair (suci-0-001-01-0000-0-0-0000000000, imsi-001010000000000) to map.
2024-03-24T18:48:05.142061868+09:00 [INFO][AUSF][UeAuth] Use 5G AKA auth method
2024-03-24T18:48:05.142233495+09:00 [INFO][AUSF][5gAka] XresStar = 3465393163316633663964366438376631326232373935643638343334383433
2024-03-24T18:48:05.142495730+09:00 [INFO][AUSF][GIN] | 201 |       127.0.0.1 | POST    | /nausf-auth/v1/ue-authentications |
2024-03-24T18:48:05.143011110+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Send Authentication Request
2024-03-24T18:48:05.143228599+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:38737] Send Downlink Nas Transport
2024-03-24T18:48:05.143803959+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Start T3560 timer
2024-03-24T18:48:05.145504013+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:38737] Handle UplinkNASTransport
2024-03-24T18:48:05.145690271+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:38737] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2024-03-24T18:48:05.145901778+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Authentication] to [Authentication]
2024-03-24T18:48:05.145993996+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Authentication Response
2024-03-24T18:48:05.146243355+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Stop T3560 timer
2024-03-24T18:48:05.146960935+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.147307432+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.147356831+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.148808851+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.151864283+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.153350341+09:00 [INFO][AUSF][5gAka] Auth5gAkaComfirmRequest
2024-03-24T18:48:05.153529334+09:00 [INFO][AUSF][5gAka] res*: 3465393163316633663964366438376631326232373935643638343334383433
Xres*: 3465393163316633663964366438376631326232373935643638343334383433
2024-03-24T18:48:05.154041604+09:00 [INFO][AUSF][5gAka] 5G AKA confirmation succeeded
2024-03-24T18:48:05.154656153+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.154790222+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.154815696+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.155633210+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.159187662+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.160779082+09:00 [INFO][UDM][UEAU] Handle ConfirmAuthDataRequest
2024-03-24T18:48:05.161492614+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.161761313+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.161971286+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.163066463+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.166128596+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.167605013+09:00 [INFO][UDR][DataRepo] Handle CreateAuthenticationStatus
2024-03-24T18:48:05.168696368+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-status |
2024-03-24T18:48:05.169135117+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-ueau/v1/imsi-001010000000000/auth-events |
2024-03-24T18:48:05.169521559+09:00 [INFO][AUSF][GIN] | 200 |       127.0.0.1 | PUT     | /nausf-auth/v1/ue-authentications/suci-0-001-01-0000-0-0-0000000000/5g-aka-confirmation |
2024-03-24T18:48:05.169776011+09:00 [INFO][AMF][Gmm] Handle event[Authentication Success], transition from [Authentication] to [SecurityMode]
2024-03-24T18:48:05.169818522+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Security Mode Command
2024-03-24T18:48:05.170083591+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:38737] Send Downlink Nas Transport
2024-03-24T18:48:05.170578582+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3560 timer
2024-03-24T18:48:05.172265206+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:38737] Handle UplinkNASTransport
2024-03-24T18:48:05.172469668+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:38737] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2024-03-24T18:48:05.172665498+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [SecurityMode] to [SecurityMode]
2024-03-24T18:48:05.172819858+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Security Mode Complete
2024-03-24T18:48:05.172987149+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3560 timer
2024-03-24T18:48:05.173175968+09:00 [INFO][AMF][Gmm] Handle event[SecurityMode Success], transition from [SecurityMode] to [ContextSetup]
2024-03-24T18:48:05.173325076+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle InitialRegistration
2024-03-24T18:48:05.173942250+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.174405641+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.174468761+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.175601729+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.178178773+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.179222908+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.180376432+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2024-03-24T18:48:05.180986285+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.181185549+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.181338488+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.182468979+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.185783770+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.187092255+09:00 [INFO][UDM][SDM] Handle GetNssai
2024-03-24T18:48:05.187991554+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.188301932+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.188458428+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.189519034+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.192604927+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.193976556+09:00 [INFO][UDR][DataRepo] Handle QueryAmData
2024-03-24T18:48:05.195108722+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data |
2024-03-24T18:48:05.195498672+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/nssai?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2024-03-24T18:48:05.195959556+09:00 [INFO][AMF][Gmm] RequestedNssai: &{Iei:47 Len:5 Buffer:[4 1 1 2 3]}
2024-03-24T18:48:05.196122893+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] RequestedNssai - ServingSnssai: &{Sst:1 Sd:010203}, HomeSnssai: <nil>
2024-03-24T18:48:05.196700474+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.197057046+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.197513450+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.199144587+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.201622910+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.202775530+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.204132097+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2024-03-24T18:48:05.204742548+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.204968061+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.205122375+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.208279565+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.212082520+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.213663555+09:00 [INFO][UDM][UECM] Handle RegistrationAmf3gppAccess
2024-03-24T18:48:05.213988237+09:00 [INFO][UDM][UECM] UEID: imsi-001010000000000
2024-03-24T18:48:05.214697908+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.214963659+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.215119754+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.216127914+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.219218228+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.220659137+09:00 [INFO][UDR][DataRepo] Handle CreateAmfContext3gpp
2024-03-24T18:48:05.221865089+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/amf-3gpp-access |
2024-03-24T18:48:05.222194025+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | PUT     | /nudm-uecm/v1/imsi-001010000000000/registrations/amf-3gpp-access |
2024-03-24T18:48:05.223314724+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.223824514+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.224016666+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.225141058+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.228547835+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.230066582+09:00 [INFO][UDM][SDM] Handle GetAmData
2024-03-24T18:48:05.231161295+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.231405779+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.231562610+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.232711801+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.237612007+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.239088321+09:00 [INFO][UDR][DataRepo] Handle QueryAmData
2024-03-24T18:48:05.239838721+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2024-03-24T18:48:05.240398961+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/am-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2024-03-24T18:48:05.241569372+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.241873316+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.242088169+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.243417372+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.246708116+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.248109892+09:00 [INFO][UDM][SDM] Handle GetSmfSelectData
2024-03-24T18:48:05.248789688+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.248998623+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.249098689+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.250168507+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.253276387+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.254554082+09:00 [INFO][UDR][DataRepo] Handle QuerySmfSelectData
2024-03-24T18:48:05.255396575+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/smf-selection-subscription-data |
2024-03-24T18:48:05.255736688+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/smf-select-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2024-03-24T18:48:05.256651643+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.256883384+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.257035562+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.258318830+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.261723409+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.262883204+09:00 [INFO][UDM][SDM] Handle GetUeContextInSmfData
2024-03-24T18:48:05.263871383+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.264145530+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.264296148+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.265335370+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.268446040+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.269751112+09:00 [INFO][UDR][DataRepo] Handle QuerySmfRegList
2024-03-24T18:48:05.270554415+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/smf-registrations |
2024-03-24T18:48:05.270978444+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/ue-context-in-smf-data |
2024-03-24T18:48:05.273114426+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.273364251+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.273532990+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.274880446+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.278910179+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.280767072+09:00 [INFO][UDM][SDM] Handle Subscribe
2024-03-24T18:48:05.281690798+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.284019399+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.284184036+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.285299275+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.288484173+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.289933762+09:00 [INFO][UDR][DataRepo] Handle CreateSdmSubscriptions
2024-03-24T18:48:05.290341035+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |
2024-03-24T18:48:05.290848861+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v1/imsi-001010000000000/sdm-subscriptions |
2024-03-24T18:48:05.291645000+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.291957971+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.292124898+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.293538592+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.296145940+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.297115620+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.298402883+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=PCF |
2024-03-24T18:48:05.299098740+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.299346853+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.299557379+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.300821064+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.304143738+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.306575047+09:00 [INFO][PCF][AmPol] Handle AM Policy Create Request
2024-03-24T18:48:05.307224460+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.307497244+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.307778712+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.309058153+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.311580545+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.312796667+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.313546761+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=UDR |
2024-03-24T18:48:05.314218920+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.314438478+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.314587388+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.315743168+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.318716811+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.320021777+09:00 [INFO][UDR][DataRepo] Handle PolicyDataUesUeIdAmDataGet
2024-03-24T18:48:05.320789402+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/policy-data/ues/imsi-001010000000000/am-data |
2024-03-24T18:48:05.321800101+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.322062999+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.322419570+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.323668171+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.326145974+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.327130044+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.327821423+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?guami=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22amfId%22%3A%22cafe00%22%7D&requester-nf-type=PCF&target-nf-type=AMF |
2024-03-24T18:48:05.328345223+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-am-policy-control/v1/policies |
2024-03-24T18:48:05.328837971+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Registration Accept
2024-03-24T18:48:05.329146239+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:38737] Send Initial Context Setup Request
2024-03-24T18:48:05.330410348+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3550 timer
2024-03-24T18:48:05.331291792+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:38737] Handle InitialContextSetupResponse
2024-03-24T18:48:05.331459036+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:38737] Handle InitialContextSetupResponse (RAN UE NGAP ID: 1)
2024-03-24T18:48:05.533558015+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:38737] Handle UplinkNASTransport
2024-03-24T18:48:05.533839076+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:38737] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2024-03-24T18:48:05.534036289+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [ContextSetup] to [ContextSetup]
2024-03-24T18:48:05.534189101+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Registration Complete
2024-03-24T18:48:05.534360801+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3550 timer
2024-03-24T18:48:05.534564048+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Configuration Update Command
2024-03-24T18:48:05.534723882+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:38737] Send Downlink Nas Transport
2024-03-24T18:48:05.535431707+09:00 [INFO][AMF][Gmm] Handle event[ContextSetup Success], transition from [ContextSetup] to [Registered]
2024-03-24T18:48:05.536225549+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:38737] Handle UplinkNASTransport
2024-03-24T18:48:05.536391486+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:38737] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2024-03-24T18:48:05.536563977+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Registered] to [Registered]
2024-03-24T18:48:05.536707000+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle UL NAS Transport
2024-03-24T18:48:05.536910353+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Transport 5GSM Message to SMF
2024-03-24T18:48:05.537057934+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Select SMF [snssai: {Sst:1 Sd:010203}, dnn: internet]
2024-03-24T18:48:05.537822147+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.538228633+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.538395043+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.539830288+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.542422413+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.543424334+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.544354748+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=NSSF |
2024-03-24T18:48:05.544907748+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.545138663+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.545297114+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.546472930+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.549620408+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.552332331+09:00 [INFO][NSSF][NsSel] Handle NSSelectionGet
2024-03-24T18:48:05.553515571+09:00 [INFO][NSSF][GIN] | 200 |       127.0.0.1 | GET     | /nnssf-nsselection/v1/network-slice-information?nf-id=22454f78-07b4-4c9c-9ce7-14944a5232d4&nf-type=AMF&slice-info-request-for-pdu-session=%7B%22sNssai%22%3A%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%2C%22roamingIndication%22%3A%22NON_ROAMING%22%7D |
2024-03-24T18:48:05.554684414+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.554992477+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.555155695+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.556427149+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.558962963+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.559956908+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.561111917+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?dnn=internet&preferred-locality=area1&requester-nf-type=AMF&service-names=nsmf-pdusession&snssais=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&target-nf-type=SMF&target-plmn-list=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2024-03-24T18:48:05.561671779+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.561906947+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.562072740+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.563291311+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.566444448+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.568408644+09:00 [INFO][SMF][PduSess] Receive Create SM Context Request
2024-03-24T18:48:05.569023814+09:00 [INFO][SMF][PduSess] In HandlePDUSessionSMContextCreate
2024-03-24T18:48:05.569338671+09:00 [INFO][SMF][CTX] UrrPeriod: 10s
2024-03-24T18:48:05.569492237+09:00 [INFO][SMF][CTX] UrrThreshold: 1000
2024-03-24T18:48:05.570167207+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.570425286+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.570576258+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.571553198+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.574345805+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.575422168+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.576512051+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=UDM |
2024-03-24T18:48:05.576970898+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Send NF Discovery Serving UDM Successfully
2024-03-24T18:48:05.577336365+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.577541115+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.577728657+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.578643319+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.581950579+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.583219101+09:00 [INFO][UDM][SDM] Handle GetSmData
2024-03-24T18:48:05.584088655+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.584321618+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.584486847+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.585530256+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.588541719+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.588862044+09:00 [INFO][UDM][SDM] getSmDataProcedure: SUPI[imsi-001010000000000] PLMNID[00101] DNN[internet] SNssai[{"sst":1,"sd":"010203"}]
2024-03-24T18:48:05.590091223+09:00 [INFO][UDR][DataRepo] Handle QuerySmData
2024-03-24T18:48:05.591122238+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/sm-data?single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2024-03-24T18:48:05.591553276+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/sm-data?dnn=internet&plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D&single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2024-03-24T18:48:05.592161468+09:00 [INFO][SMF][GSM] In HandlePDUSessionEstablishmentRequest
2024-03-24T18:48:05+09:00 [INFO][NAS][Convert] ProtocolOrContainerList:  [0xc000308da0 0xc000308dc0]
2024-03-24T18:48:05.592379021+09:00 [INFO][SMF][GSM] Protocol Configuration Options
2024-03-24T18:48:05.592411282+09:00 [INFO][SMF][GSM] &{[0xc000308da0 0xc000308dc0]}
2024-03-24T18:48:05.592425182+09:00 [INFO][SMF][GSM] Didn't Implement container type IPAddressAllocationViaNASSignallingUL
2024-03-24T18:48:05.593545566+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.593762016+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.593950424+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.594954328+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.597573527+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.598646822+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.599838420+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-instance-id=22454f78-07b4-4c9c-9ce7-14944a5232d4&target-nf-type=AMF |
2024-03-24T18:48:05.600429442+09:00 [INFO][SMF][Consumer] SendNFDiscoveryServingAMF ok
2024-03-24T18:48:05.600720347+09:00 [INFO][SMF][CTX] Allocated UE IP address: 10.60.0.1
2024-03-24T18:48:05.600889080+09:00 [INFO][SMF][CTX] Selected UPF: UPF
2024-03-24T18:48:05.601031184+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Allocated PDUAdress[10.60.0.1]
2024-03-24T18:48:05.601993713+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.602202811+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.602356928+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.603311370+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.605809625+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.606967539+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.608233268+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=SMF&target-nf-type=PCF |
2024-03-24T18:48:05.608846159+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.609078376+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.609226575+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.610119844+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.613450544+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.615924635+09:00 [INFO][PCF][SMpolicy] Handle CreateSmPolicy
2024-03-24T18:48:05.616639019+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.616938388+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.617104251+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.619968903+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.625139785+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.626492097+09:00 [INFO][UDR][DataRepo] Handle PolicyDataUesUeIdSmDataGet
2024-03-24T18:48:05.628489654+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/policy-data/ues/imsi-001010000000000/sm-data?dnn=internet&snssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2024-03-24T18:48:05.633548074+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.634046287+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.634204751+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.635483011+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.638631367+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.641345823+09:00 [INFO][UDR][DataRepo] Handle ApplicationDataInfluenceDataGet
2024-03-24T18:48:05.642002352+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/application-data/influenceData?dnns=internet&internal-Group-Ids=&snssais=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&supis=imsi-001010000000000 |
2024-03-24T18:48:05.643105323+09:00 [INFO][PCF][SMpolicy] Matched [0] trafficInfluDatas from UDR
2024-03-24T18:48:05.644405194+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.644623479+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.644771049+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.646011865+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.649119071+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.650447552+09:00 [INFO][UDR][DataRepo] Handle ApplicationDataInfluenceDataSubsToNotifyPost
2024-03-24T18:48:05.650759466+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v1/application-data/influenceData/subs-to-notify |
2024-03-24T18:48:05.651706056+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.651959947+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.652121938+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.653420995+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.655873093+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.657057841+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2024-03-24T18:48:05.657639161+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=BSF |
2024-03-24T18:48:05.658516382+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-smpolicycontrol/v1/sm-policies |
2024-03-24T18:48:05.659471743+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Has no pre-config route
2024-03-24T18:48:05.659745675+09:00 [WARN][SMF][PduSess] Create URR
2024-03-24T18:48:05.660048745+09:00 [WARN][SMF][PduSess] Create URR
2024-03-24T18:48:05.660227862+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Install PCCRule[PccRuleId-1]
2024-03-24T18:48:05.660432670+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] No srcTcData and tgtTcData. Nothing to do
2024-03-24T18:48:05.660665622+09:00 [INFO][SMF][PduSess] UECM Registration SmfInstanceId: b67ead9b-1a27-46e6-977f-5bffe0941cb0  PduSessionId: 1  SNssai: &{1 010203}  Dnn: internet  PlmnId: &{001 01}
2024-03-24T18:48:05.661357146+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.661612112+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.661723534+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.662743911+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.666184268+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.667538245+09:00 [INFO][UDM][UECM] Handle RegistrationSmfRegistrations
2024-03-24T18:48:05.668485986+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.668719830+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.668909743+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.669885312+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.672942583+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.674569169+09:00 [INFO][UDR][DataRepo] Handle CreateSmfContextNon3gpp
2024-03-24T18:48:05.674806031+09:00 [WARN][UDR][DataRepo] strconv.ParseInt: parsing "": invalid syntax
2024-03-24T18:48:05.675839340+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/smf-registrations/1 |
2024-03-24T18:48:05.676390598+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | PUT     | /nudm-uecm/v1/imsi-001010000000000/registrations/smf-registrations/1 |
2024-03-24T18:48:05.676821244+09:00 [INFO][SMF][GIN] | 201 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts |
2024-03-24T18:48:05.676933438+09:00 [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2024-03-24T18:48:05.677378383+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] create smContext[pduSessionID: 1] Success
2024-03-24T18:48:05.689217484+09:00 [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2024-03-24T18:48:05.690883918+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.691096981+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.691292277+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.692405876+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.695856885+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.697879768+09:00 [INFO][AMF][Producer] Handle N1N2 Message Transfer Request
2024-03-24T18:48:05.698172053+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:38737] Send PDU Session Resource Setup Request
2024-03-24T18:48:05.701126432+09:00 [INFO][AMF][GIN] | 200 |       127.0.0.1 | POST    | /namf-comm/v1/ue-contexts/imsi-001010000000000/n1-n2-messages |
2024-03-24T18:48:05.702847487+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:38737] Handle PDUSessionResourceSetupResponse
2024-03-24T18:48:05.703257710+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:38737] Handle PDUSessionResourceSetupResponse (RAN UE NGAP ID: 1)
2024-03-24T18:48:05.704084144+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2024-03-24T18:48:05.704496276+09:00 [INFO][NRF][Token] Handle AccessTokenRequest
2024-03-24T18:48:05.704655745+09:00 [INFO][NRF][Token] In AccessTokenProcedure
2024-03-24T18:48:05.706081007+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2024-03-24T18:48:05.709451151+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |
2024-03-24T18:48:05.711239336+09:00 [INFO][SMF][PduSess] Receive Update SM Context Request
2024-03-24T18:48:05.721223617+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Accepted Response from AN UPF
2024-03-24T18:48:05.721526682+09:00 [INFO][SMF][GIN] | 200 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts/urn:uuid:78c3395d-4132-4820-92b0-4a99b4f35b7b/modify |
```
The PDU session establishment status of VPP-UPF is as follows.
```
vpp# show upf session 
CP F-SEID: 0x0000000000000001 (1) @ 192.168.14.141
UP F-SEID: 0x0000000000000001 (1) @ 192.168.14.151 (192.168.14.141 ::)
  PFCP Association: 0
  TEID assignment per choose ID
PDR: 1 @ 0x7f6c1453ff08
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
  URR Ids: [1,2] @ 0x7f6c1453d458
  QER Ids: [2,1] @ 0x7f6bbd53c718
PDR: 2 @ 0x7f6c1453ff88
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
  URR Ids: [1,2] @ 0x7f6c1453f818
  QER Ids: [2,1] @ 0x7f6c1452fee8
PDR: 3 @ 0x7f6c14540008
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
  URR Ids: [1,2] @ 0x7f6c14541c08
  QER Ids: [1,3] @ 0x7f6c14536bd8
PDR: 4 @ 0x7f6c14540088
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
  URR Ids: [1,2] @ 0x7f6c1452cc28
  QER Ids: [1,3] @ 0x7f6c145369d8
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
  Start Time: 2024/03/24 18:50:15:687
  vTime of First Usage:       0.0000 
  vTime of Last Usage:        0.0000 
  Volume
    Up:    Measured:                    0, Theshold:                 1000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Down:  Measured:                    0, Theshold:                 1000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Total: Measured:                    0, Theshold:                    0, Pkts:          0
           Consumed:                    0, Quota:                       0
  Measurement Period:                   10 secs @ 2024/03/24 18:50:25:687, in     4.826 secs, handle 0x0000004e
URR: 2
  Measurement Method: 0002 == [VOLUME]
  Reporting Triggers: 0003 == [PERIODIC REPORTING,VOLUME THRESHOLD]
  Status: 0 == []
  Start Time: 2024/03/24 18:50:15:687
  vTime of First Usage:       0.0000 
  vTime of Last Usage:        0.0000 
  Volume
    Up:    Measured:                    0, Theshold:                 1000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Down:  Measured:                    0, Theshold:                 1000, Pkts:          0
           Consumed:                    0, Quota:                       0
    Total: Measured:                    0, Theshold:                    0, Pkts:          0
           Consumed:                    0, Quota:                       0
  Measurement Period:                   10 secs @ 2024/03/24 18:50:25:687, in     4.826 secs, handle 0x0000004f
vpp# 
```
Looking at the console log of the `nr-ue` command, UE has been assigned the IP address `10.60.0.1` from free5GC 5GC.
```
[2024-03-24 18:48:05.830] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
8: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.60.0.1/32 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::e7d1:e3b8:9ec1:ad3e/64 scope link stable-privacy 
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
PING google.com (142.251.42.206) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 142.251.42.206: icmp_seq=1 ttl=59 time=31.5 ms
64 bytes from 142.251.42.206: icmp_seq=2 ttl=59 time=15.9 ms
64 bytes from 142.251.42.206: icmp_seq=3 ttl=59 time=17.1 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i enp0s9 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), snapshot length 262144 bytes
18:53:54.935084 IP 10.60.0.1 > 142.251.42.206: ICMP echo request, id 5, seq 1, length 64
18:53:54.965727 IP 142.251.42.206 > 10.60.0.1: ICMP echo reply, id 5, seq 1, length 64
18:53:55.936412 IP 10.60.0.1 > 142.251.42.206: ICMP echo request, id 5, seq 2, length 64
18:53:55.951517 IP 142.251.42.206 > 10.60.0.1: ICMP echo reply, id 5, seq 2, length 64
18:53:56.937426 IP 10.60.0.1 > 142.251.42.206: ICMP echo request, id 5, seq 3, length 64
18:53:56.953709 IP 142.251.42.206 > 10.60.0.1: ICMP echo reply, id 5, seq 3, length 64
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
18:54:40.340420 IP 10.60.0.1.39289 > 142.251.42.206.80: Flags [S], seq 3300839082, win 65280, options [mss 1360,sackOK,TS val 3834889530 ecr 0,nop,wscale 7], length 0
18:54:40.383228 IP 142.251.42.206.80 > 10.60.0.1.39289: Flags [S.], seq 2688001, ack 3300839083, win 65535, options [mss 1460], length 0
18:54:40.384096 IP 10.60.0.1.39289 > 142.251.42.206.80: Flags [.], ack 1, win 65280, length 0
18:54:40.384193 IP 10.60.0.1.39289 > 142.251.42.206.80: Flags [P.], seq 1:75, ack 1, win 65280, length 74: HTTP: GET / HTTP/1.1
18:54:40.384271 IP 142.251.42.206.80 > 10.60.0.1.39289: Flags [.], ack 75, win 65535, length 0
18:54:40.471049 IP 142.251.42.206.80 > 10.60.0.1.39289: Flags [P.], seq 1:774, ack 75, win 65535, length 773: HTTP: HTTP/1.1 301 Moved Permanently
18:54:40.471787 IP 10.60.0.1.39289 > 142.251.42.206.80: Flags [.], ack 774, win 64507, length 0
18:54:40.473073 IP 10.60.0.1.39289 > 142.251.42.206.80: Flags [F.], seq 75, ack 774, win 64507, length 0
18:54:40.473168 IP 142.251.42.206.80 > 10.60.0.1.39289: Flags [.], ack 76, win 65535, length 0
18:54:40.511547 IP 142.251.42.206.80 > 10.60.0.1.39289: Flags [F.], seq 774, ack 76, win 65535, length 0
18:54:40.512213 IP 10.60.0.1.39289 > 142.251.42.206.80: Flags [.], ack 775, win 64507, length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using VPP-UPF with DPDK.

---

Now you could work free5GC with VPP-UPF.
I would like to thank the excellent developers and all the contributors of free5GC, OpenAir CN 5G for UPF, UPG-VPP, VPP, DPDK and UERANSIM.

<a id="changelog"></a>

## Changelog (summary)

- [2024.03.24] Updated to UPG-VPP v1.12.0.
- [2023.11.10] Changed VPP-UPF from [oai-cn5g-upf-vpp](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp) to its base [travelping/upg-vpp](https://github.com/travelping/upg-vpp).
- [2023.07.15] Enabled URR in `smfcfg.yaml`.
- [2023.06.27] The SMF fixed on 2023.06.27 now works with oai-cn5g-upf-vpp v1.5.1, so I updated to free5GC v3.3.0.
- [2023.06.15] Initial release.
