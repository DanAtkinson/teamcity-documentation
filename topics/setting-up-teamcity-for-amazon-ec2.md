[//]: # (title: Setting Up TeamCity for Amazon EC2)
[//]: # (auxiliary-id: Setting Up TeamCity for Amazon EC2)

TeamCity Amazon EC2 integration allows you to configure TeamCity with your Amazon account and then start and stop images with TeamCity agents on-demand based on the queued builds.

Integrations with other cloud solutions are [available](teamcity-integration-with-cloud-solutions.md#Available+Integrations).

## General Description

It is assumed that the machine images are preconfigured to start TeamCity agent on boot (see details [below](#Preparing+Image+with+Installed+TeamCity+Agent)).

Once a cloud profile is configured in TeamCity with one or several images, TeamCity does a test start for all the new images to learn about the agents on them.  
Once the agents are connected, TeamCity stores their parameters to be able to process the build configurations-to-agents compatibility correctly.

For each queued build, TeamCity first tries to start it on one of the regular, non-cloud agents. If there are no usual agents available, TeamCity finds a matching cloud image with a compatible agent and starts a new instance for the image. TeamCity ensures that the running instances limit configured in the cloud profile is not exceeded.

Once an agent is connected from a cloud instance started by TeamCity, it is automatically authorized (provided there are available agent licenses). After that, the agent is processed as a regular agent.   
If running timeout is configured on the cloud profile and it is reached, the instance is terminated (or stopped, if an existing Amazon instance has been selected as an image source).

On the instance terminating/stopping, its disconnected agent is removed from the authorized agents list and is deleted from the system.

TeamCity supports Amazon EC2 [spot instances](#Amazon+EC2+Spot+Instances+support) and [spot fleets](#Amazon+EC2+Spot+Fleet+support).

## Configuration

Understanding Amazon EC2 and ability to perform EC2 tasks is a prerequisite for configuring and using TeamCity Amazon EC2 integration.

Basic TeamCity EC2 setup involves:
* preparing an Amazon EC2 image (AMI) with an installed TeamCity agent
* configuring EC2 integration on TeamCity server

<note>

Note that the number of EC2 agents is limited by the total number of agent licenses you have in TeamCity.

</note>

Make sure the server URL specified on the __Administration | Global Settings__ page is correct since agents will use it to connect to the server, if a custom server URL is not specified in the [cloud profile settings](agent-cloud-profile.md#Specifying+Profile+Settings).

### Required IAM permissions

TeamCity requires the following permissions for Amazon EC2 Resources:
* `ec2:Describe*`
* `ec2:StartInstances`
* `ec2:StopInstances`
* `ec2:TerminateInstances`
* `ec2:RebootInstances`
* `ec2:RunInstances`
* `ec2:ModifyInstanceAttribute`
* [`ec2:*Tags`](#Tagging+for+TeamCity-launched+instances)

To use [spot instances](#Amazon+EC2+Spot+Instances+support), grant the following permissions in addition to those listed above:

* `ec2:RequestSpotInstances`
* `ec2:CancelSpotInstanceRequests`
<!--* `ec2:GetSpotPlacementScores` (optional, allows TeamCity to choose AWS Regions or Availability Zones based on their [spot placement scores](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-placement-score.html#sps-example-configs)).-->

To use [spot fleets](#Amazon+EC2+Spot+Fleet+support), the following additional permissions are required:
* `ec2:RequestSpotFleet`
* `ec2:DescribeSpotFleetRequests`
* `ec2:CancelSpotFleetRequests`
* `iam:CreateServiceLinkedRole` (if you are getting _"The provided credentials do not have permission to create the service-linked role for EC2 Spot Fleet"_ error; you can safely revoke this permission once the service role is created)

To launch an [instance with the IAM Role](#Configuring+an+Amazon+EC2+cloud+profile) (applicable to instances cloned from AMIs and launch templates), the following additional permissions are required:
* `iam:ListInstanceProfiles`
* `iam:PassRole`

To use [encrypted EBS volumes](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSEncryption.html), the following additional permissions are required:
* `kms:CreateGrant`
* `kms:Decrypt`
* `kms:DescribeKey`
* `kms:GenerateDataKeyWithoutPlainText`
* `kms:ReEncrypt`

<!--To [connect to an agent via SSM](#debugging-and-maintenance), the following additional permissions are required:
* `sts:GetFederationToken`
* `ssm:*Session`-->

An example of custom IAM policy definition (allows all EC2 operations from a specified IP address):

```Shell
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "1",
            "Effect": "Allow",
            "Action": "ec2:*",
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": "<TeamCity server IP address>"
                }
            },
            "Resource": "*"
        }
    ]
}
```

#### Optional permissions

See the [section below](#Configuring+an+Amazon+EC2+cloud+profile) for permissions to set IAM roles on an agent instance.

View information on example policies for [Linux](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ExamplePolicies_EC2.html) and [Windows](http://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/ExamplePolicies_EC2.html) on the Amazon website.

### Preparing Image with Installed TeamCity Agent

Here are the requirements for an image that can be used for TeamCity cloud integration:
* TeamCity agent should be correctly [installed](install-and-start-teamcity-agents.md).
* TeamCity agent should start on machine startup.
* `buildAgent.properties` can be left as is. The `serverUrl`, `name`, and `authorizationToken` properties can be empty or set to any value; they are ignored when TeamCity starts the instance.

Provided these requirements are met, usual TeamCity agent installation and cloud-provider image bundling procedures are applicable.

If you need the [connection](install-and-start-teamcity-agents.md#Agent-Server+Data+Transfer) between the server and the agent machine to be secure, you will need to set up the agent machine to establish a secure tunnel (for example, VPN) to the server on boot so that TeamCity agent receives data via the secure channel.

Recommended image (for example, Amazon AMI) preparation steps:

1. Choose one of the existing generic images.
2. Start the image.
3. Configure the running instance:   
   * Install and configure a build agent:  
     * Configure server name and agent name in `conf/buildAgent.properties` — this is optional if the image will be started by TeamCity, but it is useful to test if the agent is configured correctly.
     * It usually makes sense to specify `tempDir` and `workDir` in `conf/buildAgent.properties` to use the non-system drive (`d:` under Windows).
   * Install any additional software necessary for the builds on the machine.
   * Run the agent and check it is working OK and is compatible with all necessary build configurations, and so on.
   * Configure the system so that the agent is started on machine boot (and make sure TeamCity server is accessible on machine boot).
   * For Windows, see also [Additional configuration for Windows agents](#Additional+configuration+for+Windows+agents)   
4. Test the setup by rebooting the machine and checking that the agent connects normally to the server.
5. Prepare the Image for bundling:    
   * Remove any temporary/history information in the system.
   * Stop the agent (under Windows stop the service but leave it in _Automatic_ startup type)
   * Delete the content `logs` and `temp` directories in agent home (optional)
   * Delete the `<Agent Home>/conf/amazon-*` file (optional)
   * Change `config/buildAgent.properties` to remove properties: `name`, `serverUrl`, `authorizationToken` (optional). `serverUrl` can be removed for EC2 integration plugins. Other plugins might require that it is present and set to correct value.    
6. Stop the running instance and make a new image from it (or just stop it if an existing Amazon instance has been selected as an image source).

#### Additional configuration for Windows agents

_To ensure proper TeamCity agent communication with EC2 API (including access to additional drives) on Windows_, add a dependency from the TeamCity Build Agent service on the [AmazonSSMAgent](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent.html) or [EC2Launch/EC2Config](http://docs.amazonwebservices.com/AWSEC2/latest/WindowsGuide/UsingConfig_WinAMI.html) service (the service which ensures the machine is fully initialized in regard to AWS infrastructure use). This can be done, for example, via the [Registry](https://support.microsoft.com/en-us/windows/how-to-open-registry-editor-in-windows-10-deab38e6-91d6-e0aa-4b7c-8878d9e07b11) or using [sc config](https://technet.microsoft.com/en-us/library/cc990290(v=ws.11).aspx) (for instance, `sc config TCBuildAgent depend=EC2Config`).   
Alternatively, you can use the "Automatic (delayed start)" service starting mode.

__Important note for images based on Windows Server 2016 image__:  
Due to the [bug in the network settings](https://forums.aws.amazon.com/thread.jspa?threadID=242194), instance metadata is not available by default. It means the TeamCity agent service cannot retrieve its properties and cloud integration doesn't work (the agent does not connect to the server or is not automatically authorized). To fix this issue, do the following:

1. Install the [latest EC2Config](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/UsingConfig_Install.html?shortFooter=true#ec2config-update-version).
2. Set the dependency of the `TCBuildAgent` service on both `EC2Config` and `AmazonSSMAgent` via the command: `sc config TCBuildAgent depend=Ec2Config/AmazonSSMAgent`.

### Agent auto-upgrade Note

TeamCity agent auto-upgrades whenever distribution of agent plugins on the server changes (for example, after TeamCity upgrade). If you want to cut agent startup time, you might want to rebundle the agent AMI after agent plugins have been auto-updated.

### Configuring an Amazon EC2 cloud profile

You can configure an Amazon EC2 [agent cloud profile](agent-cloud-profile.md) in the __Cloud Profiles__ section of the __Project Settings__.

#### Custom Image Name

The value of the _Custom Image Name_ parameter of the image serves as a unique identifier for all TeamCity agents using this image. For example, this value is added as a prefix to agents IDs in the cloud profiles log on the __Agents | Cloud__ tab.

Each image in a cloud profile must have a unique custom name.

#### IAM profiles

It is possible to use IAM (Identity and Access Management) profiles with build agents launched as Amazon EC2 instances, which requires the supplied AWS account to have the following permissions:
 
* `iam:ListInstanceProfiles`
* `iam:PassRole`

IAM profiles must be [preconfigured](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html) in Amazon EC2. In the TeamCity web UI, the __IAM profile__ drop-down menu enables you to select a role. Every new launched EC2 instance will assume the [selected IAM role](http://docs.aws.amazon.com/IAM/latest/UserGuide/roles-usingrole-ec2instance.html).

#### Amazon EC2 Spot Instances support

TeamCity supports [Amazon EC2 spot instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html) which allows you to place your bid on unused EC2 capacity and use it, as long as your suggested price exceeds the current _Spot price_.

You can enable spot instances for an Amazon EC2 image in your [cloud profile settings](agent-cloud-profile.md). Click __Add image__ and select a _Source_, or edit an existing image. Select the __Spot instances__ option, specify a __Bid price__, and save the image.   
If the bid price is not specified, the default On-Demand price will be used.

TeamCity launches a spot instance as soon as the cloud profile is saved. If it is impossible to launch the instance (for example, if there is no available capacity or if your bid price is lower than the current minimum Spot Price in Amazon), TeamCity will be repeating the launch attempt once a minute, as long as there are queued builds which can run on this agent.

If a spot instance is terminated, TeamCity will fail the build with a corresponding build problem, and will re-add this build to the build queue.

<!--TeamCity can automatically choose Regions or Availability Zones in which your spot requests are most likely to succeed based on their [spot placement scores](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-placement-score.html#sps-example-configs). To allow TeamCity request and utilize these scores, add the `ec2:GetSpotPlacementScores` [IAM permission](#Required+IAM+permissions).-->

<note>

It is not recommended to use spot instances for production-critical builds due to the possibility of an [unexpected spot instance termination by Amazon](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html#using-spot-instances-managing-interruptions).

</note>

#### Amazon EC2 Spot Fleet support

A _spot fleet_ is a set of [spot instances](#Amazon+EC2+Spot+Instances+support) launched based on the predefined criteria. You can request a spot fleet instead of regular spot instances, and TeamCity will be able to select and launch spot instances with the lowest price in the availability zones you specify.

To use a spot fleet in your project:

1. In [AWS Management Console](https://aws.amazon.com/console/), generate a spot fleet request:
   * Go to __Instances | Spot Requests | Request Spot Instances__.    
   * Specify the required AMI, minimum compute unit, availability zones, and other request details.
   * Click __JSON config__ to download a JSON file with the spot fleet configuration parameters. Note that only fields from the [`SpotFleetRequestConfigData`](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_SpotFleetRequestConfigData.html) class are supported.    

   <img src="SpotFleetConfigButton.png" width="600" alt="Spot fleet config generation button in Amazon"/>
   
2. In TeamCity, add a spot fleet to a cloud profile:
   * On the __Cloud Profile__ page click __Add image__.
   * In the _Source_ drop-down menu, select _Spot Instance Fleet_. Enter the maximum number of required spot instances (_Max instances_) in a fleet and select an _[agent pool](configuring-agent-pools.md)_.    
   * In the _Spot Fleet Config_ field, insert the JSON configuration file generated in AWS Management Console.    

   <img src="SpotFleetConfig.png" width="550" alt="Spot fleet configuration in TeamCity"/>
   
   * Save the image.
   
To run the image, TeamCity will launch spot instances matching the allocation strategy specified in the spot fleet configuration.

<note>

TeamCity uses own values instead of the following parameters of the JSON config file:
* `TerminateInstancesWithExpiration`: `false`
* `ValidFrom`: the time when the image is created
* `ValidUntil`: the `ValidFrom` value plus 5 minutes
* `Type`: `request`

</note>

#### Amazon EC2 Launch Templates support

TeamCity supports [Amazon EC2 launch templates](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-launch-templates.html) for cloud instances. 
Launch templates allow reusing a once defined launch specification for all new instances, which eliminates the need to describe the launch settings every time new instances are requested.

If your cloud profile is connected to the Amazon server, TeamCity will automatically detect launch templates available on this server. 
When adding an image, select a required template as the _Source_ and specify its version, and TeamCity will request instances based on the template parameters. 
You can use the same launch template to run various instances that differ in some parameters only. Check the **Customize Launch Template** box and modify the launch templates values as required.

Optionally, you can also limit the number of launched instances and assign them to a certain [agent pool](configuring-agent-pools.md).

When the default/latest version of the template is updated on the server, TeamCity will detect these changes and update the running instances.

#### Amazon EBS-Optimized Instances

The behavior of [EBS optimization](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSOptimized.html) in TeamCity is similar to that offered by EC2 console. When configuring an image of the Amazon [cloud profile](agent-cloud-profile.md), the optimization can be set using the corresponding box of the Instance Type. Note that
* EBS optimization is turned on by default for `c4.*`, `d2.*`, and `m4.*` (non-configurable)
* EBS optimization is turned off by default for any other instance types and  can be turned on for instances that support it (such as `c3.xlarge`, and so on)

#### Tagging for TeamCity-launched instances

The following requirements must be met for tagging instances launched by TeamCity:
* you have the `ec2:*Tags` permissions
* the [maximum number of tags (50)](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Using_Tags.html#tag-restrictions) for your Amazon EC2 resource is not reached

In the absence of tagging permissions, TeamCity will still launch Amazon AMI and EBS images with no tags applied; Amazon EC2 spot instances will not be launched.

TeamCity enables users to get instance launch information by marking the created instances with the `teamcity:TeamcityData` tag containing `<server UUID>:-<cloud profile ID>:-<image reference>`. __This tag is necessary for TeamCity integration with EC2 and must not be deleted.__

Custom tags can be applied to EC2 cloud agent instances: when configuring Cloud profile settings, in the __Add Image/Edit Image__ dialog use the __Instance tags__: field to specify tags in the format of `<key1>=<value1>,<key2>=<value2>`. [Amazon tag restrictions](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Using_Tags.html#tag-restrictions) need to be considered.

When using the equal(=) sign in the tag value, no escaping is needed. For instance, the string `extraParam=name=John` will be parsed into `<key=extraParam>` and value `<name=John>.`

#### Tagging instance-dependent resources

When launching Amazon EC2 instances, TeamCity tags all the resources (for example, volumes and network adapters) associated with the created instances, which is important when evaluating the overall cost of an instance (taking into account the storage drive type and size, I/O operations (for standard drives), network (transfers out), and so on.

#### Sharing single EBS instance between several TeamCity servers

As mentioned [above](#Tagging+for+TeamCity-launched+instances), TeamCity tags every instance it launches with the `teamcity:TeamcityData` tag that represents a server, cloud profile, and source (AMI or EBS\-instance). So, when several TeamCity servers try to use the same EBS instance, the second one will see the following message "Instance is used by another TeamCity server. Unable to start/stop it". If you are sure that no other TeamCity servers are working with this instance, you can delete the `teamcity:TeamcityData` tag and the instance will become available for all TeamCity servers again.

### Proxy settings
{product="tc"}

If your TeamCity server needs to use a proxy to connect to AWS API endpoint, configure the following server [internal properties](server-startup-properties.md#TeamCity+Internal+Properties) to connect to Amazon AWS addresses.

* `teamcity.http.proxy.host.ec2` — proxy server host name
* `teamcity.http.proxy.port.ec2` — proxy server port

For proxy server authentication: 

* `teamcity.http.proxy.user.ec2` — proxy access username 
* `teamcity.http.proxy.password.ec2` — proxy access user password

For NTML authentication: 
* `teamcity.http.proxy.domain.ec2` — proxy user domain for NTLM authentication 
* `teamcity.http.proxy.workstation.ec2` — proxy access workstation for NTLM authentication

### Debugging and Maintenance
{id="debugging-and-maintenance"}

You can investigate issues on the EC2 agent by launching an interactive browser-based shell directly from the TeamCity UI. Navigate to the required agent and click the **Open terminal** button to invoke this terminal. This method is supported for all local and cloud agents.

Learn more: [](install-and-start-teamcity-agents.md#Debug+Agents+Remotely).

<!--#### SSM Terminal

This method works only for EC2 agents with preinstalled [AWS Systems Manager Agent (SSM Agent)](https://docs.aws.amazon.com/systems-manager/latest/userguide/prereqs-ssm-agent.html). TeamCity provides authorization for the browser-based shell, so you don't have to manage it manually.

>By default, **the authorized** SSM **user will have sudo privileges**. For security reasons, it is recommended to change the default user permissions and launch the browser-based shell under the same OS user of your EC2 instance as the TeamCity build agent. You can find more information on how to do it below.
>
{type="warning"}


To configure this feature:

1. In TeamCity, check your EC2 [cloud profile](https://www.jetbrains.com/help/teamcity/setting-up-teamcity-for-amazon-ec2.html#Configuration):
   * Ensure that the credentials in the cloud profile have the following permissions:
     * `sts:GetFederationToken`
     * `ssm:*Session`
   * The agents launched from the cloud profile have an IAM role with the attached `AmazonSSMManagedInstanceCore` permission policy. This policy grants instances the permissions needed for core AWS Systems Manager functionality.
2. Check whether SSM Agent has been installed and is running on the instance. Most EC2 base images have [preinstalled SSM Agent](https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html). If you need to install SSM Agent manually, read the [documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-manual-agent-install.html).
3. Configure the privileges of the SSM user. By default, the user has sudo permissions, **which is not secure**. We strongly recommend launching a browser-based shell under the same OS user as the TeamCity build agent. In the AWS IAM role, specify the `SM Session Run As = OS user account name` tag. For more information, see the [documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-preferences-run-as.html).

   Additionally, to monitor session activity, set up a [logging policy](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-logging.html) for your EC2 instance in the AWS SSM service.

   > To use the feature, you need the _Open an interactive session to the agent_ permission.

4. In the TeamCity UI, open the Agent page, Miscellaneous section, and click **Connect via SSM**. The browser-based shell will open in a separate browser tab.

>To prevent the agent instance from terminating, you can [disable the agent for maintenance](). In maintenance mode, the agent will not be terminated automatically and not be available for new builds.
>
{type="tip"}-->

### Custom script

It is possible to run a custom script on the instance start (applicable to instances cloned from AMIs and launch templates).   
The Amazon website details the script format for [Linux](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) and [Windows](http://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/ec2-instance-metadata.html#user-data-execution).

### Estimating EC2 Costs

Usual Amazon EC2 pricing applies. Note that Amazon charges can depend on the specific configuration implemented to deploy TeamCity. We advise you to check your configuration and Amazon account data regularly in order to discover and prevent unexpected charges as early as possible.

Note that traffic volumes and necessary server and agent machines characteristics depend a big deal on the TeamCity setup and nature of the builds run. See also [Estimate Hardware Requirements for TeamCity](system-requirements.md#Estimating+External+Database+Capacity).
{product="tc"}

### Estimating Traffic

Here are some points to help you estimate TeamCity-related traffic:

* If TeamCity server is not located within the same EC2 region or availability zone that is configured in TeamCity EC2 settings for agents, traffic between the server and agent is subject to usual Amazon EC2 external traffic charges.
* When estimating traffic, bear in mind that there are lots types of traffic related to TeamCity (see a non-complete list below).

__External connections originated by server__:

* VCS servers
* Email servers
* Maven repositories

__Internal connections originated by server__:

* TeamCity agents (checking status, sending commands, retrieving information like thread dumps, and so on)

__External connections originated by agent__:
* VCS servers (in case of agent-side checkout)
* Maven repositories
* Any connections performed from the build process itself

__Internal connections originated by agent__:
* TeamCity server (retrieving build sources in case of server-side checkout or personal builds, downloading artifacts, and so on)

__Usual connections served by the server__:
* Web browsers
* IDE plugins

#### Uptime Costs

As Amazon rounds machine uptime to the full hour for some configurations (more at [How are Amazon EC2 instance hours billed?](https://aws.amazon.com/premiumsupport/knowledge-center/ec2-instance-hour-billing/)), adjust timeout setting on the EC2 image setting on TeamCity cloud integration settings according to your usual builds length.

It is also highly recommended to set execution timeout for all your builds so that a build hanging does not cause prolonged instance running with no payload.
