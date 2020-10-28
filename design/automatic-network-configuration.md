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

* implement a network controller, this is aimed at being an integration
  with an existing controller only.
* implement a solution for a specific underlying infrastructure.

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
according to the network configuration of the cluster, the corresponding
BMH is automatically configured to the corresponding network.

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
    // The ConfigureLinkAggreation method corresponding to the switch
    // type device adds the processing of the MLAG situation.
    ConfigureLinkAggreation()
    DeConfigureLinkAggreation()
    CheckNetwork()
    HealthCheck()
}
```

Each type of network device must have its own CRD and implements corresponding
`DeviceManager` interface.

#### Port

Abstract the ports of various types of devices.

```go
// PortManager is used to control the `Port`
type PortManager interface {
    // Perform network configuration on the port
    Configure()
     // Cancel the network configuration on the port
    DeConfigure()
}
```

Each type of port must have its own CRD and implements corresponding
`PortManager` interface.

#### PortLink

Represents the connection relationship between two ports.

#### NetworkConfiguration

Indicates specific network configuration information.

#### NetworkBinding

Indicates the correspondence between PortLink and NetworkConfiguration,
which means that this NetworkConfiguration is applied to this PortLink.
Only one NetworkConfiguration can be applied to a PortLink, and one
NetworkConfiguration can be applied to multiple PortLinks.

### workflow

#### networkConfiguration operator workflow

Initialization:

* The administrator performs all physical connections and basic network
  device initialization (including the creation of peer links between switches).
* The administrator creates Device, Port specific resources and PortLink CR
  according to the connection situation
  * Device can be Switch CR or BMH CR
  * The Port may specifically be the SwitchPort CR corresponding to the
    switch or the NIC CR corresponding to the BMH, and each CR includes
    the Ref of the PortLink CR to which the CR belongs.
  * User creates NetworkConfiguration CR, NetworkConfiguration Controller
    adds finalizer to CR.

Configure Network:

* A role (such as CAPM3) can specify the NetworkConfiguration CR to be
  applied to a PortLink CR by creating a NetworkBinding CR.
* When the NetworkBinding controller detects that a new NetworkBinding CR
  is created, it finds the two specific Port CRs connected by the portLink,
  and then configures the port's network. After the configuration is successful,
  the corresponding `status.state` in the NetworkBinding CR is set to ‘complete’.
* The NetworkBinding controller adds the metaData information of
  the networkBinding CR to the `status.NetworkBindings` field of
  the NetworkConfiguration CR.

Modify NetworkConfiguration:

* User modify NetworkConfiguration CR
* After the NetworkConfiguration controller monitors the changes in the
  NetworkConfiguration CR, it finds all the networkBinding CRs according
  to status.networkBindings, and processes each networkBinding CR one by
  one: first delete the corresponding NetworkBinding CR, check that the
  NetworkConfiguration CR is successfully deleted, and then create a new
  NetworkConfiguration CR.

Delete NetworkConfiguration:

* When deleting the NetworkConfiguration CR, the NetworkConfiguration Controller
  determines whether the status.portLinks is empty, and if it is empty, it deletes
  the finalizer from the NetworkConfiguration CR. That is, when the
  NetworkConfiguration CR is used, it cannot be deleted.

#### CAPM3 uses the networkConfiguration operator to configure the network workflow

Create CR related to BMH's network:

* Administrator creates BMH CR.
* Administrator creates a corresponding Port type CR for the port of
  each device (BMH and network device) that needs to be configured to
  connect to the network cable.
* Administrator creates a PortLink CR based on the actual network cable
  connection information, and writes its ref into each Port type CR.
* The administrator adds the information corresponding to the NIC CR
  in the BMH CR.

Create Metal3Machine:

* User creates NetworkConfiguration CR.
* User specifies the NetworkConfiguration CR used when creating Metal3Machine CR.
* CAPM3 selects a group of BMHs according to the original logic.
  If the `networkConfiguration` field of Metal3Machine is not empty,
  call the match() method, select and filter the BMH suitable for the network
  configuration from the currently available BMHs, and then select one BMH
  according to the original method.
* Before provisioning, if the machine's network configuration field is not empty,
  CAPM3 calls the configuration() method. In this method, CAPM3 triggers the
  NetworkBinding Controller to configure the BareMetal network by creating a
  NetworkBinding CR. In addition, CAPM3 also needs to add the information of
  the Metal3Machine to the metaData.OwnerReferences of the NetworkBinding CR
  to ensure that when the Metal3Machine is deleted, the NetworkBinding is also
  will be deleted, which triggers the NetworkBinding Controller to delete the
  network configuration corresponding to BareMetal.
* NetworkBinding controller starts to configure the network.
* Configuration() method polls the status.state field of the NetworkBinding CR
  to determine the result of the network configuration and returns the result.
* After the network configuration is successful, CAPM3 continues to provision BMH.

Remove Metal3Machine:

* User deletes the Metal3Machine CR. Since the OwnerReferences of the
  NetworkBinding CR is the Metal3Machine CR, k8s will also delete the
  NetworkBinding CR.
* NetworkBinding Controller detects that the NetworkBinding CR needs to be
  deleted and starts to clear the network configuration.
* CAPM3 calls the Deconfigure() method to detect whether the NetworkBinding CR
  has been deleted, and returns the result.
* If the network configuration is deleted successfully, continue to
  deprovision the BMH.

### Changes to Current API

#### Metal3Machine CRD

```yaml
spec:
    # Specify the network configuration that metal3Machine needs to use
  networkConfiguration:
    - name: "nc1"
      namespace: "def"
