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

### Workflow

Create CR related to BMH's network: Switch, ProviderSwitch and Switchport

* Administrator/Infra team creates the corresponding CRs according to the
  specific network infrastructure.
* Administrator creates a `BareMetalHost` CR and fills in the content in `bmh.spec.ports`
  according to the actual network connection to refer to a switchPort.
* Reconciliation of the switch :
	* fetch the Provider switch, if not found, requeue until created
	* Credentials verification (inc. pulling current state)
* Reconciliation of the Switchport:
	* fetch the Switch, check the state of the switch for errors (credentials verification),
	* apply the configuration (empty config = clean the port) (Idle step)

Provisioning workflow:

* User creates the Metal3MachineTemplate referencing a SwitchPortConfig (optional).
* User specifies the `spec.networkConfiguration` field.
* User creates `SwitchPortConfig` for the deployment.
* CAPI / user creates the Metal3Machine (based on the Metal3MachineTemplate),
  referencing the SwitchPortConfig
* CAPM3 filter the BMH by calling the filter() function. Then select a BMH according
  to the original method.
* Before provisioning, if the machine's `networkConfiguration` field is not empty,
  CAPM3 will call the `configureNetwork()` method to fill networkConfiguration into
  the appropriate SwitchPort spec.
  * CAPM3 filters the interfaces of BMH based on the NicHint provided for the network
    configuration
  * CAPM3 identifies the port in the BMH spec that maps to the interface selected based
    on the NicHint
  * CAPM3 fetches the SwitchPort object referred to in the BMH spec
  * CAPM3 sets the SwitchPortConfig in the port specs, and applies it.
  * CAPM3 waits for the port to be configured
* The Switch controller detects that `Port.spec.configurationRef` changed, then carry
  out the corresponding processing :
  * Verify the user configuration to match the switch configuration (allowed vlans etc.)
    (Validating step)
  * Credentials verification (Validating step)
  * Apply the configuration to the port on the switch (Configuring step)
  * Update the port CR (status) (Configuring step)
  * Port is in Active step
* BMO detects that the status.state of the Port CR corresponding to all BMH network
  cards is empty (without config in the specs) or `Active`, and then continues to
  provision BMH.

Update Worflow:

* The Switch controller detects that `Port.spec.configurationRef` changed, then carry
  out the corresponding processing :
  * Verify the user configuration to match the switch configuration (allowed vlans etc.)
    (Validating step)
  * Credentials verification (Validating step)
  * Apply the configuration to the port on the switch (Configuring step)
  * Update the port CR (status) (Configuring step)
  * Port is in Active step

Deprovisioning workflow:

* CAPI / User set the deletion timestamp on the `Metal3Machine` CR.
* CAPM3 calls the `deconfigureNetwork()` to find all the `SwitchPort` CR corresponding
  to the BMO network cards, and clear their `spec.configurationRef`.
  * CAPM3 fetches the BMH
  * CAPM3 fetches all Switchports referenced by the BMH
  * CAPM3 removes any SwitchPortConfig that might be set on the port

* Switch controller starts to deconfigure the network by call device.DeConfigurePort().
  Return the port to a “clean” state
  * Verify the switch credentials (Cleaning Step)
  * Configure the port on the switch (Cleaning Step)
  * If timeout reached, goes to error state
  * Update the switchPort CR
  * SwitchPort in Idle state

* BMO detects that the status.state of the Port CR corresponding to all BMH network
  cards are idle, and then continue to deprovision BMH. If any of the ports are in error
  state -> goes to error state.

Deletion workflow:

* User sets deletion timestamp on all resources (that automatically triggers the deletion
  of the Provider Switch because of OwnerRefs)
* Switch deletion is blocked until all SwitchPorts are deleted
* Wait for switch port to be idle
* If switchPort is idle, then remove the finalizer
* Once no ports are left for the Switch, then remove the finalizer on the ProviderSwitch
  and on the Switch.

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
  configuration: cliSwitch0
  secret:
  # Restricted ports in the switch
  restrictedPorts:
    # SwitchPort CR's name
    port0:
      pxe:
    	  name: pxeConfig
    	  namespace: default
      portName: Geth1-1
      # True if this port is not available, false otherwise
      disabled: false
      # Indicates the range of VLANs allowed by this port in the switch
      vlanRange: 1, 6-10
      # True if this port can be used as a trunk port, false otherwise
      trunkDisable: false
```

##### ProviderSwitch CRDs

SwitchConfiguration CRDs include the login information of switch for different protocol in Switch CRD.

###### CLISwitch CRD
```yaml
apiVersion: v1alpha1
kind: CLISwitch
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

###### SNMPSwitch CRD
```yaml
apiVersion: v1alpha1
kind: SNMPSwitch
metaData:
  name: sc0
  namespace: default
spec:
  mib:
  # The version of snmp protocol
  # Allow values: 1, 2c, 3
  version: 2c
  # SNMP management agent
  community: public
  # The ip address of the switch
  ip: 192.168.0.1
  # The port of SNMP
  port: 161
  # Login credentials of switch
  secret:
```

###### NETConfSwitch CRD
```yaml
apiVersion: v1alpha1
kind: NETConfSwitch
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

###### OVSSwitch CRD
```yaml
apiVersion: v1alpha1
kind: OVSSwitch
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

###### OpenflowSwitchCRD
```yaml
apiVersion: v1alpha1
kind: OpenflowSwitch
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

###### State machine for SwitchPort controller

SwitchPort controller sets 6 states for `Port.status.state`: `None`, `Idle`, `Validating`,
`Configuring`, `Active`, `Cleaning` and `Deleting`, each state has its own
corresponding function.

* \<None\>
  1. Indicates the status of the Port CR when it was first created, and the value
     of `status.state` is nil.
  2. The state handler will add finalizers to the Port CR to avoid being deleted,
     and set the state of CR to `Idle`
* Idle - Steady state
  1. Indicates waiting for spec.configurationRef to be assigned.
  2. The state handler check spec.configurationRef's value, if isn't nil set the
     state of CR to `Validating`
  3. If deletionTimestamp set, goes to Deleting
* Validating
  1. Indicates that the switch and configuration file are verified
  2. The state handlerwill verify whether the switch can be connected and whether
     the specified configuration CR exists. If the verification is successful,
     the state is set to `Configuring`. If it fails, set the state to `Idle`.
* Configuring
  1. Indicates that the port is being configured.
  2. The state handler configure port's network and check configuration progress.
     If finished set the state of CR to `Active` state
* Active - Steady State
  1. Indicates that the port configuration is complete.
  2. The state handler check whether the target configuration is consistent with
     the actual configuration, return to `Configuring` state and clean `status.configurationRef`
     when inconsistent
  3. If deletionTimestamp is set -> got to Cleaning
* Cleaning
  1. Indicates that the port configuration is being cleared.
  2. The state handler deconfigure port's network and check deconfiguration progress,
     when finished clean `spec.configurationRef` and `status.configurationRef` then set
     CR's state to `Idle` state.
  3. If deletionTimestamp set, goes to Deleting
* Deleting
Reached when in Idle state + deletionTimestamp set
  1. Indicates that the port configuration has been cleared.
  2. Prepares for deletion
  3. The state handler will remove finalizers

![state.png](https://github.com/Hellcatlk/networkconfiguration-operator/blob/master/docs/switch/state.png)

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
