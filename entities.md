# Entities

## User

User is a representation of an end-user or administrator in HobbyFarm. Administrator status is denoted by `spec.admin: true`. 

Access codes that a user has input are stored in `spec.access_codes`. 

Password is stored as hash. 

```go
type UserSpec struct {
    // unique identifier
	Id          string   `json:"id"`
    // user's email - doesn't technically have to be an email address though
	Email       string   `json:"email"`
    // hashed pw
	Password    string   `json:"password"`
    // list (slice) of access codes this user has input
	AccessCodes []string `json:"access_codes"`
    // whether or not the user is an admin. default false obviously
	Admin       bool     `json:"admin"`
}
```

## Environment

An Environment describes a cloud provider or other system into which virtual machines can be provisioned. An environment encapsulates all information necessary for VMs to be provisioned. 

```go
type EnvironmentSpec struct {
    // pretty name for the environment
	DisplayName          string                       `json:"display_name"`
    // optional, denotes a domain name at which hostnames should be reachable.
	DNSSuffix            string                       `json:"dnssuffix"`
    // indicates the type of provider in use. influences use of terraform modules when internal provisioning in use
	Provider             string                       `json:"provider"`
    // provides specifics about template implementation in this environment
    // example:
    //
    // aws-hf-us-east-1:
    //   ubuntu1604-docker1:
    //     image: ami-098230498234
    //     sshUsername: ubuntu
	TemplateMapping      map[string]map[string]string `json:"template_mapping"`
    // specificities about the environment necessary for provisioning
	EnvironmentSpecifics map[string]string            `json:"environment_specifics"`
    // in environments behind nat, this defines how subnets are mapped from internal to external ips
	IPTranslationMap     map[string]string            `json:"ip_translation_map"`
    // the websocket endpoint this environment should point to when creating new shell sessions 
	WsEndpoint           string                       `json:"ws_endpoint"`
    // "count" or "raw", referring to capacity being measured either in number of vms or in the consumption of their specific resources
	CapacityMode         CapacityMode                 `json:"capacity_mode"`
    // whether this environment is capable of provisioning VMs in a burst method
	BurstCapable         bool                         `json:"burst_capable"`
    // the capacity of the environment measured in count of vms
	CountCapacity        map[string]int               `json:"count_capacity"`
    // the capacity of the environment measured in raw resources
	Capacity             CMSStruct                    `json:"capacity"`
    // the burst capacity of the environment measured in count of vms
	BurstCountCapacity   map[string]int               `json:"burst_count_capacity"`
    // the burst capacity of the environment measured in raw resources
	BurstCapacity        CMSStruct                    `json:"burst_capacity"`
}

type EnvironmentStatus struct {
    // how much of the resources are currently used in the environment
    // this is the "use" analog to raw capacity measurements
	Used           CMSStruct      `json:"used"`
    // how many vms are available in the environment
    // this is the "not in use" analog to the count measurements
	AvailableCount map[string]int `json:"available_count"`
}
```

## ScheduledEvent

A ScheduledEvent defines a period of time over which course(s) or scenario(s) are made available to end users. A scheduled event defines how the virtual machine resources are to be made available, how many VMs are to be created, and in which environments they should be provisioned.

```go
type ScheduledEventSpec struct {
    // id of the user who created this scheduled event
	Creator                 string                    `json:"creator"`
    // friendly name of the event
	Name                    string                    `json:"event_name"`
    // friendly description of the event
	Description             string                    `json:"description"`
    // when the event will start
	StartTime               string                    `json:"start_time"`
    // when the event will end
	EndTime                 string                    `json:"end_time"`
    // whether or not to pre-provision vms. if false, vms are pre-provisioned. if true, vms are provisioned dynamically upon user request (session creation)
	OnDemand			    bool					  `json:"on_demand"`
    // the required virtual machines for this scheduled event
    // this data structure defines the environments, vmtemplates, and counts of vms to provision
    // an example...
    //
    //   required_vms:
    //     aws-hf-us-east-2:
    //       ubuntu1604-docker1: 20
    //     aws-hf-us-east-1:
    //       ubuntu1604-docker1: 60
	RequiredVirtualMachines map[string]map[string]int `json:"required_vms"`
    // the code that will be used to access the scenario(s) or course(s). This is the code itself, not the ID of the accesscode entity
	AccessCode              string                    `json:"access_code"`
    // whether sessions that use this access code are required to also use virtual machines that belong to this scheduled event
	RestrictedBind          bool                      `json:"restricted_bind"`
    // the id of this scheduled event. used for coordination across entities
	RestrictedBindValue     string                    `json:"restricted_bind_value"`
    // a slice of scenario ids to which this event will give access
	Scenarios               []string                  `json:"scenarios"`
    // a slice of course ids to which this event will give access
	Courses                 []string                  `json:"courses"`
}

type ScheduledEventStatus struct {
    // the id of the accesscode entity that has been created for this scheduledevent
	AccessCodeId       string   `json:"access_code_id"`
    // the id(s) of virtualmachinesets that have been created for this scheduledevent. nil if on_demand is true
	VirtualMachineSets []string `json:"vmsets"`
    // true if the event has started and not yet expired
	Active             bool     `json:"active"`
    // true if the pre-provisioned vms have been created, the access code has been created, etc.
	Provisioned        bool     `json:"provisioned"`
    // true if all pre-provisioned vms are in ready state
	Ready              bool     `json:"ready"`
    // true if the event's expiration time has passed
	Finished           bool     `json:"finished"`
}
```

