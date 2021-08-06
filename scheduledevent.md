# Scheduled Events

> A ScheduledEvent defines a period of time over which course(s) or scenario(s) are made available to end users.
> 
> -- [ScheduledEvent](entities.md#scheduledevent)

Scheduled events in HobbyFarm are designed for period-of-time access to scenarios or courses. They are best used in combination with some sort of real-world event such as a training seminar. 

## Viewing Scheduled Events

Begin by logging in to the HobbyFarm admin user interface. Locate and click on the top-level navigation item labeled _Scheduled Events_. 

HobbyFarm presents a filtered list of Scheduled Events. This filtering removes any completed events from the list, only showing those events that are in the future or are currently happening. 

> To remove this filter, click on the filter icon next to the column title labeled _Status_

Each scheduled event offers two contextual actions, _Edit_ and _Delete_. These can be accessed by clicking the triple-dot menu in the first column of the table. 

## Creating a Scheduled Event

On the scheduled event listing page, click the _New_ button at the top. This will open a wizard that will walk you through the options for the new event. 

### Event Details

Provide the following details:

* _Name_: The name of the event.
* _Description_: A description of the event.
* _Access Code_: A string of characters that end-users will input to access the content. It is recommended to use something contextual such as `learning2020` or `chicagoseminar072021`.

* _Restricted Bind_: This setting constrains users to only using the virtual machines that are provisioned by this scheduled event. This is contrast to users being able to schedule virtual machines out of a general pool of VMs (if such a pool exists). Usually recommended to leave this enabled as scheduled events are designed to provision VMs specifically for users. 
* _On Demand_: This setting controls whether HobbyFarm will pre-provision virtual machines for your users, or spin them up on demand. When enabled, VMs will be created when a user creates a [Session](entities.md#session). When disabled, VMs are pre-provisioned according to the settings provided later in this wizard. 

### Event Times

_Start Time_ and _End Time_ are the times at which access to the scenario(s) and/or course(s) will begin and end. These times are defined in 30-minute increments via the user interface (although they are not limited that way via the API). 

When using static provisioning (i.e. _On Demand_ is turned off), the _Start Time_ is when HobbyFarm will begin provisioning VMs. If you wish your VMs to be ready before that time, you will want to move your _Start Time_ to an earlier selection. 

> When using the admin UI, all times are adjusted for your browser's timezone. These are then further adjusted to UTC before being persisted via the API.

### Select Course(s)

This stage is where you provide access to course(s) for your users. Selecting one or more courses will affect how many VMs are required, which will be viewed and adjusted in later stages of the wizard. 

### Select Scenario(s)

This stage is where you provide access to scenario(s) for your users. Selecting one or more scenarios will affect how many VMs are required, which will be viewed and adjusted in later stages of the wizard.

> It is required that you select **at least** one course or scenario in order to proceed.

### Select Environment(s)

This stage allows you to select the environments into which your virtual machines will be provisioned. One or more environments can be selected according to your needs. Environments that are shown will support _some number_ of the required VMs for your course(s) and/or scenario(s).

> Environments that do not support all the required virtual machine templates, or have no availability during the selected time period, will not be shown.

Make sure to take into account the required virtual machines for your course(s) and/or scenario(s) - availability may require you to schedule across multiple environments. 

### Select Virtual Machines

This stage in the wizard allows you to define how many users or virtual machines you wish to provision. There are two modes available for this stage: simple mode, and advanced. 

#### Advanced Mode

We start with an explanation of advanced mode to provide you with a better understanding of how VM scheduling works. In advanced mode (i.e. simple mode turned off), you are asked to provide a desired number of VMs in each environment. 

The user interfaces shows you the required number of VMs for each user. These figures are a result of the UI adding up the number of required VMs of each type for all the scenario(s) and/or course(s) you have selected. 


>_Example_
>
>Say we have three scenarios selected:
>1. Scenario one requires (2) VMs of the template `ubuntu-2004`. 
>2. Scenario two requires (3) VMs. Two are `ubuntu-2004` and one is `sles15-sp2`.
>3. Scenario three requires (1) VM of the template `alpine`.
>
>Thus, each user has the following required VMs: 
>  
>  * `ubuntu-2004`: 4
>  * `sles15-sp2`: 1
>  * `alpine`: 1


#### Simple Mode

In simple mode (turned on by default), administrators define their requested VMs in terms of expected attendees (end-users). For each user that an administrator defines, the number of required VMs increases multiplicatively. 

>_Example_
>
>In the previous section, a user requires four `ubuntu-2004` VMs, one `sles15-sp2` VM and one `alpine` VM. 
>
>Under simple mode, if an administrator defines one user in an environment, they are requesting four, one, and one VMs respectively. 
>
>If an administrator defines five users in an environment, they are requesting twenty, five, and five VMs respectively. 

Simple mode, like it sounds, is the easiest to use because you only have to consider how many attendees will be at your event. If you wish to provision nonstandard VMs or want to specifically define extra capacity for a certain pool of VMs, use advanced mode. 

### Finalize

This stage shows all the settings that were walked through in the wizard. Use it to ensure your settings are correct before clicking _Finish_ and creating your scheduled event. 