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
    ConfigurePort(ctx context.Context, networkConfiguration *v1alpha1.NetworkConfiguration, port *v1alpha1.Port) error
    // DeConfigurePort remove the network configure from the port
    DeConfigurePort(ctx context.Context, port *v1alpha1.Port) error
    // PortState return the port's state of the device
    PortState(ctx context.Context, portID string) PortState
}
```

Each type of network device must have its own CRD and implements corresponding
`DeviceManager` interface.

#### NetworkConfiguration

Indicates specific network configuration information.

##### Port

Indicates the specific port of the network device.

#### Match()

The Match() method obtains the `NetworkConfiguration` CR according to the input
`NetworkConfigurationRef`, and checks if the input `Port`s can satisfy the
requirements of all the `NetworkConfiguration`. If the input ports can satisfy
all refs, it will return true, otherwise it will return false.

```go
func Match(client runtime.Client, refs []NetworkConfigurationRef, ports []PortRef)
                                     (bool, error) {
}
```

```go
type NetworkConfigurationRef struct {
    name      string
    namespace string
}
```

```go
type PortRef struct {
    name      string
    namespace string
}
```

### Workflow

Create CR related to BMH's network:

* Administrator creates the corresponding `Device` CR according to the
  specific network device.
* Administrator creates a `BareMetalHost` CR and fills in the content in `bmh.spec.ports`
  according to the actual network connection.
* Administrator creates a corresponding `Port` CR for each port of the network device
  connected to the BMH network card.

Create Metal3Machine:

* User creates `NetworkConfiguration` CR.
* User specifies the `NetworkConfiguration` CR used when creating `Metal3Machine`
  CR.
* CAPM3 selects a group of BMHs according to the original logic.
  If the `networkConfiguration` field of Metal3Machine is not empty,
  call the `filter()` method, select and filter the BMH suitable for the network
  configuration from the currently available BMHs, and then select one BMH
  according to the original method.
* Before provisioning, if the machine's network configuration field is not empty,
  CAPM3 calls the `configureNetwork()` method to fill networkConfiguration into
  the appropriate `Port.spec.networkConfigurationRef`.
* Port controller starts to configure the network by call device.ConfigurePort().
* BMO detects that the Port CR corresponding to all its own network cards are
  configured and continues to provision BMH.

Remove Metal3Machine:

* User deletes the `Metal3Machine` CR.
* CAPM3 calls the `deconfigureNetwork()` to find all the `Port` CR corresponding
  to the BMO network cards, and clear their `networkConfigurationRef`.
* Port  controller starts to deconfigure the network by call device.DeConfigurePort().
* BMO stage detects that the `Port` CR corresponding to all its own network
  cards are cleared after the configuration is completed and continues to
  deprovision BMH.

### Changes to Current API

#### Metal3Machine CRD

```yaml
spec:
  # Specify the network configuration that metal3Machine needs to use
  networkConfigurationRef:
  - name: nc1
    namespace: default
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
        namespace: abc
   - mac:
   ......
```

#### CAPM3

Add the following methods in `metal3machine_manager.go`:

* filter()

  Filter out the BMHs that meet the network configuration requirements
  from a list of BMHs (by calling the `Match()` function in the
  NetworkConfiguration operator).

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

#### NetworkConfiguration CRD

// TODO: The content of the network configuration needs to be negotiated.

// It is not clear what network configuration is required for other types

// of network equipment.

```yaml
metaData:
  name: nc1
  finalizers:
    - m3m1
spec:
  acl:
    - type: // ipv4, ipv6
      action: // allow, deny
      protocol: // TCP, UDP, ICMP, ALL
      src: // xxx.xxx.xxx.xxx/xx
      srcPortRange: // 22, 22-30
      des: // xxx.xxx.xxx.xxx/xx
      desPortRange: // 22, 22-30
  trunk:
  untaggedVLAN:
  vlans:
    - id:
    - id:
  // Indicates whether link aggregation is required. If the value isn't empty,
  // select two ports for link aggregation.
  linkAggregationType: // LAG, MLAG
  // Network card performance required for this network configuration
  nicHint:
    speed: 1000
    smartNIC: false
    name: eth0
status:
  portRefs:
    - name:
      namespace:
    - name:
      namespace:
```

#### Port CRD

`Port` CR represents a specific port of a network device, including port information,
the performance of the network device to which it belongs, and the performance of
the connected network card.

```yaml
metaData:
  name:
  ownerRef:
  finalizers:
spec:
  networkConfigurationRef:
    name:
    namespace:
  portID: 0
  lagWith:
    name:
    namespace:
  deviceRef:
     name:
     kind:
     namespace:
  capability:
     speed: 1000
  smrtNic: false

status:
  state:
  networkConfigurationRef:
    name:
    namespace:
  vlans:
  - 2
  - 3
  .....
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

#### Switch

```yaml
kind: Switch
metaData:
  name: switch0
spec:
  os: fos
  ip: 192.168.0.1
  mac: ff:ff:ff:ff:ff:ff
  secret:
  // Indicates the switch at the other end of the peer link with this switch.
  peerLinkWith:
    - name: switch1
      namespace: def
  ports:
    - portID: 0
      disabled: false
      vlanRange: 1, 6-10
      trunkDisable: false
   - portID: 1
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