## AccessCode

An access code is a string used to restrict access to scenario(s) and/or course(s).
Access codes can be created by administrators either via creating a ScheduledEvent or by directly creating the access code. 

When created via a ScheduledEvent the access code expires at the same expiration time of the ScheduledEvent.

When created directly via API, an access code may specify an independent expiration or `nil` indicating no expiration time.

```go
type AccessCodeSpec struct {
    // the code, alphanumeric string > 4 characters (UI convention)
	Code                string   `json:"code"`
    // description of the use case of the access code
	Description         string   `json:"description"`
    // slice of scenario ids to which this code grants access
	Scenarios           []string `json:"scenarios"`
    // slice of course ids to which this code grants access
	Courses             []string `json:"courses"`
    // Go time string for when this access code is no longer valid
	Expiration          string   `json:"expiration"`
    // slice of vmset ids to which this access code will provision VMs
	VirtualMachineSets  []string `json:"vmsets"`
    // determines if this access code requires use of a specific set of VMs denoted by restricted_bind_value
	RestrictedBind      bool     `json:"restricted_bind"`
    // indicates a scheduledevent id to which any VMs provisioned for this code must be associated
	RestrictedBindValue string   `json:"restricted_bind_value"`
}
```

## Scenario

A scenario is the basic unit of education in HobbyFarm. It stores the content as a series of steps through which a user walks. 

The `name`, `description`, and all of the `steps` are base64-encoded. The most important reason for this is that directly storing this content would likely result in formatting errors. The admin UI displays this content with a WYSIWYG editor and translation between that, the HobbyFarm API, and Kubernetes could easily munge spaces, tabs, newlines, etc. An ancillary use case for the base64 storage is smaller transmission size, though this is not a primary goal.

A scenario dictates what virtual machines are required for its execution; they are specified in the `virtualmachines` slice. This `[]map[string]string` specifies "sets" of virtual machines, which are maps of VM names to VirtualMachineTemplates. The use of a slice for `virtualmachines`  was decided upon such that "sets" of virtual machines could be hosted in different environments or providers depending on the requirements of those underlying VirtualMachineTemplates. The `map[string]string` identifies the "pretty" or "useful" name of a VM which results in the naming of the shell tabs in the user's interface. 

The `keepalive_duration` period is used as an idle timeout after which virtual machine resources are reaped. HobbyFarm only take action if this period elapses _without_ a user calling the `keepalive()` method (done automatically from the UI) every ~30s. The original design inspiration was the idea of a user closing their laptop without properly completing their session - this prevents idle consumption of resources. 

`pause_duration` is good for situations such as when users are going to lunch, or expect to be away from their session for an extended period such as a classroom activity between interactivity. Pausing a session indicates to HobbyFarm that the user wishes their VMs to be kept around even if the keepalive duration expires. 

This duration is the maximum time a user is allowed to pause their session. If a user pauses their session, and this period elapses, HobbyFarm will automatically "resume" their session (thus restarting the keepalive timer).

