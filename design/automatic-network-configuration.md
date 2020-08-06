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
related to managing physical hosts. As bare metal hosts are provisioned
or later repurposed, there may be corresponding physical network changes
that must be done, such as reconfiguring a ToR switch port. If the provisioning
of the physical host is managed through a Kubernetes API, it would be
convenient to be able to reconfigure related network devices using a
similar API.

## Goals

- Define a Kubernetes API for configuring network devices.
- Automatically configure the network infrastructure for a host when
  adding it to a cluster.
- Do network configuration when deprovisioning the hosts.
- Design a network abstraction that can represent any of the target networking
  configuration, independently of the controller used.

## Non-Goals

- Implement a network controller which is aimed at being an integration with
  an existing controller only.
- Implement a solution for a specific underlying infrastructure.
- Implement a solution for link aggregation.
- It is not a goal (of this spec) to configure the software running on the Host to
  connect to the right networks on the right ports.

## Proposal

This document proposes to add a new mechanic to automatically perform physical
network device configuration when consuming a bmh.
There will be a new operator and a series of CRDs staying in a new separate
repository, and relative changes to CAPM3 and BMO.

### User Stories

#### Story 1

As the administrator of Metal³, I hope that the machine can be automatically
connected to the provisioning network before provisioning.

#### Story 2

As a administrator of Metal³, I hope that the machine can be automatically removed
from the provisioning network after provisioning is completed.

#### Story 3

As a user of Metal³, I hope that the machine can be automatically connected
to the network I specified after provisioning is completed.

#### Story 4

As the administrator of Metal³, I hope that the machine can be automatically
connected to the provisioning network before deprovisioning.

#### Story 5

As a user of Metal³, I hope that when the machine is removed from the cluster,
the network configuration made to the relative network device for this machine
will also be automatically cleared.

## Design Details

### Structure

In the network operator, we abstract the following roles:

#### Device

Abstract all network devices in the cluster.
Each type of network device must have its own CRD (eg. `Switch` CRD) and
an interface for backend implementation.
The CRD contains some basic information of the device
including which `ProviderDevice` to use to interact with the device.
The backend interface then will be implemented by `ProviderDevice`
to use different backends to actually control the device.

#### ProviderDevice

`ProviderDevice` represents the method of interacting with a backend.
It implements the backend interface of the target `Device` and contains
all the information the backend needs to control the device.

#### DevicePort

Indicates a specific port. For different device types, `DevicePort` must
have different CRDs
(eg. `SwitchPort` CRD as `DevicePort` for `Switch` CRD as `Device`).

#### Configuration

Indicates the content of the specific configuration for one port.
Like `DevicePort`, `Configuration` also must have different CRDs for
different device types (eg. `SwitchPortConfiguration` CRD).

#### ResourceLimit

Indicates the limit added to a user. It is created by the administrator
for each user to limit the use of resources (eg. the allowed VLAN range).
Like `DevicePort`, `ResourceLimit` also must have different CRDs for
different device types (eg. `SwitchResourceLimit` CRD).

#### Relative controllers

For each type of device, there will be a serious CRD, but only one
indispensable controller to handle the configuration job.
That controller need to monitor the corresponding `DevicePort`,
and whenever the `Configuration` of a `DevicePort` is changed,
the controller should be able to interact with the `Device` through
the `ProviderDevice` to do the configuration job.

Note: The following content is introduced with switch as a `Device`
and ansible as the backend to control the switch.

### Workflow

The next image shows the workflow of the network operator itself,
not integrated with Metal3:

![](./images/automatic-network-configuration.jpg)

If integrated with Metal3, the workflow is:

#### Create related CRs: Switch, ProviderSwitch

1. Administrator/Infra team creates the corresponding CRs(`Switch` and
  `ProviderSwitch`) according to the specific network infrastructure.
2. Reconciliation of `Switch`:
    1. Fetch the `ProviderSwitch`, if not found, requeue until created.
    2. Credentials verification (inc. pulling current state).
    3. Create all the `SwitchPort` CRs defined in `Switch` CR's `.spec.ports`.
3. Administrator/Infra team creates provisioningConfiguration (this is a specific
  instance of `SwitchPortConfiguration` CRD, which is used to connect to a
  private network used for provisioning and deprovisioning).
4. Administrator/Infra team creates a BMH and fills its `.spec.ports` with
   the actual network connection info.
5. Administrator/Infra team creates a `SwitchResourceLimit` CR for each user.
6. BMO writes the content in its `.spec.ports.provisioningConfiguration`
  (if not empty) into the corresponding `SwitchPort` CR's `.spec.configuration`.
7. SwitchPort Controller configures the switch port using the provisioningConfiguration
   to get the bmh ready for provisioning.

![](./images/automatic-network-configuration-create.png)

#### Provisioning workflow

