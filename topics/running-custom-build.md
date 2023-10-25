[//]: # (title: Running Custom Build)
[//]: # (auxiliary-id: Running Custom Build;Triggering a Custom Build)

A build configuration usually has [build triggers](configuring-build-triggers.md) configured which automatically start a new build each time the conditions are met, like scheduled time or detection of VCS changes.

Besides triggering a build automatically, TeamCity allows you to run a build manually whenever you need, and customize this build by adding properties, using specific changes, running the build on a specific agent, and so on.

There are several ways of launching a custom build in TeamCity:
* Click the ellipsis on the __Run__ button, and specify the options in the __Run Custom Build__ dialog described [below](#General+Options).
* To run a custom build with specific changes, open the build results page, go to the __[Changes](build-results-page.md#Changes+Tab)__ tab, expand the required change, click the __Run build with this change__, and proceed with the [options](#General+Options) in the __Run Custom Build__ dialog.
* Use [HTTP request](accessing-server-by-http.md) or [REST API request](https://www.jetbrains.com/help/teamcity/rest/edit-build-configuration-settings.html#Manage+Build+Triggers) to TeamCity to trigger a build.
* [Promote a build](#Promoting+Build).
* [Build triggers](configuring-build-triggers.md) can launch builds with custom parameters.

## General Options

The **General** tab contains most basic and frequently used settings.

<img src="dk-customRun-general.png" width="706" alt="Run custom build dialog, General Settings tab"/>

### Agent

This setting allows you to choose an agent that should run your build. The following options are available:

* __&lt;the fastest idle agent&gt;__ (default option) — if selected, TeamCity will automatically choose an agent to run a build.

* Select a specific TeamCity agent from a list. TeamCity displays a selected agent's current state and estimates when this will become idle if it is already running a build.

* __&lt;the fastest idle agent in the N pool&gt;__ — TeamCity will run a build on an agent from a specified pool.

* if [cloud integration](teamcity-integration-with-cloud-solutions.md) is configured, you can select to run a build on an agent from a __certain cloud image__. If no available cloud agents of this type exist, TeamCity will also attempt to start a new one.
  {product="tc"}

* __&lt;All enabled compatible agents&gt;__ — run a build simultaneously on all agents that are enabled and compatible with the build configuration. This option may be useful in the following cases:
  * Run a build for agent maintenance purposes (for example, you can create a configuration to check whether agents function properly after an environment upgrade/update).
  * Run a build on different platforms (for example, you can set up a configuration, and specify for it a number of compatible build agents with different environments installed).


### Date &amp; Time

Leave the **As soon as possible** option to place a new build to a normal queue immediately after you click **Run Build**.

To schedule a build to the specific date &amp; time, switch to the **At specific date and time** option. Scheduled builds remain at the end of a [build queue](working-with-build-queue.md) until their scheduled date and time.

<img src="dk-scheduledBuild.png" width="706" alt="Scheduled build and time"/>

### Build Options


* **run as a personal build** — specifies whether this new build should run as a [personal](personal-build.md) one.

* **put the build to the queue top** — places this new build to the top of the current [build queue](working-with-build-queue.md). Since your newly started build can have no immediately ready compatible agents, it can move down the queue as it waits for one. If this happens, click the **Move to top** icon on the build configuration page, or navigate to the [](build-results-page.md) page and click **Actions | Move to top**.

  <img src="dk-moveBuildToTop.png" width="706" alt="Move queued build to top"/>

* **delete all files in the checkout directory before the build** — specifies whether TeamCity should clear the [build checkout directory](build-checkout-directory.md).
  * If snapshot dependencies are configured, this option can be applied to snapshot dependencies. In this case, all the builds of the build chain will be forced to use clean checkout.


<anchor name="P4-shelved-files-custom-run"/>

### Perforce-Specific Settings

If the current build configuration uses a [Perforce](perforce.md) VCS root, you can also run a custom build on [shelved files](https://www.perforce.com/manuals/v17.1/p4guide/Content/CmdRef/p4_shelve.html). To do this:
1. Enable _run as a personal build_ option.
2. Enter the ID of the changelist that contains the shelved files.
3. Choose the target Perforce root.

> If stream support is enabled in a Perforce VCS Root, TeamCity will automatically detect the target stream from the changed files even if the default stream is specified.
> 
{type="note"}


> Learn how to automate running builds on shelved files with [Perforce Shelve Trigger](perforce-shelve-trigger.md).
>
{type="tip"}

## Dependencies

_This tab is available only for builds that have dependencies on other builds_.   
You can enforce rebuilding of all dependencies and select a particular build whose artifacts will be taken. By default, the last 20 builds are displayed.

To increase the number of builds displayed in the drop-down menu to 50, use the `teamcity.runCustomBuild.buildsLimit=50` [internal property](server-startup-properties.md#TeamCity+Internal+Properties).
{product="tc"}

Note that if you re-run a dependent build, TeamCity will try to rebuild all dependency builds, including failed ones, by default.

By default, dependency builds in the list are grouped by their branches sorted alphabetically. Builds of the same branch are sorted relatively to each other by date. In some cases, you might need to discard branch-based sorting and sort all dependency builds only by their date, to display the newest builds at the top. To do this, click __Sort dependencies by date__. To return to the default sorting, click __Reset all__.

## Changes

_This tab is available only if you have permissions to access VCS roots for the build configuration._   
The tab allows you to specify a particular change to be included to the build.

The build will use the change's revision to check out the sources. That is, all the changes up to the selected one will be included into the build.   
Note that TeamCity displays only the changes detected earlier for the current build configuration VCS roots. If the VCS root was detached from the build configuration after the change occurred, there is no ability to run the build on such a change. A limited number of changes is displayed. If there is an earlier change in TeamCity that you need to run a build on, you can locate the change in the Change Log and use the __Run build with this change__ action.

## Include Changes

The __Include changes__ drop-down menu allows selecting the changes in the VCS roots attached to the configuration to run the build on.
* __Latest changes at the moment the build is started__: TeamCity will automatically include all changes available at the moment.
* __Last change to include__: When you select a change in the drop-down menu, TeamCity runs the build with the selected change and all changes that were made before it. The build run with the changes earlier than the latest available is marked as a [history build](history-build.md).

## Build Branch

The __Build branch__ drop-down menu, available if you have branches in your build configuration (or in snapshot dependencies of this build configuration), allows choosing a branch to be used for the custom build.

<anchor name="TriggeringCustomBuild-UsesettingsfromVCS"/>

## Use Settings

If your project has [versioned settings](storing-project-settings-in-version-control.md) enabled, you can tell TeamCity to run a build:
* With the settings defined for the project, either the current settings on the server or the settings from VCS.
* With the project settings currently defined on the server.
* With the settings loaded from the VCS revision calculated for the build.

If changes are selected in the [step above](#Include+Changes), the revision of the project settings corresponding to the selected changes will be loaded.

To define which settings to take, use one of the corresponding options from the _Use settings_ drop-down menu (the option here will override the [project-level setting](storing-project-settings-in-version-control.md#Defining+Settings+to+Apply+to+Builds)).

## Parameters

<note>
 
These settings are available only if you have permissions to change system properties and environment variables for the build configuration.

</note>

This tab allows adding, editing, and deleting new parameters/properties/variables, or redefining their [predefined values](predefined-build-parameters.md).   
When adding/editing/deleting properties and variables, note the following:
* For a predefined property/variable, only the value is editable.
* Only newly added properties/variables can be deleted. You cannot delete predefined properties.
* Each parameter value must be no longer than 16,000 characters.

## Comment and Tags

Add an optional comment as well as one or more [tags](build-actions.md#Add+Tags+to+Build) to the build. You can also add a custom build to [favorites](build-actions.md#Add+Build+to+Favorites) by checking the corresponding box in this section.

<note>

A greater build number does not mean more recent changes and the last build in the builds' history does not reflect the state of the latest project sources: builds in the builds' history are sorted by their start time, not by changes they include.
</note>

## Promoting Build

Build promotion refers to triggering a custom build with an overridden [artifact or snapshot dependency](dependent-build.md), i.e. manual launching of a build with dependencies configured, but using a build different from the build specified in the dependency.

To promote a build, open the build results page of the dependency build and click __Actions | Promote__.

For example, your build configuration A is configured to take artifacts from the last successful build of configuration B, but you want to run a build of configuration A using artifacts of a different build of configuration B (not the last successful build), so you promote an earlier build of B.   
Build promotion affects only a single run of the dependent build. Once you click __Promote__, a build of the dependent build configuration which uses the artifacts of the specified build is queued. Any further runs of the dependent build configuration will use artifacts as configured (last successful, last pinned, and so on), unless you use another promotion.

More details are available in the [related blog post](https://blog.jetbrains.com/teamcity/2012/04/teamcity-build-dependencies-2/).

 <seealso>
        <category ref="installation">
            <a href="upgrading-teamcity-server-and-agents.md" product="tc">Upgrade</a>
        </category>
        <category ref="concepts">
            <a href="working-with-build-queue.md">Working with Build Queue</a>
            <a href="dependent-build.md">Dependent Build</a>
            <a href="personal-build.md">Personal Build</a>
        </category>
        <category ref="admin-guide">
            <a href="configuring-build-triggers.md">Configuring Build Triggers</a>
        </category>
</seealso>