```go
type ScenarioSpec struct {
    // unique identifier, almost always matches the k8s object name
	Id                string              `json:"id"`
    // display name for the scenario, base64 encoded
	Name              string              `json:"name"`
    // description for the scenario, b64 encoded
	Description       string              `json:"description"`
    // slice of steps
	Steps             []ScenarioStep      `json:"steps"`
    // required virtual machines. identifies the specific templates that this scenario requires, as well as establishing names for the VMs
    // example:
    // 
    //  virtualmachines:
    //  - node01: ubuntu1604-docker1
    //    rancher: ubuntu1604-docker1
    //  - cluster01: ubuntu1604-docker1
    //    cluster02: sles15-sp2
    //    cluster03: rhel78
	VirtualMachines   []map[string]string `json:"virtualmachines"`
    // duration of inactivity after which a user's VMs are destroyed
	KeepAliveDuration string              `json:"keepalive_duration"`
    // duration of time for which a user is allowed to pause their session
	PauseDuration     string              `json:"pause_duration"`
    // whether or not the scenario is pauseable
	Pauseable         bool                `json:"pauseable"`
}

type ScenarioStep struct {
    // base64-encoded title for this particular step
	Title   string `json:"title"`
    // base64-encoded content of this step
	Content string `json:"content"`
}
```

## Course

