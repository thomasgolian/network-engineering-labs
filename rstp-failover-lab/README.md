# Rapid Spanning Tree Convergence & Failover (RSTP Lab)

Lab was built using VMware Workstation with Cisco Modeling Labs v2.8.1 

All switches routers in this lab are running IOS XE images virtualized or containered

<br>

![CML](images/CML.jpg)

<br>

# Overview

- This lab demonstrates the implementation and behavior of Rapid Spanning Tree Protocol (RSTP) in a Layer 2 switched network.

- The objective is to analyze STP topology, validate loop prevention, and observe rapid failover during core and distribution layer failures.

- RSTP provides significantly faster convergence than traditional STP by leveraging alternate ports and rapid state transitions.


## Objectives

- Configure RSTP across multiple switches

- Analyze the root bridge election process

- Simulate core switch failure and measure RSTP convergence

- Observe port roles (Root, Designated, Alternate)

- Simulate distribution-layer link failure and measure failover response

- Validate rapid convergence behavior

- Introduce a rogue switch on an access port configured with PortFast and BPDU Guard, and observe the outcome

- Ensure a loop-free topology under failure conditions, while maintaining Layer 3 connectivity to and from the LAN (with R1 acting as the WAN/ISP)

<br>

## Topology 

<br>

![Topology](images/topologyfinal.jpg)

<br>

## Topology Description:

- Multiple Layer 2 switches interconnected with redundant links to provide path diversity

- A primary root bridge is elected, with a secondary core switch positioned as the backup root

- Redundant Layer 2 paths are available to support failover and rapid convergence scenarios

<br>

# VLAN & Interface Configuration

- Access nodes (Users) are using VLAN 10. All trunks are configured to allow VLAN 10 over trunk links.

<br>

- Access switches SW6,7,8 have extra interfaces installed and configured as access ports running port-fast and BPDU guard.

<br>

<img src="images/VLAN10-interfaces-verify.png" width="600">

<br>

Link between core SW1 and SW2 is layer 2 link for the purpose of more RSTP options. 

<br>

# Configurations:

<br>

Full configurations are available in the configs/ directory

<br>

Initial IOS XE configurations I entered for all network nodes:

```
enable secret cisco
hostname {}
no ip domain lookup

line console 0
logging synchronous
exec-timeout 0 0
password cisco
login

line vty 0 4
logging synchronous
exec-timeout 15 0
password cisco
login
transport input ssh

copy running-config startup-config 
```

<br>

Alpine Linux Desktop to test end-to-end connectivity and configured with:
<br>sudo ifconfig eth0 10.1.10.19 netmask 255.255.255.0

<br>


SW1 & SW2 SVI
```
interface vlan 10
ip address 10.1.10.2 255.255.255.0
no shutdown

interface vlan 10
ip address 10.1.10.3 255.255.255.0
no shutdown
```
<br>

SW1 default route out of the network towards R1 (ISP) is:
```
ip route 0.0.0.0 0.0.0.0 10.1.1.1
```

SW2 default route out of the network towards R1 (ISP) is:
```
ip route 0.0.0.0 0.0.0.0 10.1.2.1
```

<br>

# Key Configuration Elements:

SW1 was manually configured as the root bridge by lowering its spanning-tree bridge priority (primary). This can also be achieved by setting the bridge priority to a lower value in increments of 4096.

RSTP enabled `spanning-tree mode rapid-pvst`

Root bridge priority on SW1 & SW2
```
spanning-tree vlan 10 root primary
spanning-tree vlan 10 root secondary
```
<br>

<img src="images/SW1-root-verify.png" width="600">

<br>

<img src="images/SW2-secondary-verify.png" width="600">

<br>

## Verify: show spanning-tree vlan 10

```
SW1 - root primary
Et0/2               Desg FWD 100       128.3    P2p 
Et0/3               Desg FWD 100       128.4    P2p 
Et1/0               Desg FWD 100       128.5    P2p 

SW2 - root secondary
Et0/2               Altn BLK 100       128.3    P2p 
Et0/3               Root FWD 100       128.4    P2p 
Et1/0               Altn BLK 100       128.5    P2p
```

<br>

## Port-Fast (applied to edge ports) SW6, SW7, SW8
```
interface range {interfaces}
switchport mode access
switchport access vlan 10
spanning-tree portfast
spanning-tree bpduguard enable
```

<br>

BPDU Guard (implemented)

Verified VLAN 10 SVIs were up, up, on SW1 and SW2
<br>show ip interface status

## Commands on trunk links between switches:
```
interface range {multiple interfaces}
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan add 10
```
Pinged local SVI to ensure TCP/IP stack working

Pinged opposite core switch to ensure core-to-core link working

R1 pinged SW1 and SW3 cores successfully, - ISP/WAN links working

<br>

<img src="images/SW1-root.jpg" width="600">

<br>

***************************************************************************************
<br>

# Scenario 1) 

RSTP root primary core SW1 fails, all interfaces shutdown. Simulating a failure.
 
<br>

<img src="images/failover-to-root-secondary2.jpg" width="600">

<br>

## We can use these commands to verify:

