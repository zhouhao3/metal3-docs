# Automatic network configuration

## Status

provisional

## Summary

This proposal is to expand the scope of Metal³ to include an API to
manage physical network devices.

## Motivation

Metal³ follows the paradigm of Kubernetes Native Infrastructure (KNI),
which is an approach to use Kubernetes to manage underlying infrastructure.
Managing the configuration of some physical network devices is closely
related to managing physical hosts.
As bare metal hosts are provisioned or later repurposed, there may be
corresponding physical network changes that must be made, such as
reconfiguring a ToR switch port. If the provisioning of the physical
host is managed through a Kubernetes API, it would be convenient to be
able to reconfigure related network devices using a similar API.

### Goals

* Define a Kubernetes API for configuring network switches.
* Automatically configure the network infrastructure for a host
  when adding it to a cluster.
* Do network configuration when deprovisioning the hosts.
* Design a network abstraction that can represent any of the target
  networking configuration, independently of the controller used.

### Non-Goals

* Implement a network controller which is aimed at being an integration
  with an existing controller only.
* Implement a solution for a specific underlying infrastructure.
* Implement a solution for link aggregation in the future.
* Implement a solution for lldp in the future.

## Proposal

This document proposes to add a new mechanic to automatically perform
physical network device configuration before provisioning a BareMetalHost.
Frist we added the NetworkConfiguration operator to implement the network
configuration of the device. Then we added some methods to CAPM3 to
implement together. When network configuration is required, CAPM3 will
process some content through additional methods, and then trigger the
NetworkConfiguration operator to perform network configuration.

### User Stories

#### Story 1

As a consumer of Metal³, when adding a machine resource to the cluster,
according to the network configuration of the cluster, the physical host
is connected to the corresponding logical layer-2 network segment.

#### Story 2

As a consumer of Metal³, when a machine is deleted from the cluster,
the corresponding network configuration is automatically deleted.

#### Story 3

As a user/admin a SmartNIC that is hosted in a BMH needs to be provisioned,
pre-configured, and then capable of in operation management.

Physical server(OS) -----> SmartNIC [virtual switch(OVS)
---> physical NIC] ←-- (LAN cable)-----> PortID

#### Story 4

As a user, when creating a network configuration and specifying link aggregation,
the related BMH and siwtch can be automatically configured.

#### Story 5

As a user, when creating a network configuration, it is possible to configure ACL
settings in a switch for each bare metal server according to its functionality.
The ACL setting:

* Direction: ingress or egress
* Protocol: SSH, HTTP, …all
* Action: allow, deny
* Port status: up, down (status of the switch port)

## Design Details

### Structure

In the networkConfiguration operator, we abstract the following roles:

#### Device

Abstract all devices in the cluster (such as Switch, SmartNic).

Each type of network device must have its own CRD.

##### Port

Indicates the specific port of the network device.

```go
// PortManager is used to control the `Port`
type PortManager interface {
    // ConfigurePort set the network configure to the port
    ConfigurePort(ctx context.Context, configuration interface{}) error
    // DeConfigurePort remove the network configure from the port
    DeConfigurePort(ctx context.Context) error
    // CheckConfigutation checks whether the configuration is
    // configured on the port
    CheckConfigutation(ctx context.Context) bool
}
```

Each type of `Port` must have its own CRD and implements corresponding
`PortManager` interface.

#### Configuration

Indicates the content of the specific configuration of each port.
Each type of `Configuration` must have its own CRD

#### Port Controller

Each `Device` must have its own `Port controller` to monitor the corresponding
`Port`, and connect to `Device` to change the configuration of the
corresponding port according to the `Port` change.Each Port controller must
implement the method of configuring `Device` specified by the `Device.spec.protocol`.

The following is the processing after `Port CR` changes:
* Port.spec.configurationRef changed from empty to non-empty
  * Find the corresponding `Device` CR according to `DeviceRef`, then verify that
    the configuration of the port is allowed according to the information in the
    device CR, and then log in to the corresponding device to perform the
    corresponding configuration operation.

* Port.spec.configurationRef changed from non-empty to empty
  * Find the corresponding `Device` CR according to `DeviceRef`, and then log in
    to the corresponding device to cancel the corresponding configuration.

* Port.spec.configurationRef content changes
  * Log in to the corresponding device and apply the latest configuration.

### Workflow

Create CR related to BMH's network:

* Administrator creates the corresponding `Device` CR according to the
  specific network device.
* Administrator creates a `BareMetalHost` CR and fills in the content in `bmh.spec.ports`
  according to the actual network connection.
