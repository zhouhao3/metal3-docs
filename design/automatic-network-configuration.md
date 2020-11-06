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
    // ConfigureNetwork() for basic network configuration and ACL configuration
    ConfigureNetwork()
    DeConfigureNetwork()
}
```

Each type of network device must have its own CRD and implements corresponding
`DeviceManager` interface.

#### NetworkConfiguration

Indicates specific network configuration information.

#### NetworkBinding

Indicates the correspondence between network interface and NetworkConfiguration,
which means that this NetworkConfiguration is applied to this interface.
An interface can only have one NetworkConfiguration applied to, and one
NetworkConfiguration can be applied to multiple interfaces.

#### Match()

The Match() method obtains the `NetworkConfiguration` CR according to the input
`NetworkConfigurationRef`, and checks if the input `Port`s can satisfy the
requirements of all the `NetworkConfiguration`. If true, Match() will build a
`NetworkBinding` struct for each `NetworkConfiguration` and return the slice of the
created `NetworkBinding`s. If false, Match() will return nil and error. The
`Port` used here represents a network interface, for bare metal, it is a NIC.

```go
// Build NetworkBinding CR
func Match(client runtime.Client, refs []NetworkConfigurationRef, ports []Port)
                                     ([]*v1alpha1.NetworkBinding, error) {
}

```

```go
type NetworkConfigurationRef struct {
    name      string
    namespace string
}
```

```go
// Port contains the information of its own capabilities, connected network device
// and specific port number of the device.
type Port struct {
    ConPortID    uint
    ConDeviceRef struct {
        Name      string
        Namespace string
        Kind      string
    }
    Capability struct {
        Speed uint
        SmartNic bool
    }
}
```

![design compose](images/automatic_network_configuration_design_compose.png)

### workflow

Create CR related to BMH's network:

* Administrator creates the corresponding `Device` CR according to the
  specific network device.
* Administrator creates a `BareMetalHost` CR and fills in the content in `bmh.spec.ports`
  according to the actual network connection.

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
  CAPM3 calls the `configureNetwork()` method. In this method, CAPM3 triggers the
  NetworkBinding Controller to configure the BareMetal network by creating
  NetworkBinding CRs. In addition, CAPM3 also needs to add the `Metal3Machine` to
  the `.metaData.OwnerReferences` of the `NetworkBinding` CR to ensure that when
  the `Metal3Machine` is deleted, the corresponding `NetworkBinding` CR will also
  get deleted, which triggers the NetworkBinding Controller to clear the network
  configuration corresponding to BareMetal.
* NetworkBinding controller starts to configure the network.
* `configureNetwork()` method polls the `status.state` field of the
  `NetworkBinding` CR to determine the result of the network configuration.
* After the network configuration is successful, CAPM3 continues to provision BMH.

Remove Metal3Machine:

* User deletes the `Metal3Machine` CR. Since the OwnerReferences of the
  `NetworkBinding` CR is the `Metal3Machine` CR, k8s will also delete the
  `NetworkBinding` CR.
* The NetworkBinding Controller detects that the `NetworkBinding` CR needs to be
  deleted and starts to clear the network configuration.
* CAPM3 calls the `deconfigureNetwork()` method to detect whether the
  `NetworkBinding` CR has been deleted, and returns the result.
* If the network configuration is deleted successfully, continue to
  deprovision the BMH.

![timing diagram](images/automatic_network_configuration_timing_diagram.png)

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
    - conPortID: 10
      conDeviceRef:
        name: switch0
        kind: Switch
        namespace: abc
      capability:
        speed: 1000
        smartNic: false
   - conPortID:
   ......
```

#### CAPM3

Add the following methods in `metal3machine_manager.go`:

* `filter()`

  Filter out the BMHs that cannot meet the network configuration requirements
  from a list of BMHs (by calling the `Match()` function in the
  NetworkConfiguration operator).
  This `filter()` method will be used in the current `chooseHost()` method to
  let `MachineManager` choose a `BMH` that meets the network requirements.

* `configureNetwork()`

  Create all the needed `NetworkBinding` CRs to trigger the network configuration
  operation and check if the configuration is completed or not.

* `deconfigureNetwork()`

  Monitor the processing result after the `NetworkBinding` CR is deleted and return.

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
  VLANs:
    - id:
    - id:
  untaggedVLAN:
  trunk:
  # Indicates whether link aggregation is required. If the value is true,
  # select two ports for link aggregation.
  linkAggregation: false
  linkAggregationType: # LAG, MLAG
  # Network card performance required for this network configuration
  nicHint:
    speed: 1000
    smartNIC: false
  acl:
    - type: # ipv4, ipv6
      action: # allow, deny
      protocol: # TCP, UDP, ICMP, ALL
      src: # xxx.xxx.xxx.xxx/xx
      srcPortRange: # 22, 22-30
      des: # xxx.xxx.xxx.xxx/xx
      desPortRange: # 22, 22-30
status:
  networkBindingRefs:
    - name:
      namespace:
    - name:
      namespace:
