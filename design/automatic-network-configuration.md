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

Abstract all devices in the cluster (including various types of
network devices and BMH).

```go
// DeviceManager is used to control the `Device`
type DeviceManager interface {
    // ConfigurePort set the network configure to the port
    ConfigurePort(ctx context.Context, configuration interface{},
                                      port *v1alpha1.Port) error
    // DeConfigurePort remove the network configure from the port
    DeConfigurePort(ctx context.Context, port *v1alpha1.Port) error
    // CheckPortConfigutation checks whether the configuration is
    // configured on the port
    CheckPortConfigutation(ctx context.Context, portID string) bool
}
```

Each type of network device must have its own CRD and implements corresponding
`DeviceManager` interface.

##### Port

Indicates the specific port of the network device.

#### Configuration

Indicates the content of the specific configuration of each port, and each device needs to implement its own corresponding configuration.
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
  the appropriate `Port.spec.portConfigurationRef`.
* Port controller starts to configure the network by call device.ConfigurePort().
* BMO detects that the Port CR corresponding to all its own network cards are
  configured and continues to provision BMH.

Remove Metal3Machine:

* User deletes the `Metal3Machine` CR.
* CAPM3 calls the `deconfigureNetwork()` to find all the `Port` CR corresponding
  to the BMO network cards, and clear their `portConfigurationRef`.
* Port  controller starts to deconfigure the network by call device.DeConfigurePort().
* BMO stage detects that the `Port` CR corresponding to all its own network
  cards are cleared after the configuration is completed and continues to
  deprovision BMH.

### Changes to Current API

#### Metal3Machine CRD

```yaml
spec:
  // Specify the network configuration that metal3Machine needs to use
  networkConfiguration:
  // no smartNic
  - ConfigurationRefs:
    - name: nc1
      kind: SwitchPortConfiguration
      namespace: default
    // Network card performance required for this network configuration
    nicHint:
      name: eth0
      smartNIC: false
  // with smartNic
  - ConfigurationRefs:
    // configure for smartNic
    - name: nc1
      kind: SmartNicConfiguration
      namespace: default
    // configure for switchPort
    - name: nc2
      kind: SwitchPortConfiguration
      namespace: default
    // Network card performance required for this network configuration
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
      smartNIC: false
      portRef:
        name: port0
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

#### Port CRD

`Port` CR represents a specific port of a network device, including port information,
the performance of the network device to which it belongs, and the performance of
the connected network card.

Example 1: Normal BMH (without smartNic) connect Switch port

```yaml
apiVersion: v1alpha1
kind: Port
metaData:
  name: port0
  // Point to the Divice to which it belongs
  ownerRef:
    name: switch0
    kind: Switch
    namespace: default
  finalizers:
spec:
  // Represents the port number on the devic to which it belongs
  id: 0
  // The configuration that needs to be configured on this port
  configurationRef:
    name: sc1
    kind: SwitchPortConfiguration
    namespace: default
  // Represents the next port information of this port link
  nextRef:
    name:
    namespace: 
status:
  // Indicates the actual configuration status of the port
  state: Configured
  // Indicates the configuration information currently applied to the port
  configurationRef:
    name: sc1
    kind: SwitchPortConfiguration
    namespace: default
  .....
```

Example 2: BMH (with smartNic) connect Switch port

```yaml
apiVersion: v1alpha1
kind: Port
metaData:
  name: port2
  ownerRef:
    name: smart1
    kind: SmartNic
    namespace: default
  finalizers:
  - defaulf-bm0
spec:
  id: 0
  portConfigurationRef:
    name: sn1
    kind: SmartNicConfiguration
    namespace: default
  nextPortRef:
    name: port3
    namespace: default
status:
  // Indicates the actual configuration status of the port
  state: Configured
  // Indicates the configuration information currently applied to the port
  configurationRef:
    name: sn1
    kind: SmartNicConfiguration
    namespace: default
```

```yaml
apiVersion: v1alpha1
kind: Port
metaData:
  name: port3
  ownerRef:
    name: switch0
    kind: Switch
    namespace: default
  finalizers:
spec:
  portID: 3
  portConfigurationRef:
    name: sc2
    kind: SwitchPortConfiguration
    namespace: default
  nextPortRef:
    name: 
    namespace:
  smrtNic: true
status:
  // Indicates the actual configuration status of the port
  state: Configured
  // Indicates the configuration information currently applied to the port
  configurationRef:
    name: sw1
    kind: SwitchPortConfiguration
    namespace: default
```

#### Port Controller

* networkConfigurationRef changed from empty to non-empty
  * Find the corresponding `Device` CR according to `DeviceRef`, then verify that
    the configuration of the port is allowed according to the information in the
    device CR, and then log in to the corresponding device to perform the
    corresponding configuration operation.

* networkConfigurationRef changed from non-empty to empty
  * Find the corresponding `Device` CR according to `DeviceRef`, and then log in
    to the corresponding device to cancel the corresponding configuration.

* networkConfigurationRef content changes
  * Log in to the corresponding device and apply the latest configuration.

#### SwitchPortConfiguration CRD

```yaml
apiVersion: v1alpha1
kind: SwitchPortConfiguration
metaData:
  name: sc1
  namespace: default
  finalizers:
    - default-port0
spec:
  // Represents the ACL rules implemented in the switch.
  acl:
    - type: // ipv4, ipv6
      action: // allow, deny
      protocol: // TCP, UDP, ICMP, ALL
      src: // xxx.xxx.xxx.xxx/xx
      srcPortRange: // 22, 22-30
      des: // xxx.xxx.xxx.xxx/xx
      desPortRange: // 22, 22-30
  // Indicates which mode this port should be set to,  valid values are `access`, `trunk` or `hybrid`.
  type: accesss
  // Indicates which VLAN this port should be placed in.
  vlans:
    - id: 2
    - id: 3
```

#### Switch

```yaml
apiVersion: v1alpha1
kind: Switch
metaData:
  name: switch0
  namespace: default
spec:
  // The type of OS this switch runs
  os: fos
  // IP Address of the switch
  ip: 192.168.0.1
  // Port to use for SSH connection
  port: 22
  secret:
  // Restricted ports in the switch
  restrictedPorts:
    - portID: 0
      // True if this port is not available, false otherwise
      disabled: false
      // Indicates the range of VLANs allowed by this port in the switch
      vlanRange: 1, 6-10
      // True if this port can be used as a trunk port, false otherwise
      trunkDisable: false
    - portID: 2
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
* [demo](https://github.com/Hellcatlk/networkconfiguration-operator)