* Administrator creates a corresponding `Port` CR for each port of the network device
  connected to the BMH network card.

Create Metal3Machine:

* User creates `Configuration` corresponding to the `Device`.
* User specifies the `networkConfiguration` field when creating `Metal3Machine`
  CR.
* CAPM3 filter the BMH by calling the filter() function. Then select a BMH
  according to the original method.
* Before provisioning, if the machine's network configuration field is not empty,
  CAPM3 calls the `configureNetwork()` method to fill networkConfiguration into
  the appropriate `spec.configurationRef`.
* Port controller starts to configure the network by call device.ConfigurePort().
* BMO detects that the Port CR corresponding to all its own network cards are
  configured and continues to provision BMH.

Remove Metal3Machine:

* User deletes the `Metal3Machine` CR.
* CAPM3 calls the `deconfigureNetwork()` to find all the `Port` CR corresponding
  to the BMO network cards, and clear their `spec.configurationRef`.
* Port  controller starts to deconfigure the network by call device.DeConfigurePort().
* BMO stage detects that the `Port` CR corresponding to all its own network
  cards are cleared after the configuration is completed and continues to
  deprovision BMH.

### Changes to Current API

#### Metal3Machine CRD

```yaml
spec:
  # Specify the network configuration that metal3Machine needs to use
  networkConfiguration:
  # no smartNic
  - ConfigurationRef:
      name: nc1
      kind: SwitchPortConfiguration
      namespace: default
    # Network card performance required for this network configuration
    nicHint:
      name: eth0
      smartNIC: false
  # with smartNic
  - ConfigurationRef:
      name: nc1
      kind: SmartNicConfiguration
      namespace:  default
    nicHint:
      name: eth1
      smartNIC: true
```

#### BareMetalHost

Add the `ports` field to `.spec` to indicate the connection information
and performance of all network cards in the `BareMetalHost`.

``` yaml
spec:
  ports:
    - mac: 00:00:00:00:00:00
      portRef:
        name: port0
        kind: SwitchPort
        namespace: default
   - mac:
   ......
```

#### CAPM3

Add the following methods in `metal3machine_manager.go`:

* filter([]\*BareMetalHost smartNIC bool) []\*BareMetalHost

  According to the incoming `BareMetalHost` array and conditions, filter out
  eligible BMHs to return.

* configureNetwork()

  configureNetwork() find the corresponding network card in `BMH.status.nics`
  according to the content of the `spec.nicHint.name` field in the
  `networkConfiguration` CR, and then find the corresponding port in `BMH.spec.ports`
  according to `BMH.status.nics.mac`, and then find the corresponding `Port` CR,
  full networkConfiguration into `spec.networkConfigurationRef`.

* deconfigureNetwork()

  deconfigureNetwork() finds the Port CR corresponding to all its network cards
  according to the incoming BMH, and clears its `spec.networkConfigurationRef`.

### Add new CRD and controller

#### Switch

The following is the realization of the switch, including the realization of the
`Device`: `Switch`, `Port` realization: `SwitchPort`, `Configuration` realization:
`SwitchConfiguration`, `Port controller` realization: `SwitchPort controller`.
The following is the specific implementation:

##### SwitchPort CRD

`SwitchPort` CR represents a specific port of a network device, including port information,
the performance of the network device to which it belongs, and the performance of
the connected network card.

```yaml
apiVersion: v1alpha1
kind: SwitchPort
metaData:
  name: port0
  # Point to the Divice to which it belongs
  ownerRef:
    name: switch0
  finalizers:
spec:
  # Represents the port number on the devic to which it belongs
  id: 0
  # The configuration that needs to be configured on this port
  configurationRef:
    name: sc1
    namespace: default
status:
  # Indicates the actual configuration status of the port
  state: Configured
  # Indicates the configuration information currently applied to the port
  configurationRef:
    name: sc1
    namespace: default
```
##### SwitchPortConfiguration CRD

```yaml
apiVersion: v1alpha1
kind: SwitchPortConfiguration
metaData:
  name: sc1
  namespace: default
  finalizers:
    - default-port0
spec:
  # Represents the ACL rules implemented in the switch.
  acl:
    - type: # ipv4, ipv6
      action: # allow, deny
      protocol: # TCP, UDP, ICMP, ALL
      src: # xxx.xxx.xxx.xxx/xx
      srcPortRange: # 22, 22-30
      des: # xxx.xxx.xxx.xxx/xx
      desPortRange: # 22, 22-30
  # Indicates which mode this port should be set to.
  # Allowed values: access, trunk, hybrid
  type: accesss
  untaggedVLAN: 1
  # Indicates which VLAN this port should be placed in.
  vlans:
    - id: 2
    - id: 3
```

