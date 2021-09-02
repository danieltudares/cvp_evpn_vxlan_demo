
* [Introduction](#introduction)
  - [Requirements](#requirements)
  - [Environment setup](#environment-setup)
  - [Lab deployment](#lab-deployment)
  - [Lab validation](#lab-validation)
  - [Devices snapshot](#devices-snapshot)
  - [Useful Links](#useful-links)

## **Introduction:**

This lab will demonstrate the deployment of an EVPN/VXLAN Active-Active (A-A) multihoming Spine/Leaf using CloudVision and Ansible AVD.

The topology created on EVE-NG is as shown below: 

<img src="data/images/topology.png" width="800">

Details on the fabric underlay and overlay infrastructure is located [here](./inventory/documentation/fabric/DC_FABRIC-documentation.md)

---

## **Requirements:**

For this lab environment the following tools and Arista images are used: 

### *Arista:*
- Arista vEOS-lab 4.25.4M
- Arista CloudVision Portal 2021.1.1

### *EVE-NG:*
- EVE-NG Professional Edition 4.0.1-56

### *Python:*
- Python 3.9.6 (min supported 3.6.8)

### *Ansible:*
- Ansible 2.10.13 (min supported 2.10.7)

### *Ansible roles:*
- arista.avd
- arista.cvp

### *Additional python libraries:*
- netaddr==0.7.19
- Jinja2==2.11.3
- treelib==1.5.5
- cvprac==1.0.5
- paramiko==2.7.1
- jsonschema==3.2.0
- requests==2.25.1
- PyYAML==5.4.1
- md-toc==7.1.0

### *Workstation:* 
- Any device Mac/Win/Linux or VM with network reachability to the CloudVision server and EOS management interfaces. 

### *DHCP Server:*
- Any DHCP server that can provide IP reservation based on MAC address. The DHCP server needs to be located on the same management network as the vEOS devices for the ZTP process. For this lab we will run ISC-DHCP inside de CloudVision server (This is only for lab purposes and not recommended for prod environments).

---

## **Environment Setup:**

To run this lab, you can either clone this repo in your local environment and use ***/virtual_lab/EVE_NG/labs/EVPN_AA_Multihoming*** as your root folder or for a cleaner approach you can create a new folder in your local environment (or docker AVD container - see next section). If you choose the later, this is the folder structure and files that you will need to create and copy to your environment: 

```
|--EVPN_AA_Multihoming
   |--ansible.cfg
   |--requirements.txt
   |--inventory
      |--inventory.yml
      |--group_vars
        |--CVP.yml
        |--DC.yml
        |--DC_FABRIC.yml
        |--DC_SPINES.yml
        |--DC_L3LEAFS.yml
        |--DC_SERVERS.yml
        |--DC_TENANTS_NETWORKS.yml
    |--playbooks
       |--cvp
          |--playbook_cvp_ztp_config.yml
          |--playbook_deploy_cvp.yml
          |--playbook_cvp_validate_states.yml
          |--playbook_cvp_device_snapshot.yml
```

Two methods can be used to get Ansible up and running: running a Python virtual envionrment or using the AVD docker container. 

### AVD Docker container:
The AVD docker is a container with all requirements pre-installed. It is a quick way to start working on your lab without worrying about setting up your environment. 
Info about getting the AVD docker container up and running [here](https://avd.sh/en/releases-v3.x.x/docs/installation/setup-environment.html#use-docker-as-avd-shell)

### Python virtual environment:
1. From the project root folder (*/EVPN_AA_Multihoming*) create a new python environment and activate it:

```bash
$ pwd
/home/user/EVPN_AA_Multihoming

$ python3 -m venv .venv
$ source .venv/bin/activate
```

2. Update pip and install requirements:

```bash
$ python3 -m pip install --upgrade pip
$ pip install -r requirements.txt
```

3. Install ansible AVD and CVP collections:

```bash
$ ansible-galaxy collection install arista.avd
$ ansible-galaxy collection install arista.cvp
```

4. Setup your EVE-NG topology as in the topology image. Alternatively, you can download the lab file and import to your EVE-NG server. [Arista_CVP_EVPN_AA_multihoming_lab.zip](./data/eve_lab_topology/Arista_CVP_EVPN_AA_multihoming_lab.zip) 

5. Start your CloudVision Portal (CVP) node on EVE-NG and do the initial setup (not covered in this lab).

6. Start your DHCP server and make sure to create static reservations using the management interface mac address of each node. Alternatively if you want to run the DHCP server in your CVP server, update the ***/inventory/group_vars/CVP.yml*** file with the proper mac addresses and run the ***playbook_cvp_ztp_config.yml*** playbook. 
Note: Step 5 must be completed to run this task. Also, you can modify the IPs to fit your networks requirements, if you do change the IP, remember to update the information on the *inventory.yml* file. 

```bash
$ nano /inventory/group_vars/CVP.yml
---
ztp:
  default:
    registration: 'https://172.30.30.252/ztp/bootstrap'
    gateway: 172.30.30.1
    nameservers:
      - '172.30.30.6'
      - '1.1.1.1'
  general:
    subnets:
      - network: 172.30.30.0
        netmask: 255.255.255.0
        gateway: 172.30.30.1
        nameservers:
          - '172.30.30.6'
          - '1.1.1.1'
        start: 172.30.30.200
        end: 172.30.30.210
        lease_time: 300
  clients:
  # AVD/CVP Integration
    - name: DC-SPINE1
      mac: '50:01:11:11:00:00'
      ip4: 172.30.30.211
    - name: DC-SPINE2
      mac: '50:01:22:22:00:00'
      ip4: 172.30.30.212
    - name: DC-LEAF1
      mac: '50:01:33:33:00:00'
      ip4: 172.30.30.213
    - name: DC-LEAF2
      mac: '50:01:44:44:00:00'
      ip4: 172.30.30.214
    - name: DC-LEAF3
      mac: '50:01:55:55:00:00'
      ip4: 172.30.30.215
    - name: DC-LEAF4
      mac: '50:01:66:66:00:00'
      ip4: 172.30.30.216

# After done editing the file, run the ZTP playbook. 
$ ansible-playbook playbook/cvp/playbook_cvp_ztp_config.yml
```
7. The server nodes connected to the leafs are not managed by Ansible and needs to be configured manually according to the topology. For simplicity, this lab uses Arista vEOS devices configured with port-channels. The running config for these devices can be located under ***/data/server_configs/serverXX.conf***. 

At this point, your environment should be ready to deploy and you can move to the next section. 

---

## **Lab Deployment:**

1. Turn on all the spine and leaf nodes. On the factory default mode they will boot on ZTP mode and will search for a DHCP server. Your DHCP server should assign an IP and direct the nodes to the CVP server. After finishing the boot process, all nodes should show up on your CVP server under the ***Device > Inventory*** tab.

<img src="data/images/cvp_devices.png" width="800">

2. Make sure to review the **inventory.yml** and files under **group_vars/**, you can update the files to fit your needs or leave the values as they are. Once you are confortable with the values, run the playbook to build all documentation and intended configuration files: 

```bash
$ ansible-playbook playbooks/cvp/playbook_cvp_deploy_cvp.yml --tags build
```

When passing the **"build"** tag, this playbook will create new folders and generate all the documentation and intended config files that will eventually be pushed to CVP as configlets. At this point, your project structure should have the following new folders:

```
|--EVPN_AA_Multihoming
   |--inventory
      |--config_backup
      |--documentation
        |--devices
        |--fabric
      |--intended
        |--configs
        |--structured_configs
      |--reports
      |--snapshots
```

The final spine/leaf configuration file is created under **/inventory/intended/configs/DC-XXXX.cfg**, you can review each config file and make sure the intended config is as expected. 

```cli
$ cat /inventory/intended/configs/DC-LEAF1.cfg

!RANCID-CONTENT-TYPE: arista
!
daemon TerminAttr
   exec /usr/bin/TerminAttr -ingestgrpcurl=172.30.30.252:9910 -cvcompression=gzip -ingestauth=key,arista -smashexcludes=ale,flexCounter,hardware,kni,pulse,strata -ingestexclude=/Sysdb/cell/1/agent,/Sysdb/cell/2/agent -ingestvrf=mgmt -taillogs
   no shutdown
!
vlan internal order ascending range 1006 1199
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname DC-LEAF1
ip name-server vrf mgmt 1.1.1.1
ip name-server vrf mgmt 172.30.30.6
!
dns domain homelab.io
!
ntp local-interface vrf mgmt Management1
ntp server vrf mgmt time.google.com iburst prefer
!
spanning-tree mode mstp
spanning-tree mst 0 priority 4096
!
no aaa root
no enable password
!
clock timezone America/Toronto
!
vlan 10
   name Tenant_blue_compute
!
vlan 20
   name Tenant_green_compute
!
vlan 30
   name Tenant_green_storage
!
vrf instance mgmt
!
vrf instance Tenant_blue_vrf
!
vrf instance Tenant_green_vrf
!
... Full config file shortened for brevity ...
```

3. To create the configlets and push to the CVP server you need to run the previous playbook passing the **provision** tag.

```bash
$ ansible-playbook playbooks/cvp/playbook_cvp_deploy_cvp.yml --tags provision
```

This playbook has 3 main functions:

- Read the **inventory.yml** file and build the container structure

<img src="data/images/cvp_containers.png" width="800">

- Create configlets based on the intended config files

<img src="data/images/cvp_configlets.png" width="800">

- Create the tasks on CVP to push the configlets to each node 

<img src="data/images/cvp_tasks.png" width="800">

4. Final step is to go on CVP under **Provisioning > Tasks**, select all tasks created by ansible and execute all tasks. 

---

## **Lab validation:**

AVD offers a role to validate all operational states of the Arista EOS devices. It connects to the devices and collects the actual states and generates a report (markdown and CSV) with the results. After finishing the lab deployment, use the **playbook_cvp_validate_states.yml** to run the automated tests.

```bash
$ ansible-playbook playbooks/cvp/playbook_cvp_validate_states.yml
```

After running the playbook, the reports can be located under the **/inventory/reports/** folder.

```bash
$ cat /inventory/reports/DC_FABRIC-state.md


# Validate State Report

**Table of Contents:**

- [Validate State Report](validate-state-report)
  - [Test Results Summary](#test-results-summary)
  - [Failed Test Results Summary](#failed-test-results-summary)
  - [All Test Results](#all-test-results)

## Test Results Summary

### Summary Totals

| Total Tests | Total Tests Passed | Total Tests Failed |
| ----------- | ------------------ | ------------------ |
| 215 | 215 | 0 |

### Summary Totals Devices Under Tests

| DUT | Total Tests | Tests Passed | Tests Failed | Categories Failed |
| --- | ----------- | ------------ | ------------ | ----------------- |
| DC-LEAF1 |  38 | 38 | 0 | - |
| DC-LEAF2 |  40 | 40 | 0 | - |
| DC-LEAF3 |  41 | 41 | 0 | - |
| DC-LEAF4 |  34 | 34 | 0 | - |
| DC-SPINE1 |  31 | 31 | 0 | - |
| DC-SPINE2 |  31 | 31 | 0 | - |

### Summary Totals Per Category

| Test Category | Total Tests | Tests Passed | Tests Failed |
| ------------- | ----------- | ------------ | ------------ |
| NTP |  6 | 6 | 0 |
| Interface State |  75 | 75 | 0 |
| LLDP Topology |  16 | 16 | 0 |
| IP Reachability |  16 | 16 | 0 |
| BGP |  38 | 38 | 0 |
| Routing Table |  40 | 40 | 0 |
| Loopback0 Reachability |  24 | 24 | 0 |

## Failed Test Results Summary

| Test ID | Node | Test Category | Test Description | Test | Test Result | Failure Reason |
| ------- | ---- | ------------- | ---------------- | ---- | ----------- | -------------- |

... ... Rest of file shortened for brevity ...
```

---

## **Devices Snapshot:**

Another feature of AVD is automated device snapshot. The snapshot playbook collects the output from different "show" commands and generate a report with the information. Use the **playbook_cvp_device_snapshot.yml** to run and collect the devices snapshot.

```bash
$ ansible-playbook playbooks/cvp/playbook_cvp_device_snapshot.yml
```
After running the playbook, the snapshot reports can be located under the **/inventory/snapshots/** folder.

```bash

$ cat inventory/snapshots/DC-LEAF1/report.md

# DC-LEAF1 Commands Output

## Table of Contents

- [show lldp neighbors](#show-lldp-neighbors)
- [show ip interface brief](#show-ip-interface-brief)
- [show interfaces description](#show-interfaces-description)
- [show version](#show-version)
- [show running-config](#show-running-config)
## show interfaces description


Interface                      Status         Protocol           Description
Et1                            up             up                 P2P_LINK_TO_DC-SPINE1_Ethernet1
Et2                            up             up                 P2P_LINK_TO_DC-SPINE2_Ethernet1
Et3                            up             up                 
Et4                            up             up                 
Et5                            up             up                 server01_E1
Et6                            up             up                 server02_E1
Et7                            up             up                 server03_E1
Et8                            up             up                 
Et9                            up             up                 
Et10                           up             up                 
Et11                           up             up                 
Lo0                            up             up                 EVPN_Overlay_Peering
Lo1                            up             up                 VTEP_VXLAN_Tunnel_Source
Lo100                          up             up                 Tenant_blue_vrf_VTEP_DIAGNOSTICS
Lo200                          up             up                 Tenant_green_vrf_VTEP_DIAGNOSTICS
Ma1                            up             up                 oob_management
Po5                            up             up                 server01_PortChannel
Po6                            up             up                 server02_PortChannel
Po7                            up             up                 server03_PortChannel
Vl10                           up             up                 Tenant_blue_compute
Vl20                           up             up                 Tenant_green_compute
Vl30                           up             up                 Tenant_green_storage
Vl1197                         up             up                 
Vl1198                         up             up                 
Vl1199                         up             up                 
Vx1                            up             up

## show ip interface brief


Address 
Interface       IP Address           Status     Protocol         MTU    Owner   
--------------- -------------------- ---------- ------------ ---------- ------- 
Ethernet1       10.10.100.1/31       up         up              1500            
Ethernet2       10.10.100.3/31       up         up              1500            
Loopback0       192.168.100.3/32     up         up             65535            
Loopback1       192.168.101.3/32     up         up             65535            
Loopback100     10.255.1.3/32        up         up             65535            
Loopback200     10.255.2.3/32        up         up             65535            
Management1     172.30.30.213/24     up         up              1500            
Vlan10          10.10.10.1/24        up         up              1500            
Vlan20          10.10.20.1/24        up         up              1500            
Vlan30          10.10.30.1/24        up         up              1500            
Vlan1197        unassigned           up         up              9164            
Vlan1198        unassigned           up         up              9164            
Vlan1199        unassigned           up         up              9164

## show lldp neighbors


Last table change time   : 0:27:31 ago
Number of table inserts  : 6
Number of table deletes  : 0
Number of table drops    : 0
Number of table age-outs : 0

Port          Neighbor Device ID         Neighbor Port ID    TTL 
---------- -------------------------- ---------------------- --- 
Et1           DC-SPINE1.homelab.io       Ethernet1           120 
Et2           DC-SPINE2.homelab.io       Ethernet1           120 
Et5           server01                   Ethernet1           120 
Et6           server02                   Ethernet1           120 
Et7           server03                   Ethernet1           120 
Ma1           mgmt-sw                    Ethernet9           120


... ... Rest of file shortened for brevity ...
```