A course describes a set of [Scenarios](#scenario). It is used to group like-content scenarios together to form a coherent learning path. This is especially useful in multi-stage learning products, or for multi-day learning events. 

`Name`, and `description` are both base64-encoded fields.

`Scenarios` is a slice containing the IDs of the scenarios that are grouped together by this course. This slice also defines the ordering of these scenarios. 

`VirtualMachines` functions the same as in [Scenarios](#scenario), as do `keepalive_duration`, `pause_duration`, and `pauseable`. 

```go
type CourseSpec struct {
    // unique identifier, almost always matches the k8s object name
	Id                string              `json:"id"`
    // display name for the course, base64 encoded
	Name              string              `json:"name"`
    // description for the course, base64 encoded
	Description       string              `json:"description"`
    // slice of scenario ids to which this course gives access
	Scenarios         []string            `json:"scenarios"`
    // required virtual machines. identifies the specific templates that this scenario requires, as well as establishing names for the VMs
    // example:
    // 
    //  virtualmachines:
    //  - node01: ubuntu1604-docker1
    //    rancher: ubuntu1604-docker1
    //  - cluster01: ubuntu1604-docker1
    //    cluster02: sles15-sp2
    //    cluster03: rhel78
	VirtualMachines   []map[string]string `json:"virtualmachines"`
    // duration of inactivity after which a user's VMs are destroyed
	KeepAliveDuration string              `json:"keepalive_duration"`
    // duration of time for which a user is allowed to pause their session
	PauseDuration     string              `json:"pause_duration"`
    // whether or not the scenario is pauseable
	Pauseable         bool                `json:"pauseable"`
}
```

## Session

A Session is an instance of a scenario or course being executed by a user. The session object tracks metadata about this occurrence such as when the scenario or course was started, by whom it was started, when it is set to expire, etc. It is the object that gives life to a user doing learning in HobbyFarm. 

A session tracks, via `scenario` or `course`, what content is being delivered. The `user` field ties this to a [User](#user). 

A session is the jumping-off point for virtual machine provisioning and allocation via the `vm_claim` field. This is a list of [VirtualMachineClaims](#virtualmachineclaim) that have been created as a result of the Session being created. 

The session's status tells us all we need to know about the state of the session. `Paused`, and `paused_time` respectively tell us whether or not the session is paused and if so, at what time it was paused. `Active` tells us if the session is still occurring - this changes to `false` if the `end_time` passes. 

`Finished` tells us if the user clicked the Finish button while executing their course or scenario. A finished session is one that is ready for reaping - and the child virtual machines are marked as such by HobbyFarm. `Finished` is also set to `true` when the `end_time` passes and HobbyFarm executes clean-up actions. 

```go
type SessionSpec struct {
    // unique identifier, almost always matches the k8s object name
	Id         string   `json:"id"`
    // the id of the scenario being accessed
	ScenarioId string   `json:"scenario"`
    // the id of the course being accessed
	CourseId   string   `json:"course"`
    // the user id to which this session belongs
	UserId     string   `json:"user"`
    // the virtual machine claim owned by this session
	VmClaimSet []string `json:"vm_claim"`
    // the access code granting access to the scenario or course
	AccessCode string   `json:"access_code"`
}

type SessionStatus struct {
    // whether or not the session is paused
	Paused         bool   `json:"paused"`
    // at what time the session was paused
	PausedTime     string `json:"paused_time"`
    // whether or not the session is active
	Active         bool   `json:"active"`
    // whether or not the session is finished
	Finished       bool   `json:"finished"`
    // at what time the user started the session
	StartTime      string `json:"start_time"`
    // at what time the session is set to expire
	ExpirationTime string `json:"end_time"`
}
```

## VirtualMachineClaim

A VirtualMachineClaim or vmclaim is a request for virtual machine resources. A vmclaim takes the required VMs from a [Scenario](#scenario) or [Course](#course), along with the provisioning requirements set forth by an [Access Code](#accesscode), and marries them together. This allows HobbyFarm to make decisions about how to satisfy a user's request for virtual machines. 

The VirtualMachineClaimStatus indicates the state of the vmclaim at a point in time. 

`bind_mode` tells us what mode is being used to ultimately bind the virtual machines:

* `static`: The VMs are bound from a pool of pre-provisioned virtual machines.
* `dynamic`: The VMs are being spun up on-demand, either by design, or because of static pool insufficiency. 

`static_bind_attempts` tells us how many times HobbyFarm attempted to pull virtual machines from a pool of pre-provisioned VMs. There is a cap on this number, it is defined [here](https://github.com/hobbyfarm/gargantua/blob/f8b587cda738414fe83af87255c06788d1dd8cc9/pkg/controllers/vmclaimcontroller/vmclaimcontroller.go#L23). If this cap is exceeded, HobbyFarm will switch to dynamic bind mode, attempting to find burst pool machines.

`dynamic_bind_request_id` is filled if the vmclaim is attempting to be satisfied dynamically. This field will contain the ID of the corresponding [DynamicBindRequest](#dynamicbindrequest).

`bound`, `ready`, and `tainted`, tell us about the state of the virtual machines. When `bound` is set to true, the vmclaim has received an allocation for all requested virtual machines. Those machines, however, may not yet be `ready`. Once they become ready, they are able to be used, and so the `ready` field indicates this to the user interface. 

`tainted` tells us when virtual machines are ready for reclamation. This field is set to `true` when a session expires. This field is used to signal to HobbyFarm it is time to taint and reclaim the individual VMs that have been bound to this vmclaim. 

```go
type VirtualMachineClaimSpec struct {
    // unique identifier, almost always matches the k8s object name
	Id                  string                           `json:"id"`
    // id of the user to which this vmclaim belongs
	UserId              string                           `json:"user"`
    // whether or not this vmclaim shall be bound to resources provisioned by a scheduled event
	RestrictedBind      bool                             `json:"restricted_bind"`
    // the id of the scheduled event to which this virtual machine claim shall derive its resources
	RestrictedBindValue string                           `json:"restricted_bind_value"`
    // maps each named vm (e.g. cluster01) from a scenario or course to a provisioned vm, according to a template
    // example:
    //
    // vm:
    //   cluster01:
    //     template: ubuntu2004-docker1
    //     vm_id: dynamic-500e1b92-9fabba3e
    //   rancher01:
    //     template: ubuntu2004-docker1
    //     vm_id: dynamic-500e1b92-f778858b
	VirtualMachines     map[string]VirtualMachineClaimVM `json:"vm"`
    // whether or not this virtual machine claim is capable of having its requested vms satisifed dynamically
	DynamicCapable      bool                             `json:"dynamic_bind_capable"`
}

type VirtualMachineClaimStatus struct {
    // what the resultant bind mode is after the vmclaim has been processed
    // this may either be 'static' or 'dynamic'
    // which of those it is depends on a number of factors including available vms in an environment, restricted bind settings, etc.
	BindMode             string `json:"bind_mode"`
    // the number of attempts hobbyfarm has made to statically bind vms for this vmclaim
	StaticBindAttempts   int    `json:"static_bind_attempts"`
    // the id of the dynamic bind request in case of dynamic bind
	DynamicBindRequestId string `json:"dynamic_bind_request_id"`
    // whether or not all vms are bound
	Bound                bool   `json:"bound"`
    // whether or not all vms are ready
	Ready                bool   `json:"ready"`
    // whether or not the vm claim is tainted
	Tainted              bool   `json:"tainted"`
}

type VirtualMachineClaimVM struct {
    // the virtualmachinetemplate for this vm
	Template         string `json:"template"`
    // the id of the virtualmachine object
	VirtualMachineId string `json:"vm_id"`
}
```

## VirtualMachine

The VirtualMachine entity represents a virtual machine that has been provisioned in a cloud provider or environment.

VirtualMachines can be provisioned either by HobbyFarm directly (using an internal terraform provisioner) or via an external provisioner. See the docs below for the state of fields depending on the provisioner. 

Provisioner is determined via an annotation on an [Environment](#environment). If the environment has the annotation `hobbyfarm.io/provisioner`, HobbyFarm will **not** provision the VM. Otherwise, Terraform will attempt to create the VM. 

```go
type VirtualMachineSpec struct {
    // unique identifier, almost always matches the k8s object name
	Id                       string `json:"id"`
    // id of the virtualmachinetemplate from which this vm has been created
	VirtualMachineTemplateId string `json:"vm_template_id"`
    // username to be used when ssh'ing into this vm
	SshUsername              string `json:"ssh_username"`
    // kubernetes secret name that contains the ssh keypair
	KeyPair                  string `json:"keypair_name"`
    // the id of the virtualmachineclaim that "owns" this vm
	VirtualMachineClaimId    string `json:"vm_claim_id"`
    // id of the user to which this vm "belongs"
	UserId                   string `json:"user"`
    // whether or not this virtual machine should be provisioned by hobbyfarm itself (e.g. terraform)
	Provision                bool   `json:"provision"`
    // if this vm belongs to a vmset, the id of that vmset
	VirtualMachineSetId      string `json:"vm_set_id"`
}

type VirtualMachineStatus struct {
    // indicates overall vm status. default nil. could be one of readyforprovisioning, provisioning, running, terminating
	Status        VmStatus `json:"status"`
    // whether or not the vm has been allocated to a vmclaim 
	Allocated     bool     `json:"allocated"`
    // set to true when a user gets access to the vm. false otherwise
	Tainted       bool     `json:"tainted"`
    // public ip of the vm. set either by terraform provisioning or an external provisioner
	PublicIP      string   `json:"public_ip"`
    // private ip of the vm. set either by terraform provisioning or an external provisioner
	PrivateIP     string   `json:"private_ip"`
    // id of the environment from which the vm has been provisioned
	EnvironmentId string   `json:"environment_id"`
    // hostname of the virtual machine. terraform provisioning sets this to cloud provider hostname, e.g. vm.ec2.amazonaws.com. external provisioner behavior may vary
	Hostname      string   `json:"hostname"`
    // the name of the terraform state object that hobbyfarm generated when provisioning this vm. not used when externally provisioned.
	TFState       string   `json:"tfstate,omitempty"`
    // the hobbyfarm endpoint that can be used to reach an ssh session for this vm
	WsEndpoint    string   `json:"ws_endpoint"`
}
```

## VirtualMachineSet

A VirtualMachineSet is used to provision a number of identical virtual machines. It is used to create a pre-provisioned pool of VMs (in contrast to VMs that would be provisioned "on demand"). The VMSet can be used for either HobbyFarm internal provisioning, or for external provisioners. 

```go
type VirtualMachineSetSpec struct {
    // desired count of virtual machines in this set
	Count               int    `json:"count"`
    // the environment into which the vms should be provisioned
	Environment         string `json:"environment"`
    // the id of the vmtemplate to be used when provisioning the vms
	VMTemplate          string `json:"vm_template"`
    // base name is a prefix for the created vms
    // for example, if base name is "testing-", generated vms will be 
    // something like "testing-09js98wd-89usefs"
	BaseName            string `json:"base_name"`
    // whether or not the virtual machines in this set are for exclusive (restricted) binding only. false if it is a general use pool of vms
	RestrictedBind      bool   `json:"restricted_bind"`
    // the id of the scheduled event to which these virtual machines may belong
	RestrictedBindValue string `json:"restricted_bind_value"`
}

type VirtualMachineSetStatus struct {
    // used if hobbyfarm is doing the provisioning. a slice of provision status objects
	Machines         []VirtualMachineProvision `json:"machines"`
    // of the provisioned count of vms, how many are available for use
	AvailableCount   int                       `json:"available"`
    // total number of vms that have been provisioned. point-in-time indicator, ceiling is spec.count
	ProvisionedCount int                       `json:"provisioned"`
}

// only used if hobbyfarm is doing the provisioning
type VirtualMachineProvision struct {
    // name of the vm
	VirtualMachineName string `json:"vm_name"`
    // state object
	TFControllerState  string `json:"tfc_state"`
    // configmap for the controller
	TFControllerCM     string `json:"tfc_cm"`
}
```

## VirtualMachineTemplate

A VirtualMachineTemplate defines a provider-agnostic representation of a virtual machine template. It is a generic approach that allows multiple [Environments](#environment) to implement the same vmtemplate, which provides admins the ability to schedule resources from one or more environments. 

VMTemplates do not specify AMIs or source images. Instead, it defines only an "image" and required resources. This allows providers to specifically implement these options as needed. 

For example, a vmtemplate such as "ubuntu2004-docker1" may exist, requiring 2 CPU, 4 GB RAM, and 20GB storage. For that vmtemplate, multiple providers could offer the ability to schedule those VMs. An environment in AWS may offer this by making available _t3.large_ instances using a source ubuntu AMI. An environment in DigitalOcean may offer this by making available _s-2vcpu-4gb_ droplets with a source ubuntu image.

A vmtemplate thus describes a standardized offering that multiple environments may implement in their own way. 

```go
type VirtualMachineTemplateSpec struct {
    // unique identifier, almost always matches the k8s object name
	Id        string            `json:"id"`
    // something unique but general. e.g. "ubuntu2004" or "centos7-4cpu-8gb"
	Name      string            `json:"name"`
    // the source image to be used. This is generic, not provider specific. Used only for informational purposes - implementation by environments may differ
	Image     string            `json:"image"`
    // required resources for the vm template
	Resources CMSStruct         `json:"resources"`
    // unused
	CountMap  map[string]string `json:"count_map"`
}

type CMSStruct struct {
	CPU     int `json:"cpu"`     // cores
	Memory  int `json:"memory"`  // in MB
	Storage int `json:"storage"` // in GB
}
```

## DynamicBindConfiguration

The DynamicBindConfiguration is similar to a [VirtualMachineSet](#virtualmachineset), except it is used for dynamic provisioning of VMs instead of static. A dbc controls how many VMs can be provisioned on-demand in an environment.

The original motivation behind the creation of this resource was that environments should have the ability to "burst", or add extra dynamic capacity. The reasoning was that sometimes events may occur where more attendees show up than were originally planned for. By having burst capacity available, those attendees could still receive a VM even if the static capacity of the environment had been exhausted. 

```go
type DynamicBindConfigurationSpec struct {
    // unique identifier, almost always matches the k8s object name
	Id                  string         `json:"id"`
    // the environment into which the vms should be provisioned
	Environment         string         `json:"environment"`
    // base name is a prefix for the created vms
    // for example, if base name is "testing-", generated vms will be 
    // something like "testing-09js98wd-89usefs"
	BaseName            string         `json:"base_name"`
    // whether or not the virtual machines in this set are for exclusive (restricted) binding only. false if it is a general use pool of vms
	RestrictedBind      bool   `json:"restricted_bind"`
    // the id of the scheduled event to which these virtual machines may belong
	RestrictedBindValue string `json:"restricted_bind_value"`
    // tells hobbyfarm what the capacity of this environment is for bursting
	BurstCountCapacity  map[string]int `json:"burst_count_capacity"`
    // tells hobbyfarm what the capacity of this environment is for bursting
    // described in cpu, memory, storage instead of vm count 
	BurstCapacity       CMSStruct      `json:"burst_capacity"`
}
```

## DynamicBindRequest

A DynamicBindRequest is a request for VMs to be dynamically bound by HobbyFarm. It is created in response to either `on_demand` being used for a [ScheduledEvent](#scheduledevent) or a [VirtualMachineClaim](#virtualmachineclaim) having used up all its static bind attempts. 

```go
type DynamicBindRequestSpec struct {
    // unique identifier, almost always matches the k8s object name
	Id                  string `json:"id"`
    // the id of the virtual machine claim from which this dbr was spawned
	VirtualMachineClaim string `json:"vm_claim"`
    // number of attempts to dynamically bind vms
	Attempts            int    `json:"attempts"`
}

type DynamicBindRequestStatus struct {
    // point-in-time indicator of how many attempts hobbyfarm has made to dynamically bind vms
	CurrentAttempts            int               `json:"current_attempts"`
    // dbrs carry an expiration date so that hobbyfarm does not attempt to bind vms into eternity
	Expired                    bool              `json:"expired"`
    // whether or not the request has been fulfilled
	Fulfilled                  bool              `json:"fulfilled"`
    // the dbc that governs this dbr
	DynamicBindConfigurationId string                                       `json:"dynamic_bind_configuration_id"`
    // ids of the virtual machines that have been bound by this dbr
	VirtualMachineIds          map[string]string `json:"virtual_machines_id"`
}
```