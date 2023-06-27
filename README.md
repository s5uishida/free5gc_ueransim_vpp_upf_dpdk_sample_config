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
- 5GC - free5GC v3.3.0 (2023.06.27) - https://github.com/free5gc/free5gc
- VPP-UPF - OpenAir CN 5G for UPF v1.5.1 (2023.06.14) - https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp
- UE / RAN - UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Memory<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | free5GC 5GC C-Plane | 192.168.0.141/24 | Ubuntu 22.04 | 1 | 2GB | 20GB |
| VM-UP | OpenAir CN 5G for UPF | 192.168.0.151/24 | Ubuntu 22.04 | 2 | 8GB | 20GB |
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

<h2 id="changes">Changes in configuration files of free5GC 5GC, VPP-UPF and UERANSIM UE / RAN</h2>

Please refer to the following for building free5GC, VPP-UPF and UERANSIM respectively.
- free5GC v3.3.0 (2023.06.27) - https://github.com/free5gc/free5gc/wiki/Installation
- OpenAir CN 5G for UPF v1.5.1 (2023.06.14) - https://github.com/s5uishida/install_vpp_upf_dpdk
- UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM/wiki/Installation

<h3 id="changes_cp">Changes in configuration files of free5GC 5GC C-Plane</h3>

The combination of DNN and S-NSSAI parameters can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- S-NSSAI

For the sake of simplicity, This time, only DNN will be changed. S-NSSAI of all UEs is fixed as `SST=1` and `SD=010203`.

- `free5gc/config/amfcfg.yaml`
```diff
--- amfcfg.yaml.orig    2023-06-27 19:09:10.707252803 +0900
+++ amfcfg.yaml 2023-06-27 19:13:07.031736927 +0900
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
--- ausfcfg.yaml.orig   2023-06-27 19:09:10.707252803 +0900
+++ ausfcfg.yaml        2023-06-27 19:13:38.268247295 +0900
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
--- nrfcfg.yaml.orig    2023-06-27 19:09:10.708252882 +0900
+++ nrfcfg.yaml 2023-06-27 19:47:32.345722855 +0900
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
--- nssfcfg.yaml.orig   2023-06-27 19:09:10.708252882 +0900
+++ nssfcfg.yaml        2023-06-27 19:13:46.780422895 +0900
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
--- smfcfg.yaml.orig    2023-06-27 19:09:10.708252882 +0900
+++ smfcfg.yaml 2023-06-27 19:13:49.164469607 +0900
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
@@ -91,6 +91,8 @@
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   #urrPeriod: 10 # default usage report period in seconds
   #urrThreshold: 1000 # default usage report threshold in bytes
+  ulcl: false
+  nwInstFqdnEncoding: true
 
 logger: # log output setting
   enable: true # true or false
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
- free5GC v3.3.0 (2023.06.27) - https://github.com/free5gc/free5gc/wiki/Installation
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
  Recovery Time Stamp: 2023/06/27 19:35:16:000
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
[2023-06-27 19:35:52.272] [sctp] [info] Trying to establish SCTP connection... (192.168.0.141:38412)
[2023-06-27 19:35:52.275] [sctp] [info] SCTP connection established (192.168.0.141:38412)
[2023-06-27 19:35:52.275] [sctp] [debug] SCTP association setup ascId[4]
[2023-06-27 19:35:52.275] [ngap] [debug] Sending NG Setup Request
[2023-06-27 19:35:52.277] [ngap] [debug] NG Setup Response received
[2023-06-27 19:35:52.277] [ngap] [info] NG Setup procedure is successful
```
The free5GC C-Plane log when executed is as follows.
```
2023-06-27T19:35:52.275713178+09:00 [INFO][AMF][Ngap] [AMF] SCTP Accept from: 192.168.0.131:32966
2023-06-27T19:35:52.276508549+09:00 [INFO][AMF][Ngap] Create a new NG connection for: 192.168.0.131:32966
2023-06-27T19:35:52.277117776+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:32966] Handle NGSetupRequest
2023-06-27T19:35:52.277183570+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:32966] Send NG-Setup response
```

