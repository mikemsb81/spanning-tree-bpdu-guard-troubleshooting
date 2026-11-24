# Spanning Tree PortFast and BPDU Guard Troubleshooting

Cisco Packet Tracer lab by Michael Bessard.

## Overview

Configured PortFast and BPDU Guard on a VLAN10 endpoint port, verified normal PC connectivity, and connected a temporary switch to trigger BPDU Guard. Used switch logs and interface status to identify the resulting err-disabled state. Removed the switch, manually recovered the port, and verified restored forwarding and gateway connectivity with the protection retained.

**Lab file:** [spanning-tree-bpdu-guard-troubleshooting.pkt](spanning-tree-bpdu-guard-troubleshooting.pkt)

## Topology

This lab builds on an existing two-switch network with router-on-a-stick routing, DHCP, DHCP snooping, and port security on a different access port. The new configuration was applied to SW2 Fa0/3.

| Connection | Role |
|---|---|
| R1 G0/0 to SW1 G0/2 | Existing trunk; R1 provides VLAN gateways and DHCP |
| SW1 G0/1 to SW2 G0/1 | Existing trunk carrying VLANs 10 and 20 |
| PC5 FastEthernet0 to SW2 Fa0/3 | Normal endpoint connection for baseline and recovery |
| SW3 Fa0/1 to SW2 Fa0/3 | Temporary switch connection used to trigger BPDU Guard |

PC5 and SW3 were connected to Fa0/3 at different times. SW3 had only one connection to the network during the test. No redundant loop was created.

| Device | Address | Default gateway |
|---|---|---|
| PC5 | 192.168.10.22/24, obtained through DHCP | 192.168.10.1 |
| R1 VLAN10 gateway | 192.168.10.1/24 | N/A |

## Concepts

- **BPDU:** Bridge Protocol Data Unit, a message used by spanning tree to exchange bridge and path information and maintain a loop-free Layer 2 topology.
- **PortFast:** Lets an endpoint port begin forwarding without the usual spanning-tree listening and learning delay. It does not disable spanning tree.
- **BPDU Guard:** Disables a protected port when a BPDU arrives, enforcing the expectation that the port connects to an endpoint rather than another bridge.
- **Err-disabled:** A protective interface state that requires the cause to be addressed before recovery.

BPDU Guard reacts to a received BPDU; a loop or a superior root advertisement is not required to trigger it.

## Configuration

On SW2:

```text
enable
configure terminal
interface fastEthernet0/3
 description PORTFAST-BPDU-GUARD-LAB
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
end
```

Verified the configured interface:

```text
show running-config | section interface FastEthernet0/3
```

The output confirmed access VLAN10, access mode, PortFast, and interface-level BPDU Guard.

## Baseline verification

Connected PC5 to SW2 Fa0/3 using a copper straight-through cable and selected DHCP under Desktop > IP Configuration.

On PC5:

```text
ipconfig
ping 192.168.10.1
```

PC5 showed 192.168.10.22 with mask 255.255.255.0 and gateway 192.168.10.1. The captured successful ping returned four replies with 0% loss. An earlier failed ping summary is also visible in the screenshot; its cause was not established.

On SW2:

```text
show spanning-tree interface fastEthernet0/3 detail
```

Fa0/3 was designated forwarding for VLAN0010, and the output explicitly stated that the port was in PortFast mode. PortFast transition timing was not measured.

## Controlled trigger

Disconnected PC5 and connected a temporary default-configured Cisco 2960, labeled SW3, from Fa0/1 to SW2 Fa0/3 using a copper crossover cable.

SW2 logged the following diagnostic messages:

```text
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port FastEthernet0/3 with BPDU Guard enabled. Disabling port.
%PM-4-ERR_DISABLE: bpduguard error detected on 0/3, putting 0/3 in err-disable state
```

Checked the interface:

```text
show interfaces fastEthernet0/3
show interfaces status
```

The detailed output showed Fa0/3 down with line protocol down and the reason **err-disabled**. The interface status table also showed Fa0/3 as err-disabled in VLAN10.