1. User creates `SwitchPortConfiguration` CRs for the deployment.
2. User creates the `Metal3MachineTemplate` CR referencing `SwitchPortConfiguration`
   CRs.
3. CAPI/user creates the `Metal3Machine` (based on the `Metal3MachineTemplate`),
   referencing the `SwitchPortConfiguration` CRs.
4. CAPM3 filters the BMH by calling the `filter()` function. Then select a BMH according
   to the original method.
5. Before provisioning, if the machine's `.spec.networkConfiguration` is not empty,
   CAPM3 will call the `configureNetwork()` method to write the networkConfiguration
   into the appropriate BMH's spec.
6. BMO writes the `SwitchPortConfiguration` reference into the corresponding
   `SwitchPort` CRs' `.spec.configuration` and waits for the ports to be configured.
7. SwitchPort controller detects that `.spec.configuration` changed,
  then carries out the corresponding processing:
    1. Verify the user configuration to match the switch configuration.
    2. Verify that the user has permission to use the resources according to the
       content of `SwitchResourceLimit` CR (allowed vlans etc.).
    3. Credentials verification.
    4. Apply the configuration to the port on the switch.
    5. Update the `SwitchPort` CR's status.
    6. Switch port is in Active step.

![](./images/automatic-network-configuration-provisioning.png)

#### Update Workflow

1. Switch controller detects that `.spec.configuration` changed, then
  carry out the corresponding processing:
    1. Verify the user configuration to match the switch configuration
    (allowed vlans etc.).
    2. Credentials verification.
    3. Clear the previous configuration.
    4. Apply the configuration.
    5. Update the port CR (status).
    6. Port is in Active step.

#### Deprovisioning workflow

1. CAPI/User sets the deletion timestamp on the `Metal3Machine` CR.
2. CAPM3 calls the `deconfigureNetwork()` to clear the contents of
  BMH's `.spec.ports.configuration`.
3. BMO deconfigures the ports.
    1. `.spec.ports.provisioningConfiguration` is empty:
        1. BMO Clears the `SwitchPort` CR's `.spec.configuration`.
        2. SwitchPort controller starts to deconfigure the network by calling
           `device.DeConfigurePort()`. Return the port to a `cleaning` state.
            1. Verify the switch credentials.
            2. Configure the port on the switch.
            3. If timeout reached, goes to error state.
            4. Update the `SwitchPort` CR.
            5. `SwitchPort` in Idle state.
    2. `.spec.ports.provisioningConfiguration` is not empty:
        1. BMO writes provisioningConfiguration reference into the
          corresponding `SwitchPort` CR.
        2. SwitchPort controller starts to configure the network.
          Return the port to a `configuring` state.
5. BMO detects that all the related `SwitchPort` CRs' `.status.state` are
   idle or active, and then continues to deprovision BMH.
  If any of the ports are in error state, goes to error state.

![](./images/automatic-network-configuration-deprovisioning.png)

#### Deletion workflow

1. User sets deletion timestamp on all resources (that automatically triggers
  the deletion of the `ProviderSwitch` because of OwnerRefs).
2. `Switch` deletion is blocked until all `SwitchPort`s are deleted.
3. Wait for all `SwitchPort`s to be idle.
4. If `SwitchPort` is idle, then remove the finalizer.
5. Once no `SwitchPort`s are left for the `Switch`, then remove the finalizer on
  the `ProviderSwitch` and on the `Switch`.

### Changes to Current API

#### Metal3Machine CRD

```yaml
spec:
  # Specify all the network configuration that metal3Machine needs to use in a list
  networkConfiguration:
  - ConfigurationRef:
      name: nc1
      kind: SwitchPortConfiguration
      namespace: default
    nicHint:
      name: eth0
      smartNIC: false
  - ConfigurationRef:
```

#### BareMetalHost

Add the `ports` field to `.spec` to indicate the connection information and
configuration of all network cards in the `BareMetalHost`.
The mac address is provided to match the introspection data to finally
find the nic that matches the nicHint.

```yaml
spec:
  ports:
  - mac: 00:00:00:00:00:00
    # port refers to a `SwitchPort` which represents the real port on the
    # switch that this nic is connected to
    port:
      name: port0
      kind: SwitchPort
      namespace: default
    # provisioningConfiguration indicates that this nic should be used
    # for provisioning with the referred configuration
    provisioningConfiguration:
      name: provisioningSwitchPortConfiguration
      kind: SwitchportConfiguration
      namespace: default
    # configuration is the network configuration this nic should use
    # after the bmh is deployed
    configuration:
      name: switchPortConfiguration
      kind: SwitchportConfiguration
      namespace: default
```

### Add new CRD and controller

#### Switch

The following is the implementation for switch:

##### Switch CRD

