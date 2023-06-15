# free5GC 5GC & UERANSIM UE / RAN Sample Configuration - VPP-UPF with DPDK
This describes a simple configuration for working free5GC and VPP-UPF with DPDK.
In particular, see [here](https://github.com/s5uishida/install_vpp_upf_dpdk) for VPP-UPF with DPDK configuration.

---

<h2 id="conf_list">List of Sample Configurations</h2>

1. [One SMF, Multiple UPFs and DNNs](https://github.com/s5uishida/free5gc_ueransim_sample_config)
2. [Select nearby UPF according to the connected gNodeB](https://github.com/s5uishida/free5gc_ueransim_nearby_upf_sample_config)
3. [Select UPF based on S-NSSAI](https://github.com/s5uishida/free5gc_ueransim_snssai_upf_sample_config)
4. [ULCL(Uplink Classifier)](https://github.com/s5uishida/free5gc_ueransim_ulcl_sample_config)
5. [ULCL with one I-UPF and two PSA-UPFs](https://github.com/s5uishida/free5gc_ueransim_ulcl_2_sample_config)
6. VPP-UPF with DPDK (this article)

---

<h2 id="misc">Miscellaneous Notes</h2>

- [Install MongoDB 6.0 and free5GC WebUI](https://github.com/s5uishida/free5gc_install_mongodb6_webui)
- [A Note for 5G SUCI Profile A/B Scheme](https://github.com/s5uishida/note_5g_suci_profile_ab)
- [Install VPP-UPF with DPDK on Host](https://github.com/s5uishida/install_vpp_upf_dpdk)

---

<h2 id="toc">Table of Contents</h2>

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

<h2 id="overview">Overview of free5GC 5GC Simulation Mobile Network</h2>

This describes a simple configuration of C-Plane, VPP-UPF and Data Network Gateway for free5GC.
**Note that this configuration is implemented with Virtualbox VMs.**

The following minimum configuration was set as a condition.
- One UPF and Data Network Gateway
- One UE and one DNN

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The 5GC / VPP-UPF / UE / RAN used are as follows.
- 5GC - free5GC v3.2.1 (2023.02.12) - https://github.com/free5gc/free5gc
- VPP-UPF - OpenAir CN 5G for UPF v1.5.1 (2023.06.14) - https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp
- UE / RAN - UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | Memory (Min) | HDD (Min) |
| --- | --- | --- | --- | --- | --- |
| VM1 | free5GC 5GC C-Plane | 192.168.0.141/24 | Ubuntu 22.04 | 2GB | 20GB |
| VM-UP | OpenAir CN 5G for UPF | 192.168.0.151/24 | Ubuntu 22.04 | 8GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 22.04 | 1GB | 10GB |
| VM2 | UERANSIM RAN (gNodeB) | 192.168.0.131/24 | Ubuntu 22.04 | 1GB | 10GB |
| VM3 | UERANSIM UE | 192.168.0.132/24 | Ubuntu 22.04 | 1GB | 10GB |

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

<h2 id="changes">Changes in configuration files of free5GC 5GC, VPP-UPF and UERANSIM UE / RAN</h2>

Please refer to the following for building free5GC, VPP-UPF and UERANSIM respectively.
- free5GC v3.2.1 (2023.02.12) - https://github.com/free5gc/free5gc/wiki/Installation
- OpenAir CN 5G for UPF v1.5.1 (2023.06.14) - https://github.com/s5uishida/install_vpp_upf_dpdk
- UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM/wiki/Installation

<h3 id="changes_cp">Changes in configuration files of free5GC 5GC C-Plane</h3>

The combination of DNN and S-NSSAI parameters can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- S-NSSAI

For the sake of simplicity, This time, only DNN will be changed. S-NSSAI of all UEs is fixed as `SST=1` and `SD=010203`.

- `free5gc/config/amfcfg.yaml`
```diff
--- amfcfg.yaml.orig    2022-04-01 20:25:54.000000000 +0900
+++ amfcfg.yaml 2023-06-15 22:19:52.212588413 +0900
@@ -5,7 +5,7 @@
 configuration:
   amfName: AMF # the name of this AMF
   ngapIpList:  # the IP list of N2 interfaces on this AMF
-    - 127.0.0.18
+    - 192.168.0.141
   sbi: # Service-based interface information
     scheme: http # the protocol for sbi (http or https)
     registerIPv4: 127.0.0.18 # IP used to register to NRF
@@ -23,18 +23,18 @@
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
       tac: 1 # Tracking Area Code (uinteger, range: 0~16777215)
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
--- ausfcfg.yaml.orig   2022-04-01 20:25:54.000000000 +0900
+++ ausfcfg.yaml        2023-06-15 22:21:03.494330601 +0900
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
--- nrfcfg.yaml.orig    2023-03-18 02:14:22.000000000 +0900
+++ nrfcfg.yaml 2023-06-15 22:21:32.517467671 +0900
@@ -14,8 +14,8 @@
     # rootcert: config/cert/root.pem # Root Certificate
     # oauth: false # OAuth2
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
--- nssfcfg.yaml.orig   2022-04-01 20:25:54.000000000 +0900
+++ nssfcfg.yaml        2023-06-15 22:22:04.020942866 +0900
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
--- smfcfg.yaml.orig    2022-04-01 20:25:54.000000000 +0900
+++ smfcfg.yaml 2023-06-15 22:22:33.420602704 +0900
@@ -32,18 +32,18 @@
           dns: # the IP address of DNS
             ipv4: 8.8.8.8
   plmnList: # the list of PLMN IDs that this SMF belongs to (optional, remove this key when unnecessary)
-    - mcc: "208" # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: "93" # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: "001" # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: "01" # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   locality: area1 # Name of the location where a set of AMF, SMF and UPFs are located
   pfcp: # the IP address of N4 interface on this SMF (PFCP)
-    addr: 127.0.0.1
+    addr: 192.168.14.141
   userplaneInformation: # list of userplane information
     upNodes: # information of userplane node (AN or UPF)
       gNB1: # the name of the node
         type: AN # the type of the node (AN or UPF)
       UPF:  # the name of the node
         type: UPF # the type of the node (AN or UPF)
-        nodeID: 127.0.0.8 # the IP/FQDN of N4 interface on this UPF (PFCP)
+        nodeID: 192.168.14.151 # the IP/FQDN of N4 interface on this UPF (PFCP)
         sNssaiUpfInfos: # S-NSSAI information list for this UPF
           - sNssai: # S-NSSAI (Single Network Slice Selection Assistance Information)
               sst: 1 # Slice/Service Type (uinteger, range: 0~255)
@@ -62,12 +62,13 @@
         interfaces: # Interface list for this UPF
           - interfaceType: N3 # the type of the interface (N3 or N9)
             endpoints: # the IP address of this N3/N9 interface on this UPF
-              - 127.0.0.8
+              - 192.168.13.151
             networkInstance: internet # Data Network Name (DNN)
     links: # the topology graph of userplane, A and B represent the two nodes of each link
       - A: gNB1
         B: UPF
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
+  ulcl: false
 
 # the kind of log output
 # debugLevel: how detailed to output, value: trace, debug, info, warn, error, fatal, panic
```

<h3 id="changes_up">Changes in configuration files of VPP-UPF</h3>

See [here](https://github.com/s5uishida/install_vpp_upf_dpdk#create-configuration-files) for the original files.

- `openair-upf/startup.conf`  
There is no change.

- `openair-upf/init.conf`  
There is no change.

<h3 id="changes_ueransim">Changes in configuration files of UERANSIM UE / RAN</h3>

<h4 id="changes_ran">Changes in configuration files of RAN</h4>

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

<h4 id="changes_ue">Changes in configuration files of UE (IMSI-001010000000000)</h4>

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

<h2 id="network_settings">Network settings of free5GC 5GC, VPP-UPF and UERANSIM UE / RAN</h2>

<h3 id="network_settings_up">Network settings of VPP-UPF and Data Network Gateway</h3>

See [this1](https://github.com/s5uishida/install_vpp_upf_dpdk#setup-vpp-upf-with-dpdk-on-vm-up) and [this2](https://github.com/s5uishida/install_vpp_upf_dpdk#setup-data-network-gateway-on-vm-dn).

<h2 id="build">Build free5GC, VPP-UPF and UERANSIM</h2>

Please refer to the following for building free5GC, VPP-UPF and UERANSIM respectively.
- free5GC v3.2.1 (2023.02.12) - https://github.com/free5gc/free5gc/wiki/Installation
- OpenAir CN 5G for UPF v1.5.1 (2023.06.14) - https://github.com/s5uishida/install_vpp_upf_dpdk
- UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM/wiki/Installation

Install MongoDB on free5GC 5GC C-Plane machine.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

**Note. If you want to use the latest committed version, please run the following script to checkout all NFs and Web Console to the latest `main` branch before building.**
```bash
#!/usr/bin/env bash

NF_LIST="nrf amf smf udr pcf udm nssf ausf upf n3iwf"

for NF in ${NF_LIST}; do
    cd NFs/${NF}
    git checkout main
    cd ../..
done

cd webconsole
git checkout main

cd ..
git checkout main
```

<h2 id="run">Run free5GC 5GC, VPP-UPF and UERANSIM UE / RAN</h2>

First run VPP-UPF, then the 5GC and UERANSIM (UE & RAN implementation).

<h3 id="run_up">Run VPP-UPF</h3>

See [this](https://github.com/s5uishida/install_vpp_upf_dpdk#run-vpp-upf-with-dpdk-on-vm-up).

<h3 id="run_cp">Run free5GC 5GC C-Plane</h3>

Next, run free5GC 5GC C-Plane.
Create the following shell script and run it.
```bash
#!/usr/bin/env bash

PID_LIST=()

NF_LIST="nrf amf smf udr pcf udm nssf ausf"

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
  Recovery Time Stamp: 2023/06/15 22:26:23:000
  Sessions: 0
vpp#
```

<h3 id="run_ueran">Run UERANSIM</h3>

Here, the case of UE (IMSI-001010000000000) & RAN is described.
First, do an NG Setup between gNodeB and 5GC, then register the UE with 5GC and establish a PDU session.

Please refer to the following for usage of UERANSIM.

https://github.com/aligungr/UERANSIM/wiki/Usage

<h4 id="start_gnb">Start gNB</h4>

Start gNB as follows.
```
# ./nr-gnb -c ../config/free5gc-gnb.yaml
UERANSIM v3.2.6
[2023-06-15 22:27:17.175] [sctp] [info] Trying to establish SCTP connection... (192.168.0.141:38412)
[2023-06-15 22:27:17.178] [sctp] [info] SCTP connection established (192.168.0.141:38412)
[2023-06-15 22:27:17.179] [sctp] [debug] SCTP association setup ascId[4]
[2023-06-15 22:27:17.179] [ngap] [debug] Sending NG Setup Request
[2023-06-15 22:27:17.190] [ngap] [debug] NG Setup Response received
[2023-06-15 22:27:17.190] [ngap] [info] NG Setup procedure is successful
```
The free5GC C-Plane log when executed is as follows.
```
2023-06-15T22:27:17.152262195+09:00 [INFO][AMF][NGAP] [AMF] SCTP Accept from: 192.168.0.131:59319
2023-06-15T22:27:17.152946853+09:00 [INFO][AMF][NGAP] Create a new NG connection for: 192.168.0.131:59319
2023-06-15T22:27:17.161408967+09:00 [INFO][AMF][NGAP][192.168.0.131:59319] Handle NG Setup request
2023-06-15T22:27:17.161737854+09:00 [INFO][AMF][NGAP][192.168.0.131:59319] Send NG-Setup response
```

<h4 id="start_ue">Start UE</h4>

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/free5gc-ue.yaml
UERANSIM v3.2.6
[2023-06-15 22:28:40.518] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2023-06-15 22:28:40.518] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2023-06-15 22:28:40.519] [nas] [info] Selected plmn[001/01]
[2023-06-15 22:28:40.520] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2023-06-15 22:28:40.520] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2023-06-15 22:28:40.520] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2023-06-15 22:28:40.521] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2023-06-15 22:28:40.523] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2023-06-15 22:28:40.523] [nas] [debug] Sending Initial Registration
[2023-06-15 22:28:40.523] [rrc] [debug] Sending RRC Setup Request
[2023-06-15 22:28:40.523] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2023-06-15 22:28:40.524] [rrc] [info] RRC connection established
[2023-06-15 22:28:40.524] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2023-06-15 22:28:40.524] [nas] [info] UE switches to state [CM-CONNECTED]
[2023-06-15 22:28:40.564] [nas] [debug] Authentication Request received
[2023-06-15 22:28:40.578] [nas] [debug] Security Mode Command received
[2023-06-15 22:28:40.578] [nas] [debug] Selected integrity[2] ciphering[0]
[2023-06-15 22:28:40.621] [nas] [debug] Registration accept received
[2023-06-15 22:28:40.621] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2023-06-15 22:28:40.621] [nas] [debug] Sending Registration Complete
[2023-06-15 22:28:40.622] [nas] [info] Initial Registration is successful
[2023-06-15 22:28:40.622] [nas] [debug] Sending PDU Session Establishment Request
[2023-06-15 22:28:40.622] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2023-06-15 22:28:40.882] [nas] [debug] PDU Session Establishment Accept received
[2023-06-15 22:28:40.885] [nas] [info] PDU Session establishment is successful PSI[1]
[2023-06-15 22:28:40.906] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
The free5GC C-Plane log when executed is as follows.
```
2023-06-15T22:28:40.548175229+09:00 [INFO][AMF][NGAP][192.168.0.131:59319] Handle Initial UE Message
2023-06-15T22:28:40+09:00 [INFO][LIB][FSM] Handle event[Gmm Message], transition from [Deregistered] to [Deregistered]
2023-06-15T22:28:40.555318347+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1] Handle Registration Request
2023-06-15T22:28:40+09:00 [INFO][LIB][FSM] Handle event[Start Authentication], transition from [Deregistered] to [Authentication]
2023-06-15T22:28:40.555825574+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1] Authentication procedure
2023-06-15T22:28:40.556912069+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.558396783+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=AUSF |
2023-06-15T22:28:40.560624764+09:00 [INFO][AUSF][UeAuthPost] HandleUeAuthPostRequest
2023-06-15T22:28:40.560706814+09:00 [INFO][AUSF][UeAuthPost] Serving network authorized
2023-06-15T22:28:40.561599351+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.562841982+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AUSF&service-names=nudm-ueau&target-nf-type=UDM |
2023-06-15T22:28:40.567625275+09:00 [INFO][UDM][UEAU] Handle GenerateAuthDataRequest
2023-06-15T22:28:40.567884823+09:00 [INFO][UDM][Suci] suciPart: [suci 0 001 01 0000 0 0 0000000000]
2023-06-15T22:28:40.568057543+09:00 [INFO][UDM][Suci] scheme 0
2023-06-15T22:28:40.568257425+09:00 [INFO][UDM][Suci] SUPI type is IMSI
http://127.0.0.10:8000
2023-06-15T22:28:40.570580047+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.57170843+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=UDM&target-nf-type=UDR |
2023-06-15T22:28:40.575697619+09:00 [INFO][UDR][DRepo] Handle QueryAuthSubsData
2023-06-15T22:28:40.579016849+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2023-06-15T22:28:40.579749283+09:00 [INFO][UDM][UEAU] Nil Op
2023-06-15T22:28:40.580741853+09:00 [INFO][UDR][DRepo] Handle ModifyAuthentication
2023-06-15T22:28:40.582417773+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PATCH   | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2023-06-15T22:28:40.582930318+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | POST    | /nudm-ueau/v1/suci-0-001-01-0000-0-0-0000000000/security-information/generate-auth-data |
2023-06-15T22:28:40.583458341+09:00 [INFO][AUSF][UeAuthPost] Add SuciSupiPair (suci-0-001-01-0000-0-0-0000000000, imsi-001010000000000) to map.
2023-06-15T22:28:40.583684542+09:00 [INFO][AUSF][UeAuthPost] Use 5G AKA auth method
2023-06-15T22:28:40.58406113+09:00 [INFO][AUSF][5gAkaAuth] XresStar = 6266643133313632303962336232383137653830346161323039363636303433
2023-06-15T22:28:40.584600241+09:00 [INFO][AUSF][GIN] | 201 |       127.0.0.1 | POST    | /nausf-auth/v1/ue-authentications |
2023-06-15T22:28:40.585200922+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1] Send Authentication Request
2023-06-15T22:28:40.586398437+09:00 [INFO][AMF][NGAP][192.168.0.131:59319][AMF_UE_NGAP_ID:1] Send Downlink Nas Transport
2023-06-15T22:28:40.58877823+09:00 [INFO][AMF][NGAP][192.168.0.131:59319][AMF_UE_NGAP_ID:1] Uplink NAS Transport (RAN UE NGAP ID: 1)
2023-06-15T22:28:40+09:00 [INFO][LIB][FSM] Handle event[Gmm Message], transition from [Authentication] to [Authentication]
2023-06-15T22:28:40.589318491+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1] Handle Authentication Response
2023-06-15T22:28:40.590330067+09:00 [INFO][AUSF][5gAkaAuth] Auth5gAkaComfirmRequest
2023-06-15T22:28:40.590543166+09:00 [INFO][AUSF][5gAkaAuth] res*: 6266643133313632303962336232383137653830346161323039363636303433
Xres*: 6266643133313632303962336232383137653830346161323039363636303433
2023-06-15T22:28:40.591517574+09:00 [INFO][AUSF][5gAkaAuth] 5G AKA confirmation succeeded
2023-06-15T22:28:40.59402055+09:00 [INFO][UDM][UEAU] Handle ConfirmAuthDataRequest
2023-06-15T22:28:40.59575582+09:00 [INFO][UDR][DRepo] Handle CreateAuthenticationStatus
2023-06-15T22:28:40.59833415+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-status |
2023-06-15T22:28:40.598648927+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-ueau/v1/imsi-001010000000000/auth-events |
2023-06-15T22:28:40.599209287+09:00 [INFO][AUSF][GIN] | 200 |       127.0.0.1 | PUT     | /nausf-auth/v1/ue-authentications/suci-0-001-01-0000-0-0-0000000000/5g-aka-confirmation |
2023-06-15T22:28:40+09:00 [INFO][LIB][FSM] Handle event[Authentication Success], transition from [Authentication] to [SecurityMode]
2023-06-15T22:28:40.600007411+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1][SUPI:imsi-001010000000000] Send Security Mode Command
2023-06-15T22:28:40.600226369+09:00 [INFO][AMF][NGAP][192.168.0.131:59319][AMF_UE_NGAP_ID:1] Send Downlink Nas Transport
2023-06-15T22:28:40.602238191+09:00 [INFO][AMF][NGAP][192.168.0.131:59319][AMF_UE_NGAP_ID:1] Uplink NAS Transport (RAN UE NGAP ID: 1)
2023-06-15T22:28:40+09:00 [INFO][LIB][FSM] Handle event[Gmm Message], transition from [SecurityMode] to [SecurityMode]
2023-06-15T22:28:40.602631859+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1][SUPI:imsi-001010000000000] Handle Security Mode Complete
2023-06-15T22:28:40+09:00 [INFO][LIB][FSM] Handle event[SecurityMode Success], transition from [SecurityMode] to [ContextSetup]
2023-06-15T22:28:40.602994579+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1][SUPI:imsi-001010000000000] Handle InitialRegistration
2023-06-15T22:28:40.603721819+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.60491682+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2023-06-15T22:28:40.606007991+09:00 [INFO][UDM][SDM] Handle GetNssai
2023-06-15T22:28:40.607509982+09:00 [INFO][UDR][DRepo] Handle QueryAmData
2023-06-15T22:28:40.609509746+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features= |
2023-06-15T22:28:40.609934438+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/nssai?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-06-15T22:28:40.610347986+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1][SUPI:imsi-001010000000000] RequestedNssai - ServingSnssai: &{Sst:1 Sd:010203}, HomeSnssai: <nil>
2023-06-15T22:28:40.611181051+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.612363591+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2023-06-15T22:28:40.613959127+09:00 [INFO][UDM][UECM] Handle RegistrationAmf3gppAccess
2023-06-15T22:28:40.614276429+09:00 [INFO][UDM][UECM] UEID: imsi-001010000000000
2023-06-15T22:28:40.615307119+09:00 [INFO][UDR][DRepo] Handle CreateAmfContext3gpp
2023-06-15T22:28:40.617732098+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/amf-3gpp-access |
2023-06-15T22:28:40.618023277+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | PUT     | /nudm-uecm/v1/imsi-001010000000000/registrations/amf-3gpp-access |
2023-06-15T22:28:40.618556938+09:00 [INFO][UDM][SDM] Handle GetAmData
2023-06-15T22:28:40.618901249+09:00 [INFO][UDR][DRepo] Handle QueryAmData
2023-06-15T22:28:40.619506789+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-06-15T22:28:40.61993813+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/am-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-06-15T22:28:40.62054775+09:00 [INFO][UDM][SDM] Handle GetSmfSelectData
2023-06-15T22:28:40.620902407+09:00 [INFO][UDR][DRepo] Handle QuerySmfSelectData
2023-06-15T22:28:40.622661569+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/smf-selection-subscription-data?supported-features= |
2023-06-15T22:28:40.623078201+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/smf-select-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-06-15T22:28:40.623618751+09:00 [INFO][UDM][SDM] Handle GetUeContextInSmfData
2023-06-15T22:28:40.624369693+09:00 [INFO][UDR][DRepo] Handle QuerySmfRegList
2023-06-15T22:28:40.624790608+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/smf-registrations?supported-features= |
2023-06-15T22:28:40.625366578+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/ue-context-in-smf-data |
2023-06-15T22:28:40.625933854+09:00 [INFO][UDM][SDM] Handle Subscribe
2023-06-15T22:28:40.626514872+09:00 [INFO][UDR][DRepo] Handle CreateSdmSubscriptions
2023-06-15T22:28:40.626790656+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |
2023-06-15T22:28:40.627355819+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v1/imsi-001010000000000/sdm-subscriptions |
2023-06-15T22:28:40.628061066+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.629567346+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=PCF |
2023-06-15T22:28:40.631812112+09:00 [INFO][PCF][Ampolicy] Handle AM Policy Create Request
2023-06-15T22:28:40.632536316+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.633448226+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=UDR |
2023-06-15T22:28:40.634448035+09:00 [INFO][UDR][DRepo] Handle PolicyDataUesUeIdAmDataGet
2023-06-15T22:28:40.636293105+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/policy-data/ues/imsi-001010000000000/am-data |
2023-06-15T22:28:40.637275315+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.638650507+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?guami=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22amfId%22%3A%22cafe00%22%7D&requester-nf-type=PCF&target-nf-type=AMF |
2023-06-15T22:28:40.641016734+09:00 [INFO][AMF][Comm] Handle AMF Status Change Subscribe Request
2023-06-15T22:28:40.641225799+09:00 [INFO][AMF][Comm] new AMF Status Subscription[1]
2023-06-15T22:28:40.641411481+09:00 [INFO][AMF][GIN] | 201 |       127.0.0.1 | POST    | /namf-comm/v1/subscriptions |
2023-06-15T22:28:40.641852286+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-am-policy-control/v1/policies |
2023-06-15T22:28:40.642438742+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1][SUPI:imsi-001010000000000] Send Registration Accept
2023-06-15T22:28:40.642681023+09:00 [INFO][AMF][NGAP][192.168.0.131:59319][AMF_UE_NGAP_ID:1] Send Initial Context Setup Request
2023-06-15T22:28:40.644459408+09:00 [INFO][AMF][NGAP][192.168.0.131:59319][AMF_UE_NGAP_ID:1] Handle Initial Context Setup Response
2023-06-15T22:28:40.848327908+09:00 [INFO][AMF][NGAP][192.168.0.131:59319][AMF_UE_NGAP_ID:1] Uplink NAS Transport (RAN UE NGAP ID: 1)
2023-06-15T22:28:40+09:00 [INFO][LIB][FSM] Handle event[Gmm Message], transition from [ContextSetup] to [ContextSetup]
2023-06-15T22:28:40.848862005+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1][SUPI:imsi-001010000000000] Handle Registration Complete
2023-06-15T22:28:40+09:00 [INFO][LIB][FSM] Handle event[ContextSetup Success], transition from [ContextSetup] to [Registered]
2023-06-15T22:28:40.849858163+09:00 [INFO][AMF][NGAP][192.168.0.131:59319][AMF_UE_NGAP_ID:1] Uplink NAS Transport (RAN UE NGAP ID: 1)
2023-06-15T22:28:40+09:00 [INFO][LIB][FSM] Handle event[Gmm Message], transition from [Registered] to [Registered]
2023-06-15T22:28:40.850267165+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1][SUPI:imsi-001010000000000] Handle UL NAS Transport
2023-06-15T22:28:40.850428111+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1][SUPI:imsi-001010000000000] Transport 5GSM Message to SMF
2023-06-15T22:28:40.850702624+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1][SUPI:imsi-001010000000000] Select SMF [snssai: {Sst:1 Sd:010203}, dnn: internet]
2023-06-15T22:28:40.851496662+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.852797426+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=NSSF |
2023-06-15T22:28:40.856183134+09:00 [INFO][NSSF][NsSelect] Handle NSSelectionGet
2023-06-15T22:28:40.856865134+09:00 [INFO][NSSF][GIN] | 200 |       127.0.0.1 | GET     | /nnssf-nsselection/v1/network-slice-information?nf-id=47931d2e-4878-4c48-bd5c-14a3be16e49d&nf-type=AMF&slice-info-request-for-pdu-session=%7B%22sNssai%22%3A%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%2C%22roamingIndication%22%3A%22NON_ROAMING%22%7D |
2023-06-15T22:28:40.857790014+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.859248616+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?dnn=internet&preferred-locality=area1&requester-nf-type=AMF&service-names=nsmf-pdusession&snssais=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&target-nf-type=SMF&target-plmn-list=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-06-15T22:28:40.861222616+09:00 [INFO][SMF][PduSess] Receive Create SM Context Request
2023-06-15T22:28:40.862251278+09:00 [INFO][SMF][PduSess] In HandlePDUSessionSMContextCreate
2023-06-15T22:28:40.863541639+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.864856917+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=UDM |
2023-06-15T22:28:40.865317023+09:00 [INFO][SMF][PduSess] Send NF Discovery Serving UDM Successfully
2023-06-15T22:28:40.865508003+09:00 [INFO][SMF][CTX] Allocated UE IP address: 10.60.0.1
2023-06-15T22:28:40.865782253+09:00 [INFO][SMF][CTX] Selected UPF: UPF
2023-06-15T22:28:40.865949185+09:00 [INFO][SMF][PduSess] UE[imsi-001010000000000] PDUSessionID[1] IP[10.60.0.1]
2023-06-15T22:28:40.866828643+09:00 [INFO][UDM][SDM] Handle GetSmData
2023-06-15T22:28:40.86708752+09:00 [INFO][UDM][SDM] getSmDataProcedure: SUPI[imsi-001010000000000] PLMNID[00101] DNN[internet] SNssai[{"sst":1,"sd":"010203"}]
2023-06-15T22:28:40.867993158+09:00 [INFO][UDR][DRepo] Handle QuerySmData
2023-06-15T22:28:40.870154683+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/sm-data?single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2023-06-15T22:28:40.870636952+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/sm-data?dnn=internet&plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D&single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2023-06-15T22:28:40.871335648+09:00 [INFO][SMF][GSM] In HandlePDUSessionEstablishmentRequest
2023-06-15T22:28:40+09:00 [INFO][NAS][Convert] ProtocolOrContainerList:  [0xc0003fc640 0xc0003fc660]
2023-06-15T22:28:40.871734584+09:00 [INFO][SMF][GSM] Protocol Configuration Options
2023-06-15T22:28:40.872250784+09:00 [INFO][SMF][GSM] &{[0xc0003fc640 0xc0003fc660]}
2023-06-15T22:28:40.872448287+09:00 [INFO][SMF][GSM] Didn't Implement container type IPAddressAllocationViaNASSignallingUL
2023-06-15T22:28:40.872638969+09:00 [INFO][SMF][PduSess] PCF Selection for SMContext SUPI[imsi-001010000000000] PDUSessionID[1]
2023-06-15T22:28:40.873616357+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.874905377+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=SMF&target-nf-type=PCF |
2023-06-15T22:28:40.876395141+09:00 [INFO][PCF][SMpolicy] Handle CreateSmPolicy
2023-06-15T22:28:40.87728894+09:00 [INFO][UDR][DRepo] Handle PolicyDataUesUeIdSmDataGet
2023-06-15T22:28:40.879870931+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/policy-data/ues/imsi-001010000000000/sm-data?dnn=internet&snssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2023-06-15T22:28:40.881867205+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-smpolicycontrol/v1/sm-policies |
2023-06-15T22:28:40.883014023+09:00 [INFO][SMF][PduSess] SUPI[imsi-001010000000000] has no pre-config route
2023-06-15T22:28:40.883889081+09:00 [INFO][NRF][DSCV] Handle NFDiscoveryRequest
2023-06-15T22:28:40.885407525+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-instance-id=47931d2e-4878-4c48-bd5c-14a3be16e49d&target-nf-type=AMF |
2023-06-15T22:28:40.888805895+09:00 [INFO][SMF][Consumer] SendNFDiscoveryServingAMF ok
2023-06-15T22:28:40.889390596+09:00 [INFO][SMF][GIN] | 201 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts |
2023-06-15T22:28:40.889754255+09:00 [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2023-06-15T22:28:40.890576068+09:00 [INFO][AMF][GMM][AMF_UE_NGAP_ID:1][SUPI:imsi-001010000000000] create smContext[pduSessionID: 1] Success
2023-06-15T22:28:40.897623819+09:00 [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2023-06-15T22:28:40.901035424+09:00 [INFO][AMF][Producer] Handle N1N2 Message Transfer Request
2023-06-15T22:28:40.901269239+09:00 [INFO][AMF][NGAP][192.168.0.131:59319][AMF_UE_NGAP_ID:1] Send PDU Session Resource Setup Request
2023-06-15T22:28:40.902280389+09:00 [INFO][AMF][GIN] | 200 |       127.0.0.1 | POST    | /namf-comm/v1/ue-contexts/imsi-001010000000000/n1-n2-messages |
2023-06-15T22:28:40.905226175+09:00 [INFO][AMF][NGAP][192.168.0.131:59319][AMF_UE_NGAP_ID:1] Handle PDU Session Resource Setup Response
2023-06-15T22:28:40.908302908+09:00 [INFO][SMF][PduSess] Receive Update SM Context Request
2023-06-15T22:28:40.908926803+09:00 [INFO][SMF][PduSess] In HandlePDUSessionSMContextUpdate
2023-06-15T22:28:40.909507482+09:00 [INFO][SMF][PduSess] Sending PFCP Session Modification Request to AN UPF
2023-06-15T22:28:40.928409293+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Accepted Response from AN UPF
2023-06-15T22:28:40.928773676+09:00 [INFO][SMF][GIN] | 200 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts/urn:uuid:4ea63fc7-2a83-410a-bd64-6b8a1577de69/modify |
```
The PDU session establishment status of VPP-UPF is as follows.
```
vpp# show upf session 
CP F-SEID: 0x0000000000000001 (1) @ 192.168.14.141
UP F-SEID: 0x0000000000000001 (1) @ 192.168.14.151
  PFCP Association: 0
PDR: 1 @ 0x7f6091e57490
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
  URR Ids: [] @ 0x0
  QER Ids: [1] @ 0x7f6091ce2770
PDR: 2 @ 0x7f6091e57510
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
  URR Ids: [] @ 0x0
  QER Ids: [1] @ 0x7f6091ce2800
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
vpp#
```
Looking at the console log of the `nr-ue` command, UE has been assigned the IP address `10.60.0.1` from free5GC 5GC.
```
[2023-06-15 22:28:40.906] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
5: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.60.0.1/32 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::4091:8638:1191:5d8e/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
...
```

<h2 id="ping">Ping google.com</h2>

Specify the UE's TUNnel interface and try ping.

Please refer to the following for usage of TUNnel interface.

https://github.com/aligungr/UERANSIM/wiki/Usage

<h3 id="ping_1">Case for going through DN 10.60.0.0/16</h3>

Run `tcpdump` on VM-DN and check that the packet goes through N6 (enp0s9).
- `ping google.com` on VM3 (UE)
```
# ping google.com -I uesimtun0 -n
PING google.com (172.217.175.78) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 172.217.175.78: icmp_seq=1 ttl=59 time=19.5 ms
64 bytes from 172.217.175.78: icmp_seq=2 ttl=59 time=27.5 ms
64 bytes from 172.217.175.78: icmp_seq=3 ttl=59 time=15.8 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i enp0s9 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), snapshot length 262144 bytes
22:33:51.198657 IP 10.60.0.1 > 172.217.175.78: ICMP echo request, id 3, seq 1, length 64
22:33:51.217348 IP 172.217.175.78 > 10.60.0.1: ICMP echo reply, id 3, seq 1, length 64
22:33:52.200409 IP 10.60.0.1 > 172.217.175.78: ICMP echo request, id 3, seq 2, length 64
22:33:52.227052 IP 172.217.175.78 > 10.60.0.1: ICMP echo reply, id 3, seq 2, length 64
22:33:53.201199 IP 10.60.0.1 > 172.217.175.78: ICMP echo request, id 3, seq 3, length 64
22:33:53.216222 IP 172.217.175.78 > 10.60.0.1: ICMP echo reply, id 3, seq 3, length 64
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
22:34:39.401995 IP 10.60.0.1.35159 > 172.217.31.174.80: Flags [S], seq 959375985, win 65280, options [mss 1360,sackOK,TS val 1336707287 ecr 0,nop,wscale 7], length 0
22:34:39.418832 IP 172.217.31.174.80 > 10.60.0.1.35159: Flags [S.], seq 8640001, ack 959375986, win 65535, options [mss 1460], length 0
22:34:39.419549 IP 10.60.0.1.35159 > 172.217.31.174.80: Flags [.], ack 1, win 65280, length 0
22:34:39.419744 IP 10.60.0.1.35159 > 172.217.31.174.80: Flags [P.], seq 1:75, ack 1, win 65280, length 74: HTTP: GET / HTTP/1.1
22:34:39.419835 IP 172.217.31.174.80 > 10.60.0.1.35159: Flags [.], ack 75, win 65535, length 0
22:34:39.477449 IP 172.217.31.174.80 > 10.60.0.1.35159: Flags [P.], seq 1:774, ack 75, win 65535, length 773: HTTP: HTTP/1.1 301 Moved Permanently
22:34:39.478285 IP 10.60.0.1.35159 > 172.217.31.174.80: Flags [.], ack 774, win 64507, length 0
22:34:39.479895 IP 10.60.0.1.35159 > 172.217.31.174.80: Flags [F.], seq 75, ack 774, win 64507, length 0
22:34:39.480005 IP 172.217.31.174.80 > 10.60.0.1.35159: Flags [.], ack 76, win 65535, length 0
22:34:39.496579 IP 172.217.31.174.80 > 10.60.0.1.35159: Flags [F.], seq 774, ack 76, win 65535, length 0
22:34:39.497360 IP 10.60.0.1.35159 > 172.217.31.174.80: Flags [.], ack 775, win 64507, length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using VPP-UPF with DPDK.

---

Now you could work free5GC with VPP-UPF.
I would like to thank the excellent developers and all the contributors of free5GC, OpenAir CN 5G for UPF, UPG-VPP and DPDK.

<h2 id="changelog">Changelog (summary)</h2>

- [2023.06.15] Initial release.