```

#### NetworkConfiguration Controller

* User creates `NetworkConfiguration` CR
  * The NetworkConfiguration Controller detects whether the data entered by the
    user is legal, and adds `finalizers`.

* User deletes `NetworkConfiguration` CR
  * The NetworkConfiguration Controller detects whether the length of
    `.status.networkBindingRefs` is 0, if it is not 0, it is not allowed to
    delete the CR, otherwise the Controller deletes the `finalizers` and deletes
    the CR.

* User modify `NetworkConfiguration` CR
  * The NetworkConfiguration Controller deletes all the old `NetworkBinding` CRs
    and creates a new one for each deleted `NetworkBinding` CR according to the
    contents in `.status.networkBindingRefs`.

#### NetworkBinding CRD

`NetworkBinding` CR indicates a networkConfiguration is applied to some ports.
When this CR is created, the operation of configuring network will be triggered,
and when this CR is deleted, the operation of clearing network configuration will
be triggered.
This CR cannot be modified. If user wants to change the network configuration
applied on a port, it is needed to clear the configuration first and then apply
new configuration. So modification is done by deleting old `NetworkBinding` CR
and creating new one.

```yaml
metaData:
  name:
  ownerRef:
  finalizers:
spec:
  networkConfigurationRef:
    name:
    namespace:
  ports:
  - portID:
    deviceRef:
      name:
      kind:
      namespace:
  - portID:
    deviceRef:
      name:
      kind:
      namespace:
status:
  state:
```

#### NetworkBinding Controller

Add the environment variable `ADMIN_NAMESPACE`, the NetworkBinding Controller
judges whether the user has the authority to configure the port according to the
content of `Device CR annotation`.

* User creates `NetworkBinding` CR
  * Instantiate the specific `DeviceManager` according to the `deviceRef`
    information in the `.spec.ports` field, and then call the
    `ConfigureNetwork()` method to modify the corresponding `Device` CR.

* User deletes `NetworkBinding` CR
  * Instantiate the specific `DeviceManager` according to the `deviceRef`
    information in the `.spec.ports` field, and then call the
    `DeConfigureNetwork()` method to modify the corresponding `Device` CR.

#### Switch

```yaml
kind: Switch
metaData:
  name: switch0
  finalizers:
  annotations:
    adminOnlyPorts: "0/22,0/23,3/34-3/45"
    reservedPorts: "0/1,0/2"
spec:
  os: fos
  ip: 192.168.0.1
  mac: ff:ff:ff:ff:ff:ff
  secret:
  # Indicates the switch at the other end of the peer link with this switch.
  peerLinkWith:
  - name: switch1
    namespace: def
status:
  ports:
  - portID:
    # Current network configuration applied to this port
    networkConfiguration:
    # Reference refer to desired networkConfiguration CR
    networkConfigurationRef:
      name:
      namespace:
    lagWith:
    - portID:
      deviceRef:
        name:
        kind:
        namespace:
    state:
```

#### Switch Controller

* Administrator creates `Switch` CR
  * The Switch Controller checks whether the data entered by the user is legal,
    and adds `finalizers`, then uses `Ansible` to obtain all port information
    of the switch, and initializes all ports in `.status.ports`.

* Administrator deletes `Switch` CR
  * The Switch Controller detects whether the number of configured ports is 0.
    If it is not 0, it is not allowed to delete the CR, otherwise delete the
    `finalizers` and delete the CR. The controller determines whether a port is
    configured by checking the port's `.networkConfiguration`. If this field is not
    nil, the port is configured.

* The NetworkBinding Controller modify `.status.ports`
  * The Switch Controller judges whether the contents of `.status.ports.networkConfiguration`
    are as same as the contents of `NetworkConfiguration` CR pointed to by
    `.status.ports.networkConfigurationRef`. If they are not the same, perform the
    next step, else judge whether the field of `.status.ports.state` is `Ready`.
    If it is not Ready, proceed to the next step.
  * The Switch Controller configures the port. If `.status.ports.networkConfigurationRef`
    is empty, clear the configuration according to `.status.ports.networkConfiguration`.
    If `.status.ports.networkConfigurationRef` is not empty, delete the port configuration
    according to `.status.ports.networkConfiguration`, and then copy the contents
    of `NetworkConfiguration` CR pointed to by `.status.ports.networkConfigurationRef`
    to `.status.ports.networkConfiguration`. Then configure the port.
  * The Switch Controller writes the operation instructions to be performed on the
    switch into the `playbook.yml`, and then calls Ansible once in a while to execute
    the `playbook.yml`.

### Implementation Details/Notes/Constraints

TBD

### Risks and Mitigations

TBD

### Work Items

* Implement networkConfiguration operator
* Change the API of Metal3Machine and BareMetalHost
* Add related methods to CAPM3
* Unit tests
* E2e tests in metal3-dev-env

#### Issues waiting to be resolved

* Permissions issue
* The use of ansible (directly call the code or use ansible-operator?)
* In addition to the VLAN of the switch, how should other types of network
  equipment be configured?

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