Verify Spanning Tree Status: 
```
show spanning-tree
```
SW1 initially served as the root bridge. After its failure, SW3 (distribution) detects the loss of the root and reconverges RSTP for VLAN 10, electing SW2 as the new root bridge. As the secondary (backup) root, SW2 assumes the role based on its lower bridge priority.

<br>

For fun we can use this command to see all the BPDUs in real time on SW3 before the failure:
```
debug spanning-tree bpdu
```
<br>

![BPDUs](images/debug-bpdu-sw3.jpg)

<br>

We can see the conversation and reconvergence of the switches on the CLI when we enable debugging: 
<br>debug spanning-tree events

<img src="images/debug-rstp-events.jpg" width="600">

<br>

- Verify Root Bridge: 
<br>show spanning-tree root

- SW2 is now the root bridge for VLAN 10, as it has the lowest bridge priority among active switches. With SW1 down and no longer sending BPDUs, SW2 assumes the root role.

- When SW1 is restored, it will regain root status for VLAN 10 due to its root primary configuration, which sets a lower bridge priority than the other switches.

<br>

![SW2 Root](images/sw2-root-converge.jpg)

<br>

Failover testing was performed by shutting down all interfaces on SW1 root to simulate switch catastrophic failure. 

## Observed Behavior:

- SW2 secondary root now takes over as the VLAN 10 RSTP root core switch, changing all interfaces to designated / forwarding. 

- No switching loops were introduced

- Network connectivity was maintained

- We could see SW3 detect something was wrong after not receiving BPDUs from root SW1 - and logs enable us to see the process.

<br>

***************************************************************************************

<br>

# Scenario 2) 

- A downstream link failure was observed between distribution switch SW3 and access switch SW7, resulting in loss of connectivity on the trunk.

- Failover testing was performed by manually shutting down the primary link between SW3 and SW7.

<br>

<img src="images/SW3-link-failure.jpg" width="600">

<br>

![SW3 Verify Down](images/SW3-link-failure-verify.jpg)

<br>

- We can verify by checking the outgoing interface before and after the topology change:

- Before = SW7 took E0/0 outgoing interface as path to root
<br>After = SW7 takes E0/1 outgoing interface as path to root

Originally topology path will return once SW3 regains connectivity on it's downstream trunk to SW7. 

<br>

![SW7 Path Change](images/sw7-path.jpg)

<br>

<img src="images/RSTP-reconvergence2.jpg" width="600">

<br>

Observed Behavior:

- Alternate port transitioned to forwarding state

- Convergence occurred rapidly (sub-second to a few seconds)

- No switching loops were introduced

- Network connectivity was maintained

- Traffic took alternative path to SW5 in order to reach the root / core layer. 

- We can verify by using show spanning-tree commands and using debug logs to see the overhead conversation between switches. 

<br>

***************************************************************************************

# Scenario 3

RSTP topology is operating normally. PortFast and BPDU Guard were configured on SW6’s access ports to validate protection against unintended switch connections.

RSTP BPDU Guard Demonstration:

- A rogue switch was introduced on an access port of SW6 to trigger BPDU Guard and observe the resulting behavior.

<br>

<img src="images/SW6-rogueswitch.jpg" width="600">

<br>

Observed Behavior:

- With PortFast and BPDU Guard enabled on its access ports, SW6 detects BPDUs received on an access interface. As a result, BPDU Guard immediately places the interface into an err-disabled state.

<br>

![BPDU err-disable Verify](images/SW6-BPDUguard.jpg)

<br> 

Let's watch it again but we will first enable debugging on SW6 so we can view the logs as the rogue switch gets plugged in:
```
show spanning-tree events
show spanning-tree bpdu
```
Here we see a level 2 Critical log from Spanning-Tree - BPDU Guard is blocking int E1/0, placing interface into err-disabled:

<br> 

![BPDU Blocking](images/bpdu-guard-block.jpg)


- After the condition is cleared, the err-disabled interface must be manually recovered by issuing a shutdown followed by a no shutdown command.

- When a BPDU is received on a PortFast-enabled interface with BPDU Guard configured, the switch immediately places the port into an err-disabled state to prevent potential Layer 2 loops.

<br> 

***************************************************************************************

<br>

# Final Results

- Successful RSTP deployment

- Rapid convergence observed during topology change

- Loop-free network maintained under failure conditions

- Efficient utilization of redundant paths

- Distribution layer link failure - RSTP recovers.

- Rogue switch plugged into access layer switch - BPDU Guard successfully detects - places into err-disabled state.

<br>

# Key Takeaways

- A link failure at the distribution layer doesn't break the network. RSTP recovers

- Demonstrated the ability to design, test, and validate Layer 2 failover scenarios using RSTP

- Alternate ports enable near-instant failover

- Redundancy at the core and distribution level is crucial to maintain network availability 

- Proper Layer 2 design is critical for network stability and performance

- Watched BPDU Guard block a rogue switch in real time (virtual machine)

- Using debugging logs help us verify and understand each step of RSTP convergence

- Understanding STP more from building it from the ground up with real Cisco IOS XE nodes. 

<br>

*********************************************************************************************

![Topology](images/topologyfinal.jpg)

<br>