```yaml
apiVersion: metal3.io/v1alpha1
kind: Switch
metadata:
  name: switch-example
  namespace: default
spec:
  provider:
    kind: AnsibleSwitch
    name: switch-sample
  # ports is a map whose key is the name of a SwitchPort CR and value is
  # some info and limit of the port that SwitchPort CR represents
  ports:
    "switchport-example":
      # The real port name in the switch
      name: <port-name>
      # True if this port is not available, false otherwise
      disabled: false
      # Indicates the range of VLANs allowed by this port in the switch
      vlanRange: 1, 6-10
      # True if this port can be used as a trunk port, false otherwise
      trunkDisable: false
```

##### ProviderSwitch CRD

ProviderSwitch CRDs include the login information of switches for different
protocols. For each ProviderSwitch CRD, there will be a provider
implementation knowing how to interact with the switch with this information.

###### AnsibleSwitch CRD

Use ansible as the backend to connect to the configuration switch
(using the network-runner tool, currently only supports the switches supported
by network-runner).

```yaml
apiVersion: v1alpha1
kind: AnsibleSwitch
metaData:
  name: switch-sample
  namespace: default
spec:
  # The type of OS this switch runs
  os: fos
  # The ip address of the switch
  ip: 192.168.0.1
  # The port to use for SSH connection
  port: 22
  # Only for ovs switch
  bridge:
  # The login credentials of switch
  credentialsSecret:
    name: switch-example-secret
```

##### SwitchPort CRD

```yaml
apiVersion: metal3.io/v1alpha1
kind: SwitchPort
metadata:
  name: switchport-example
  ownerReferences:
    - apiVersion: metal3.io/v1alpha1
      kind: Switch
      name: switch-example
      uid: <switch-uuid>
spec:
  configuration:
    name: switchportconfiguration-example
    namespect: default
```

##### SwitchPortConfiguration CRD

```yaml
apiVersion: metal3.io/v1alpha1
kind: SwitchPortConfiguration
metadata:
  name: switchportconfiguration-example
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
  # Indicates which VLAN this port should be placed in.
  untaggedVLAN: 1
  # The range of tagged vlans.
  vlans: "2-10"
  disabled: false
```

##### SwitchResourceLimit CRD

A `SwitchResourceLimit` CR must be created by the administrator for each
user under their namespace.
There is only one `SwitchResourceLimit` CR in one namespace, and
all the `SwitchResourceLimit` CR must use the same name 'user-limit'.

```yaml
apiVersion: metal3.io/v1alpha1
kind: SwitchResourceLimit
metadata:
  name: user-limit
  namespace: user1
spec:
  # Indicates the range of VLANs allowed by the user
  vlanRange: 5-10
```

##### SwitchPort Controller

###### State machine for SwitchPort controller

SwitchPort controller sets 6 states for Port.status.state:
None, Idle, Configuring, Active, Cleaning and Deleting,
each state has its own corresponding function.

- \<None>
    1. Indicates the status of the Port CR when it was first created,
    and the value of `status.state` is nil.
    2. The state handler will add finalizers to the Port CR to avoid being
    deleted, and set the state of CR to `Idle`.
- Idle - Steady state
    1. Indicates waiting for spec.configurationRef to be assigned.
    2. The state handler check spec.configurationRef's value,
    if isn't nil set the state of CR to Configuring
    3. If deletionTimestamp set, goes to `Deleting`.
- Configuring
    1. Indicates that the port is being configured.
    2. The state handler configures the port's network and checks configuration
    progress. If finished set the state of CR to `Active` state.
- Active - Steady State
    1. Indicates that the port configuration is complete.
    2. The state handler check whether the target configuration is consistent
    with the actual configuration, return to `Configuring` state and clean
    `status.configurationRef` when inconsistent.
    3. If deletionTimestamp is set -> go to `Cleaning`
- Cleaning
    1. Indicates that the port configuration is being cleared.
    2. The state handler deconfigure port's network and check deconfiguration
    progress, when finished clean `spec.configurationRef` and
    `status.configurationRef` then set CR's state to `Idle` state.
    3. If deletionTimestamp set, goes to Deleting.
- Deleting Reached when in Idle state + deletionTimestamp set
    1. Indicates that the port configuration has been cleared.
    2. Prepares for deletion.
    3. The state handler will remove finalizers.

![](./images/automatic-network-configuration-state.png)

### Multi-tenancy models

- No multi-tenancy : one set of controllers deployed, all objects are
  reconciled by the same controllers.
- "Limited" multi-tenancy : one set of CRs (the most up-to-date), one set of
  webhooks and a single network controller with clusterRole and in its own
  namespace. All other controllers (CAPI + CAPM3) per namespace. Permission
  separation between namespaces.
- Full multi-tenancy : one set of CRs (the most up-to-date), one set of
  webhooks. All other controllers per namespace. No permission separation
  within the same namespace.