##### Switch CRD

```yaml
apiVersion: v1alpha1
kind: Switch
metaData:
  name: switch0
  namespace: default
spec:
  # Indicates the configured protocol, can't specify multiple protocol
  # Allowed values: cli, snmp, netconf, ovs, openflow
  protocol: cli
  # SwitchConfiguration CR's name
  configuration:
  secret:
  # Restricted ports in the switch
  restrictedPorts:
    <SwitchPort CR's name>:
      portID: if 0/1
      # True if this port is not available, false otherwise
      disabled: false
      # Indicates the range of VLANs allowed by this port in the switch
      vlanRange: 1, 6-10
      # True if this port can be used as a trunk port, false otherwise
      trunkDisable: false
```

##### SwitchConfiguration CRDs

SwitchConfiguration CRDs include the login information of switch for different protocol in Switch CRD.

CLISwitchConfiguration CRD
```yaml
apiVersion: v1alpha1
kind: CLISwitchConfiguration
metaData:
  name: sc0
  namespace: default
spec:
  # The type of OS this switch runs
  os: fos
  # The ip address of the switch
  ip: 192.168.0.1
  # The port to use for SSH connection
  port: 22
  # Login credentials of switch
  secret:
```

SNMPSwitchConfiguration CRD
```yaml
apiVersion: v1alpha1
kind: SNMPSwitchConfiguration
metaData:
  name: sc0
  namespace: default
spec:
  mib:
  # The ip address of the switch
  ip: 192.168.0.1
  # The port of SNMP
  port: 161
  # Login credentials of switch
  secret:
```

NETConfSwitchConfiguration CRD
```yaml
apiVersion: v1alpha1
kind: NETConfSwitchConfiguration
metaData:
  name: sc0
  namespace: default
spec:
  # The ip address of switch
  ip: 192.168.0.1
  # The port of netconf client
  port: 830
  # The login credentials of switch
  secret:
```

OVSSwitchConfiguration CRD
```yaml
apiVersion: v1alpha1
kind: OVSSwitchConfiguration
metaData:
  name: sc0
  namespace: default
spec:
  # The ip address of switch
  ip: 192.168.0.1
  # The port to use for SSH connection
  port: 22
  # The login credentials of switch
  secret:
```

OpenflowSwitchConfiguration CRD
```yaml
apiVersion: v1alpha1
kind: OpenflowSwitchConfiguration
metaData:
  name: sc0
  namespace: default
spec:
  # The ip address of switch
  ip: 192.168.0.1
  # The port of openflow
  port: 6633
  # The version of openflow protocol
  # Allow values: 1.0, 1.1, 1.2, 1.3
  version: 1.3
  # The login credentials of switch
  secret:
```

##### SwitchPort Controller

Implement the following protocols to configure `Switch`:
1. Ansible


2. Snmp


3. Netconf

#### SmartNic

##### SmartNicPort
```yaml
apiVersion: v1alpha1
kind: SmartNicPort
metaData:
  name: port2
  ownerRef:
    name: smart1
  finalizers:
  - defaulf-bm0
spec:
  id: 0
  configurationRef:
    name: sn1
    namespace: default
  # Represents the next port information of this port link
  nextPortRef:
    name: port3
    kind: SwitchPort
    namespace: default
status:
  # Indicates the actual configuration status of the port
  state: Configured
  # Indicates the configuration information currently applied to the port
  configurationRef:
    name: sn1
    namespace: default
```
##### SmartNicConfiguration CRD

```yaml
apiVersion: v1alpha1
kind: SmartNicConfiguration
metaData:
  name: sc1
  namespace: default
  finalizers:
    - default-port0
spec:
  ...
  nextConfigurationRef:
  name: sp1
  kind: SwitchPortConfiguration
  namespace: default
```

### Implementation Details/Notes/Constraints

TBD

### Risks and Mitigations

TBD

### Work Items

* Implement networkConfiguration operator
* Change the API of Metal3Machine and BareMetalHost
* Add related methods to CAPM3
* Unit tests

### Dependencies

NONE

### Test Plan

* Unit tests for all the cases should be in place.
* e2e testing in Metal3-dev-env.

## Drawbacks

NONE

## Alternatives

NONE

## References

* [issue](https://github.com/metal3-io/baremetal-operator/issues/570)
* [physical-network-api-prototype](https://github.com/metal3-io/metal3-docs/blob/master/design/physical-network-api-prototype.md)
* [contracts](https://github.com/Hellcatlk/networkconfiguration-operator/docs)
