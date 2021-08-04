# Virtual Machine Provisioning

This document aims to walk through the process of provisioning a virtual machine in HobbyFarm. It starts with a user's clicking of the "Start Scenario" link, all the way through the display of an ssh terminal for them. This document is written to provide insight on this provisioning process such that accurate troubleshooting assessments may be made. 

## General Flow

o7 _general flow_

In order for a virtual machine to be provisioned, an administrator must first give access for an end-user. This is done always through an [Access Code](entities.md#accesscode), though that access code may be provisioned either directly or indirectly through a [Scheduled Event](entities.md#scheduledevent). 

Once an access code exists that grants access to a [Scenario](entities.md#scenario) or [Course](entities.md#course), an end-user may use that access code (provided they are aware of it) to start a [Session](entities.md#session). 

A session is the first trigger to HobbyFarm that a user wishes to receive virtual machines. It indicates what scenario or course the user has clicked on and as such what virtual machines need to be made available. Once a session has been created, HobbyFarm will also create a corresponding [VirtualMachineClaim](entities.md#virtualmachineclaim). This object represents a to-be-fulfilled request for virtual machines.

Depending on how the virtual machines are to be allocated, HobbyFarm takes steps to get them to the end user. In the case of [Static Provisioning](#staticprovisioning), virtual machines are ready in a pre-existing pool and are simply assigned or "bound" to the user. In the case of [Dynamic Provisioning](#dynamicprovisioning), virtual machines need to be provisioned for the user and so HobbyFarm (or an external provisioner) takes action to create the requested VMs. 

Once the VMs have been created or allocated, their status is reflected in the 
virtual machine claim. Once all VMs in the claim are ready and bound, the vmclaim status changes to bound and ready. This signals to the UI that the VMs are ready for consumption and so the user may proceed. 

Upon completion of the scenario or course (or time-out in the event of idleness), the vmclaim is marked as "tainted". This indicates to HobbyFarm that the VMs are ready for reclamation. The reclamation process, in the case of static provisioning, results in the teardown of these VMs followed by the provisioning of new VMs to satisfy the requirements set forth by the [VirtualMachineSet](entities.md#virtualmachineset). In the case of dynamic provisioning, the virtual machines are torn down with no other changes (as new VMs would be spun up on-demand). 

## Step 1: Setup