### Permissions

The permissions for a role include the permissions given to the controllers
deployed by that role.

#### Scenario: One cluster operator

Description: Only one cluster operator has full management access to all resources

Multi-tenancy models:

- None

Roles:

- Operator: Full control
  Scope: Cluster

Operations Allowed:
||Operation / resource|Goal|
|:-|:-|:-|
|Operator|All/All|Configure everything|

Operations Not Allowed:
||Operation|Goal|
|:-|:-|:-|
|Operator|None||

#### Scenario: Many cluster operators

Description: Each operator has full management access to all resources in his namespace.

Multi-tenancy models:

- Full

Roles:

- Operator: Full control
  Scope: Cluster

Operations Allowed:
||Operation / resource|Goal|
|:-|:-|:-|
|Operator|All/All|Configure everything|

Operations Not Allowed:
||Operation|Goal|
|:-|:-|:-|
|Operator|None||

#### Scenario: Infra team and consumers

Description: There is an owner of the infra and some consumers. The infra
team is the owner of the management cluster and manages the physical resources.
A consumer is the manager of the target cluster, responsible for its lifecycle.
The consumer gets resources from the infra team (tenant). No consumer action
should impact another consumer. Metal3 setup on a shared management cluster,
with namespaces for each consumer. In order to ensure that users do not affect
each other, the infra team writes the resource limit (vlan range etc.)
for each user into the ResourceLimit CR.

The switch is shared between multiple consumers (dedicated ports, vlans etc.)
and managed by the infra team.

Multi-tenancy models:

- Limited

Roles:

- User: Configuration of existing ports
  Scope: Namespaced
- Operator: Definition of the infra
  Scope: Namespaced

Operations Allowed:
||Operation/resource|Goal|
|:-|:-|:-|
|User|Create-delete-Update/SwitchPortConfiguration|Create any configuration|
|User|Modify the SwitchPort|Link a configuration|
|Operator|All/All||

Operations Not Allowed:
||Operation/resource|Goal|
|:-|:-|:-|
|User|Create-Delete/SwitchPort|Modify the infra objects|
|User|All/Switch|Modify the infra objects|
|Operator|None||

### Implementation Details/Notes/Constraints

#### Immutable approach

- SwitchPortConfiguration object immutable.
- Changes would be done by new configuration and linking the new configuration
  in the ports.
- Reconciliation triggered for each port.
- Required to keep the CAPI model support. New configuration should not impact
  old machine running.

#### Clean port definition

Defined by infra team, which ports are for PXE/iPXE boot, which are not.

Two cleaning configurations:

1. PXE/iPXE boot (allows IPA booting).
2. Deconfigured (turning port off would disable LLDP messages for example,
   preventing IPA from gathering this info).
    1. Clean all vlan settings will be enough, acl etc.
    2. Port is still up.
    3. LLDP enabled.

### Risks and Mitigations

- Breaking cluster during upgrade by adding empty switchPorts.

### Work Items

- Implement network operator.
- Change the API of Metal3Machine and BareMetalHost.
- Add related methods to CAPM3.
- Unit tests.

### Dependencies

- Ansible for some switch providers. Ansible collections for each switch OS.
- Go libraries for other protocols
- Kube-builder.

### Test Plan

#### Unit tests

- Mocks support (internal interfaces and external, physical devices mocking)
- Create a “fixture” switchProvider.
- Create a go mock for the provider interface.
- K8S fake client

#### Integration tests

ProviderSwitch needs extensive testing (ansible playbooks with
[Molecule](https://molecule.readthedocs.io/en/latest/examples.html) for example).
[fake-switches](https://github.com/internap/fake-switches) could be used to
mock hardware.

#### E2E tests

- metal3-dev-env with OVS/linux bridge.
- Integrate the tests in the integration test suite in Jenkins.

### Upgrade/downgrade strategy

Upgrade would be supported directly as the changes are API additions.
If not defined, the behaviour of upgraded BMO or CAPM3 does not change.
This provides some compatibility for objects deployed without the networking
support. Backwards compatible (BMH ports not provided).

To start using the network configuration tools, the user will need to create
the switchPort with a SwitchPortConfiguration already associated, otherwise
the port on the switch would be deconfigured. It could also work by creating
the objects during a rolling upgrade of BMHs.

To stop using those tools, this could be achieved during a rolling upgrade.

We will not support downgrade for now.

## Drawbacks

NONE

## Alternatives

NONE

## References

- [Issue](https://github.com/metal3-io/baremetal-operator/issues/570)
- [Physical-network-api-prototype](https://github.com/metal3-io/metal3-docs/blob/master/design/physical-network-api-prototype.md)
- [Contracts](https://github.com/Hellcatlk/network-operator/tree/master/docs)
- [POC](https://github.com/Hellcatlk/network-operator)