<h4 id="start_ue">Start UE</h4>

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/free5gc-ue.yaml
UERANSIM v3.2.6
[2023-06-27 19:36:33.987] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2023-06-27 19:36:33.988] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2023-06-27 19:36:33.988] [nas] [info] Selected plmn[001/01]
[2023-06-27 19:36:33.988] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2023-06-27 19:36:33.988] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2023-06-27 19:36:33.988] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2023-06-27 19:36:33.988] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2023-06-27 19:36:33.989] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2023-06-27 19:36:33.989] [nas] [debug] Sending Initial Registration
[2023-06-27 19:36:33.989] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2023-06-27 19:36:33.989] [rrc] [debug] Sending RRC Setup Request
[2023-06-27 19:36:33.990] [rrc] [info] RRC connection established
[2023-06-27 19:36:33.990] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2023-06-27 19:36:33.990] [nas] [info] UE switches to state [CM-CONNECTED]
[2023-06-27 19:36:34.017] [nas] [debug] Authentication Request received
[2023-06-27 19:36:34.027] [nas] [debug] Security Mode Command received
[2023-06-27 19:36:34.027] [nas] [debug] Selected integrity[2] ciphering[0]
[2023-06-27 19:36:34.064] [nas] [debug] Registration accept received
[2023-06-27 19:36:34.065] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2023-06-27 19:36:34.065] [nas] [debug] Sending Registration Complete
[2023-06-27 19:36:34.065] [nas] [info] Initial Registration is successful
[2023-06-27 19:36:34.065] [nas] [debug] Sending PDU Session Establishment Request
[2023-06-27 19:36:34.065] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2023-06-27 19:36:34.327] [nas] [debug] PDU Session Establishment Accept received
[2023-06-27 19:36:34.332] [nas] [info] PDU Session establishment is successful PSI[1]
[2023-06-27 19:36:34.356] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
The free5GC C-Plane log when executed is as follows.
```
2023-06-27T19:36:33.985203015+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:32966] Handle InitialUEMessage
2023-06-27T19:36:33.985437390+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:32966] New RanUe [RanUeNgapID:1][AmfUeNgapID:1]
2023-06-27T19:36:33.985668347+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:32966] 5GSMobileIdentity ["SUCI":"suci-0-001-01-0000-0-0-0000000000", err: <nil>]
2023-06-27T19:36:33.986292493+09:00 [INFO][AMF][CTX] New AmfUe [supi:][guti:00101cafe0000000001]
2023-06-27T19:36:33.986536256+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Deregistered] to [Deregistered]
2023-06-27T19:36:33.986859136+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Registration Request
2023-06-27T19:36:33.987012794+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] RegistrationType: Initial Registration
2023-06-27T19:36:33.987161489+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] MobileIdentity5GS: SUCI[suci-0-001-01-0000-0-0-0000000000]
2023-06-27T19:36:33.987309356+09:00 [INFO][AMF][Gmm] Handle event[Start Authentication], transition from [Deregistered] to [Authentication]
2023-06-27T19:36:33.987447441+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Authentication procedure
2023-06-27T19:36:33.988249723+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:33.990183396+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=AUSF |
2023-06-27T19:36:33.994704656+09:00 [INFO][AUSF][UeAuth] HandleUeAuthPostRequest
2023-06-27T19:36:33.994885157+09:00 [INFO][AUSF][UeAuth] Serving network authorized
2023-06-27T19:36:33.995696381+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:33.997093808+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AUSF&service-names=nudm-ueau&target-nf-type=UDM |
2023-06-27T19:36:33.998209762+09:00 [INFO][UDM][UEAU] Handle GenerateAuthDataRequest
2023-06-27T19:36:33.998438617+09:00 [INFO][UDM][Suci] suciPart: [suci 0 001 01 0000 0 0 0000000000]
2023-06-27T19:36:33.998629946+09:00 [INFO][UDM][Suci] scheme 0
2023-06-27T19:36:33.998813363+09:00 [INFO][UDM][Suci] SUPI type is IMSI
http://127.0.0.10:8000
2023-06-27T19:36:33.999738291+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:34.001838047+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=UDM&target-nf-type=UDR |
2023-06-27T19:36:34.002926773+09:00 [INFO][UDR][DataRepo] Handle QueryAuthSubsData
2023-06-27T19:36:34.004727015+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2023-06-27T19:36:34.005656021+09:00 [INFO][UDM][UEAU] Nil Op
2023-06-27T19:36:34.006128368+09:00 [INFO][UDR][DataRepo] Handle ModifyAuthentication
2023-06-27T19:36:34.008202478+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PATCH   | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |
2023-06-27T19:36:34.008734598+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | POST    | /nudm-ueau/v1/suci-0-001-01-0000-0-0-0000000000/security-information/generate-auth-data |
2023-06-27T19:36:34.009307205+09:00 [INFO][AUSF][UeAuth] Add SuciSupiPair (suci-0-001-01-0000-0-0-0000000000, imsi-001010000000000) to map.
2023-06-27T19:36:34.009582431+09:00 [INFO][AUSF][UeAuth] Use 5G AKA auth method
2023-06-27T19:36:34.009631716+09:00 [INFO][AUSF][5gAka] XresStar = 3532646535663236336166613966363661643363386439303761336465336365
2023-06-27T19:36:34.009748908+09:00 [INFO][AUSF][GIN] | 201 |       127.0.0.1 | POST    | /nausf-auth/v1/ue-authentications |
2023-06-27T19:36:34.010360278+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Send Authentication Request
2023-06-27T19:36:34.010548459+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:32966] Send Downlink Nas Transport
2023-06-27T19:36:34.010888582+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Start T3560 timer
2023-06-27T19:36:34.013005444+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:32966] Handle UplinkNASTransport
2023-06-27T19:36:34.013187641+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:32966] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2023-06-27T19:36:34.013388081+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Authentication] to [Authentication]
2023-06-27T19:36:34.013622026+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Authentication Response
2023-06-27T19:36:34.013778670+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Stop T3560 timer
2023-06-27T19:36:34.014647482+09:00 [INFO][AUSF][5gAka] Auth5gAkaComfirmRequest
2023-06-27T19:36:34.014803244+09:00 [INFO][AUSF][5gAka] res*: 3532646535663236336166613966363661643363386439303761336465336365
Xres*: 3532646535663236336166613966363661643363386439303761336465336365
2023-06-27T19:36:34.015362717+09:00 [INFO][AUSF][5gAka] 5G AKA confirmation succeeded
2023-06-27T19:36:34.016372150+09:00 [INFO][UDM][UEAU] Handle ConfirmAuthDataRequest
2023-06-27T19:36:34.017266206+09:00 [INFO][UDR][DataRepo] Handle CreateAuthenticationStatus
2023-06-27T19:36:34.018368912+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/authentication-data/authentication-status |
2023-06-27T19:36:34.018838558+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-ueau/v1/imsi-001010000000000/auth-events |
2023-06-27T19:36:34.019375044+09:00 [INFO][AUSF][GIN] | 200 |       127.0.0.1 | PUT     | /nausf-auth/v1/ue-authentications/suci-0-001-01-0000-0-0-0000000000/5g-aka-confirmation |
2023-06-27T19:36:34.020017999+09:00 [INFO][AMF][Gmm] Handle event[Authentication Success], transition from [Authentication] to [SecurityMode]
2023-06-27T19:36:34.020332050+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Security Mode Command
2023-06-27T19:36:34.020576300+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:32966] Send Downlink Nas Transport
2023-06-27T19:36:34.021175308+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3560 timer
2023-06-27T19:36:34.022801560+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:32966] Handle UplinkNASTransport
2023-06-27T19:36:34.022985915+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:32966] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2023-06-27T19:36:34.023170001+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [SecurityMode] to [SecurityMode]
2023-06-27T19:36:34.023356322+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Security Mode Complete
2023-06-27T19:36:34.023546463+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3560 timer
2023-06-27T19:36:34.023752152+09:00 [INFO][AMF][Gmm] Handle event[SecurityMode Success], transition from [SecurityMode] to [ContextSetup]
2023-06-27T19:36:34.023897141+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle InitialRegistration
2023-06-27T19:36:34.024631746+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:34.026073288+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2023-06-27T19:36:34.027055208+09:00 [INFO][UDM][SDM] Handle GetNssai
2023-06-27T19:36:34.027913716+09:00 [INFO][UDR][DataRepo] Handle QueryAmData
2023-06-27T19:36:34.028633285+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data |
2023-06-27T19:36:34.029155201+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/nssai?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-06-27T19:36:34.029749796+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] RequestedNssai - ServingSnssai: &{Sst:1 Sd:010203}, HomeSnssai: <nil>
2023-06-27T19:36:34.030711756+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:34.031868855+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |
2023-06-27T19:36:34.033171948+09:00 [INFO][UDM][UECM] Handle RegistrationAmf3gppAccess
2023-06-27T19:36:34.033395907+09:00 [INFO][UDM][UECM] UEID: imsi-001010000000000
2023-06-27T19:36:34.034251045+09:00 [INFO][UDR][DataRepo] Handle CreateAmfContext3gpp
2023-06-27T19:36:34.035326117+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/amf-3gpp-access |
2023-06-27T19:36:34.035772527+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | PUT     | /nudm-uecm/v1/imsi-001010000000000/registrations/amf-3gpp-access |
2023-06-27T19:36:34.036408062+09:00 [INFO][UDM][SDM] Handle GetAmData
2023-06-27T19:36:34.036868337+09:00 [INFO][UDR][DataRepo] Handle QueryAmData
2023-06-27T19:36:34.037556024+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-06-27T19:36:34.038002465+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/am-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-06-27T19:36:34.038738707+09:00 [INFO][UDM][SDM] Handle GetSmfSelectData
2023-06-27T19:36:34.039405164+09:00 [INFO][UDR][DataRepo] Handle QuerySmfSelectData
2023-06-27T19:36:34.039999248+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/smf-selection-subscription-data |
2023-06-27T19:36:34.040404510+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/smf-select-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-06-27T19:36:34.040928860+09:00 [INFO][UDM][SDM] Handle GetUeContextInSmfData
2023-06-27T19:36:34.041210981+09:00 [INFO][UDR][DataRepo] Handle QuerySmfRegList
2023-06-27T19:36:34.041851549+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/smf-registrations |
2023-06-27T19:36:34.042257488+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/ue-context-in-smf-data |
2023-06-27T19:36:34.042951866+09:00 [INFO][UDM][SDM] Handle Subscribe
2023-06-27T19:36:34.043456235+09:00 [INFO][UDR][DataRepo] Handle CreateSdmSubscriptions
2023-06-27T19:36:34.043760213+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v1/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |
2023-06-27T19:36:34.044146844+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v1/imsi-001010000000000/sdm-subscriptions |
2023-06-27T19:36:34.045087312+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:34.046417809+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=PCF |
2023-06-27T19:36:34.047957308+09:00 [INFO][PCF][AmPol] Handle AM Policy Create Request
2023-06-27T19:36:34.048864712+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:34.049710184+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=UDR |
2023-06-27T19:36:34.050736535+09:00 [INFO][UDR][DataRepo] Handle PolicyDataUesUeIdAmDataGet
2023-06-27T19:36:34.051301781+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/policy-data/ues/imsi-001010000000000/am-data |
2023-06-27T19:36:34.052265784+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:34.053808179+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?guami=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22amfId%22%3A%22cafe00%22%7D&requester-nf-type=PCF&target-nf-type=AMF |
2023-06-27T19:36:34.055049274+09:00 [INFO][AMF][Comm] Handle AMF Status Change Subscribe Request
2023-06-27T19:36:34.055259361+09:00 [INFO][AMF][Comm] new AMF Status Subscription[1]
2023-06-27T19:36:34.055435590+09:00 [INFO][AMF][GIN] | 201 |       127.0.0.1 | POST    | /namf-comm/v1/subscriptions |
2023-06-27T19:36:34.056094783+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-am-policy-control/v1/policies |
2023-06-27T19:36:34.056603644+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Registration Accept
2023-06-27T19:36:34.056896669+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:32966] Send Initial Context Setup Request
2023-06-27T19:36:34.057992407+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3550 timer
2023-06-27T19:36:34.059049993+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:32966] Handle InitialContextSetupResponse
2023-06-27T19:36:34.059232648+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:32966] Handle InitialContextSetupResponse (RAN UE NGAP ID: 1)
2023-06-27T19:36:34.265375387+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:32966] Handle UplinkNASTransport
2023-06-27T19:36:34.265809085+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:32966] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2023-06-27T19:36:34.266056306+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [ContextSetup] to [ContextSetup]
2023-06-27T19:36:34.266300794+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Registration Complete
2023-06-27T19:36:34.266551945+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3550 timer
2023-06-27T19:36:34.266791384+09:00 [INFO][AMF][Gmm] Handle event[ContextSetup Success], transition from [ContextSetup] to [Registered]
2023-06-27T19:36:34.267681355+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:32966] Handle UplinkNASTransport
2023-06-27T19:36:34.267891051+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:32966] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2023-06-27T19:36:34.268114041+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Registered] to [Registered]
2023-06-27T19:36:34.268317185+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle UL NAS Transport
2023-06-27T19:36:34.268559263+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Transport 5GSM Message to SMF
2023-06-27T19:36:34.268760196+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Select SMF [snssai: {Sst:1 Sd:010203}, dnn: internet]
2023-06-27T19:36:34.269836187+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:34.271162135+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=NSSF |
2023-06-27T19:36:34.272678065+09:00 [INFO][NSSF][NsSel] Handle NSSelectionGet
2023-06-27T19:36:34.273187034+09:00 [INFO][NSSF][GIN] | 200 |       127.0.0.1 | GET     | /nnssf-nsselection/v1/network-slice-information?nf-id=3da46200-5cf1-4f5b-9678-6b8ae2072fb1&nf-type=AMF&slice-info-request-for-pdu-session=%7B%22sNssai%22%3A%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%2C%22roamingIndication%22%3A%22NON_ROAMING%22%7D |
2023-06-27T19:36:34.274688645+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:34.275926819+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?dnn=internet&preferred-locality=area1&requester-nf-type=AMF&service-names=nsmf-pdusession&snssais=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&target-nf-type=SMF&target-plmn-list=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |
2023-06-27T19:36:34.277373824+09:00 [INFO][SMF][PduSess] Receive Create SM Context Request
2023-06-27T19:36:34.278099458+09:00 [INFO][SMF][PduSess] In HandlePDUSessionSMContextCreate
2023-06-27T19:36:34.278317322+09:00 [INFO][SMF][CTX] UrrPeriod: 0s
2023-06-27T19:36:34.278612463+09:00 [INFO][SMF][CTX] UrrThreshold: 0
2023-06-27T19:36:34.279259869+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:34.280666541+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=UDM |
2023-06-27T19:36:34.281129558+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Send NF Discovery Serving UDM Successfully
2023-06-27T19:36:34.281763896+09:00 [INFO][UDM][SDM] Handle GetSmData
2023-06-27T19:36:34.281846993+09:00 [INFO][UDM][SDM] getSmDataProcedure: SUPI[imsi-001010000000000] PLMNID[00101] DNN[internet] SNssai[{"sst":1,"sd":"010203"}]
2023-06-27T19:36:34.282422651+09:00 [INFO][UDR][DataRepo] Handle QuerySmData
2023-06-27T19:36:34.283369561+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/subscription-data/imsi-001010000000000/00101/provisioned-data/sm-data?single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2023-06-27T19:36:34.284013010+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v1/imsi-001010000000000/sm-data?dnn=internet&plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D&single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2023-06-27T19:36:34.284599325+09:00 [INFO][SMF][GSM] In HandlePDUSessionEstablishmentRequest
2023-06-27T19:36:34+09:00 [INFO][NAS][Convert] ProtocolOrContainerList:  [0xc00020ad60 0xc00020ad80]
2023-06-27T19:36:34.285009209+09:00 [INFO][SMF][GSM] Protocol Configuration Options
2023-06-27T19:36:34.285183515+09:00 [INFO][SMF][GSM] &{[0xc00020ad60 0xc00020ad80]}
2023-06-27T19:36:34.285329700+09:00 [INFO][SMF][GSM] Didn't Implement container type IPAddressAllocationViaNASSignallingUL
2023-06-27T19:36:34.286208458+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:34.287373624+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-instance-id=3da46200-5cf1-4f5b-9678-6b8ae2072fb1&target-nf-type=AMF |
2023-06-27T19:36:34.287936965+09:00 [INFO][SMF][Consumer] SendNFDiscoveryServingAMF ok
2023-06-27T19:36:34.288190995+09:00 [INFO][SMF][CTX] Allocated UE IP address: 10.60.0.1
2023-06-27T19:36:34.288339564+09:00 [INFO][SMF][CTX] Selected UPF: UPF
2023-06-27T19:36:34.288585767+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Allocated PDUAdress[10.60.0.1]
2023-06-27T19:36:34.288972082+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:34.290215137+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=SMF&target-nf-type=PCF |
2023-06-27T19:36:34.291768638+09:00 [INFO][PCF][SMpolicy] Handle CreateSmPolicy
2023-06-27T19:36:34.292249138+09:00 [INFO][UDR][DataRepo] Handle PolicyDataUesUeIdSmDataGet
2023-06-27T19:36:34.293210042+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/policy-data/ues/imsi-001010000000000/sm-data?dnn=internet&snssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |
2023-06-27T19:36:34.295325540+09:00 [INFO][UDR][DataRepo] Handle ApplicationDataInfluenceDataGet
2023-06-27T19:36:34.295802047+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v1/application-data/influenceData?dnns=internet&internal-Group-Ids=&snssais=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&supis=imsi-001010000000000 |
2023-06-27T19:36:34.296162714+09:00 [INFO][PCF][SMpolicy] Matched [0] trafficInfluDatas from UDR
2023-06-27T19:36:34.297016327+09:00 [INFO][UDR][DataRepo] Handle ApplicationDataInfluenceDataSubsToNotifyPost
2023-06-27T19:36:34.297217350+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v1/application-data/influenceData/subs-to-notify |
2023-06-27T19:36:34.297991935+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2023-06-27T19:36:34.298407712+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=BSF |
2023-06-27T19:36:34.299431876+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-smpolicycontrol/v1/sm-policies |
2023-06-27T19:36:34.300540380+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Has no pre-config route
2023-06-27T19:36:34.300658964+09:00 [WARN][SMF][PduSess] No Create URR
2023-06-27T19:36:34.300787409+09:00 [INFO][SMF][GIN] | 201 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts |
2023-06-27T19:36:34.300967248+09:00 [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2023-06-27T19:36:34.302054326+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] create smContext[pduSessionID: 1] Success
2023-06-27T19:36:34.316336760+09:00 [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2023-06-27T19:36:34.318341027+09:00 [INFO][AMF][Producer] Handle N1N2 Message Transfer Request
2023-06-27T19:36:34.318420455+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:32966] Send PDU Session Resource Setup Request
2023-06-27T19:36:34.319139052+09:00 [INFO][AMF][GIN] | 200 |       127.0.0.1 | POST    | /namf-comm/v1/ue-contexts/imsi-001010000000000/n1-n2-messages |
2023-06-27T19:36:34.321743856+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:32966] Handle PDUSessionResourceSetupResponse
2023-06-27T19:36:34.321978947+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:32966] Handle PDUSessionResourceSetupResponse (RAN UE NGAP ID: 1)
2023-06-27T19:36:34.322905362+09:00 [INFO][SMF][PduSess] Receive Update SM Context Request
2023-06-27T19:36:34.337129808+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Accepted Response from AN UPF
2023-06-27T19:36:34.337425210+09:00 [INFO][SMF][GIN] | 200 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts/urn:uuid:8224cb4c-729c-4ec5-9d76-566d85c33d64/modify |
```
The PDU session establishment status of VPP-UPF is as follows.
```
vpp# show upf session 
CP F-SEID: 0x0000000000000001 (1) @ 192.168.14.141
UP F-SEID: 0x0000000000000001 (1) @ 192.168.14.151
  PFCP Association: 0
  TEID assignment per choose ID
PDR: 1 @ 0x7f806b1ada20
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
  QER Ids: [2,1] @ 0x7f806b022850
PDR: 2 @ 0x7f806b1adaa0
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
  QER Ids: [2,1] @ 0x7f806b0304b0
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
[2023-06-27 19:36:34.356] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
5: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.60.0.1/32 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::4ed2:65ec:d295:6fd/64 scope link stable-privacy 
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
PING google.com (142.251.42.142) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 142.251.42.142: icmp_seq=2 ttl=59 time=17.7 ms
64 bytes from 142.251.42.142: icmp_seq=3 ttl=59 time=17.1 ms
64 bytes from 142.251.42.142: icmp_seq=4 ttl=59 time=16.4 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i enp0s9 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), snapshot length 262144 bytes
19:40:54.585595 IP 10.60.0.1 > 142.251.42.142: ICMP echo request, id 2, seq 2, length 64
19:40:54.602480 IP 142.251.42.142 > 10.60.0.1: ICMP echo reply, id 2, seq 2, length 64
19:40:55.586590 IP 10.60.0.1 > 142.251.42.142: ICMP echo request, id 2, seq 3, length 64
19:40:55.602818 IP 142.251.42.142 > 10.60.0.1: ICMP echo reply, id 2, seq 3, length 64
19:40:56.587732 IP 10.60.0.1 > 142.251.42.142: ICMP echo request, id 2, seq 4, length 64
19:40:56.603237 IP 142.251.42.142 > 10.60.0.1: ICMP echo reply, id 2, seq 4, length 64
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
19:42:16.567357 IP 10.60.0.1.34037 > 142.251.42.142.80: Flags [S], seq 1292625217, win 65280, options [mss 1360,sackOK,TS val 3445267733 ecr 0,nop,wscale 7], length 0
19:42:16.592984 IP 142.251.42.142.80 > 10.60.0.1.34037: Flags [S.], seq 2112001, ack 1292625218, win 65535, options [mss 1460], length 0
19:42:16.593631 IP 10.60.0.1.34037 > 142.251.42.142.80: Flags [.], ack 1, win 65280, length 0
19:42:16.593715 IP 10.60.0.1.34037 > 142.251.42.142.80: Flags [P.], seq 1:75, ack 1, win 65280, length 74: HTTP: GET / HTTP/1.1
19:42:16.593798 IP 142.251.42.142.80 > 10.60.0.1.34037: Flags [.], ack 75, win 65535, length 0
19:42:16.653039 IP 142.251.42.142.80 > 10.60.0.1.34037: Flags [P.], seq 1:774, ack 75, win 65535, length 773: HTTP: HTTP/1.1 301 Moved Permanently
19:42:16.654627 IP 10.60.0.1.34037 > 142.251.42.142.80: Flags [.], ack 774, win 64507, length 0
19:42:16.656107 IP 10.60.0.1.34037 > 142.251.42.142.80: Flags [F.], seq 75, ack 774, win 64507, length 0
19:42:16.656208 IP 142.251.42.142.80 > 10.60.0.1.34037: Flags [.], ack 76, win 65535, length 0
19:42:16.672519 IP 142.251.42.142.80 > 10.60.0.1.34037: Flags [F.], seq 774, ack 76, win 65535, length 0
19:42:16.673090 IP 10.60.0.1.34037 > 142.251.42.142.80: Flags [.], ack 775, win 64507, length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using VPP-UPF with DPDK.

---

Now you could work free5GC with VPP-UPF.
I would like to thank the excellent developers and all the contributors of free5GC, OpenAir CN 5G for UPF, UPG-VPP and DPDK.

<h2 id="changelog">Changelog (summary)</h2>

- [2023.06.27] Updated to free5GC v3.3.0.
- [2023.06.15] Initial release.