### Separate VLAN mismatch warning

A CDP native VLAN mismatch warning appeared because SW2 Fa0/3 used VLAN10 while the temporary switch's default Fa0/1 used VLAN1. This warning was separate from the explicit BPDU Guard shutdown reason. The test did not establish a usable inter-switch data path or correct the mismatch; SW3 was removed after the trigger was captured.

## Recovery

Disconnected SW3 and reconnected PC5 to SW2 Fa0/3. Recovered the port on SW2:

```text
configure terminal
interface fastEthernet0/3
 shutdown
 no shutdown
end
```

PortFast and BPDU Guard were left configured. Removing the BPDU source before resetting the port avoided an immediate repeat trigger.

On PC5:

```text
ipconfig
ping 192.168.10.1
```

PC5 showed 192.168.10.22/24, gateway 192.168.10.1, and received four replies with 0% loss.

On SW2:

```text
show interfaces status
show spanning-tree interface fastEthernet0/3 detail
```

Fa0/3 showed **connected** in VLAN10 and **designated forwarding**, with PortFast mode explicitly reported again.

Saved SW2's configuration:

```text
copy running-config startup-config
```

Saved the Packet Tracer project with PC5 connected and SW3 disconnected.

## Observed results

| Stage | Check | Result |
|---|---|---|
| Configuration | Fa0/3 interface configuration | Access VLAN10, PortFast and BPDU Guard enabled |
| Baseline | PC5 addressing | 192.168.10.22/24; gateway 192.168.10.1 |
| Baseline | PC5 gateway ping | Successful captured run: 4/4 replies |
| Baseline | Spanning-tree detail | Designated forwarding, PortFast mode |
| Trigger | Switch logs | Received BPDU; BPDU Guard disabled Fa0/3 |
| Trigger | Interface detail and status table | Err-disabled |
| Recovery | Interface status | Connected in VLAN10 |
| Recovery | Spanning-tree detail | Designated forwarding, PortFast mode |
| Recovery | PC5 gateway ping | 4/4 replies, 0% loss |

## Screenshot evidence

| Screenshot | Evidence |
|---|---|
| [01-portfast-bpduguard-config.png](01-portfast-bpduguard-config.png) | Fa0/3 access VLAN and protection configuration |
| [02-pc5-baseline-connectivity.png](02-pc5-baseline-connectivity.png) | Baseline addressing and successful gateway ping |
| [03-portfast-forwarding-baseline.png](03-portfast-forwarding-baseline.png) | Baseline forwarding and PortFast mode |
| [04-bpduguard-trigger-logs.png](04-bpduguard-trigger-logs.png) | BPDU receipt, shutdown reason, and separate CDP warning |
| [05-interface-err-disabled.png](05-interface-err-disabled.png) | Detailed err-disabled interface state |
| [06-port-status-err-disabled.png](06-port-status-err-disabled.png) | Interface status table during the trigger |
| [07-pc5-connectivity-restored.png](07-pc5-connectivity-restored.png) | PC5 addressing and gateway ping after recovery |
| [08-portfast-forwarding-restored.png](08-portfast-forwarding-restored.png) | Connected status and restored PortFast forwarding |

## Lessons learned and scope

- Use PortFast for endpoint-facing ports where immediate forwarding is appropriate.
- BPDU Guard can enforce the endpoint-only expectation by disabling a port that receives spanning-tree messages.
- Read the explicit shutdown reason in logs rather than attributing every down port to cabling or a nearby warning.
- Remove the trigger before manually recovering an err-disabled interface.
- Preserve the protection while restoring authorized endpoint service.
- Verify recovery using interface state, spanning-tree state, and endpoint connectivity.

This lab tested one BPDU Guard trigger and manual recovery. It did not test an actual loop, root bridge manipulation, Root Guard, Loop Guard, BPDU filtering, automatic err-disable recovery, or restart persistence. Cross-VLAN connectivity was not retested in this exercise.