```

#### BareMetalHost

Add the NIC CR corresponding to each network card in BMH.

```yaml
metaData:
  name: "bm1"
spec:
  nics:
    - name: "bm1-eth0"
      namespace: "def"
    - name: "bm1-eth1"
      namespace: "def"
```

#### CAPM3

Add the following methods in `metal3machine_manager.go`:

* match()

  Find out the BMH that meets the network configuration requirements according
  to NetworkConfiguration CR. If the `linkAggregation` field is true, select two
  network cards on the same switch (LAG) or different switches (MLAG) to configure
  link aggregation according to the value of the `linkAggregationType` field.

* configuration()

  Match the network cards one by one according to the NetworkConfiguration CR.
  If linkAggregation is 0, after selecting a suitable network card, directly
  create the corresponding NetworkBinding CR and exit. If linkAggregation is
  greater than 1, find the portLink CR of the corresponding number of network
  cards and fill them in the portLinks field of the newly created
  NetworkBinding CR.

* deconfigure()

  Monitor the processing result after the NetworkBinding CR is deleted and return.

### Add new CRD and controller

#### NetworkConfiguration CRD

// TODO: The content of the network configuration needs to be negotiated.

// It is not clear what network configuration is required for other types

// of network equipment.

```yaml
metaData:
  name: nc1
  finalizers: [m3m1]
spec:
  vlans:
    - id:
    - id:
  # The untagged VLAN ID
  vlanId:
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
  networkBinding:
    - name:
      namespace:
    - name:
      namespace:
```

#### PortLink CRD

Indicates the connection information of the two device ports.

```yaml
apiVersion:
kind: PortLink
metaData:
  name: pl1
  namespace:
spec:
  ports:
    - name: bm0-eth0
      kind: NICPort
      namespace: def
    - name: switch0-port0
      kind: SwitchPort
      namespace: def
```

#### NetworkBinding CRD

```yaml
apiVersion:
kind: NetworkBinding
metaData:
  name:
  namespace:
  finalizers: ["PortLink"]
spec:
  portLinks:
    - name:
      namespace:
    - name:
      namespace:
  networkConfiguration:
    name:
    namespace:
status:
  # Currently applied network configuration, we need to record these configuration
  # for deconfiguring the network because the networkConfiguration referred in `.spec`
  # may be changed.
  networkConfigurationCopy:
  state:
```

#### NetworkBinding Controller

* When NetworkBinding CR is created
  * NetworkBinding Controller copies the networkConfiguration CR to the
    `status.networkConfigurationCopy` field to start configuring the network.
  * Instantiate a specific `PortManager` according to the kind of port in the ports
    field of the `portLink` CR, and then call the `Configure()` method
    corresponding to the `PortManager`.
  * The `Configure()` method corresponding to `PorManager` will instantiate the
    corresponding `DeviceManager`, and then call the `ConfigureNetwork()` method
    corresponding to the `DeviceManager`.
  * According to the length of portLinks, it is determined whether the configuration
    of link aggregation is required. If it is greater than one, the
    ConfigureLinkAggreation() method is called for configuration.
  * After the configuration is complete, the configuration result will be
    fed back to the `status.state` field.
* When the NetworkBinding CR is deleted
  * Determine whether link aggregation needs to be cancelled according to the length
    of portLinks. If it is greater than one, call the `DeConfigureLinkAggreation()`
    method.
  * Call the `DeConfigure` method of the corresponding `Manager` to clear the
    network configuration recorded in `networkBinding.status.networkConfigurationCopy`.
  * Delete NetworkBinding CR

#### CRD of some port instances

##### NICPort

```yaml
kind: NICPort
metaData:
  name: bm0-eth0
spec:
  portLink:
    name: pl1
    namespace: def
  capability:
    speed: 1000
    smartNIC: false
```

##### SwitchPort

```yaml
kind: SwitchPort
metaData:
  name: switch0-port0
spec:
  portLink:
    name: pl1
    namespace: def
  device:
    name: switch0
    kind: SwitchDevice
    namepsace: def
```

#### CRD of device instance

##### Switch

```yaml
kind: Switch
metaData:
  name: switch0
spec:
  specialNamespace: ["def", "abc"]
  os: fos
  ip: 192.168.0.1
  mac: ff:ff:ff:ff:ff:ff
  # Users with different permissions use different accounts to log in to
  # the device to operate.
  account:
    special:
      username:
      password:
    normal:
      username:
      password:
  # Indicates the switch at the other end of the peer link with this switch.
  peerLinks:
    - name: switch0
      namespace: def
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
