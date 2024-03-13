[//]: # (title: Configuring Test Reports and Code Coverage)
[//]: # (auxiliary-id: Configuring Test Reports and Code Coverage;Configuring Unit Testing and Code Coverage;Code Inspection;Code Coverage)

This section contains articles concerning support of test reports, code inspections, and code coverage in TeamCity.

## Test Reports in TeamCity

TeamCity provides out-of-the-box support for a number of testing frameworks. To reduce the feedback time on test failures, it reports [detailed test results](viewing-tests-and-configuration-problems.md) on the fly whenever possible. These results are displayed in the [build overview](working-with-build-results.md). See the list of currently [supported testing frameworks](testing-frameworks.md).

Many build runners produce test reports automatically, depending on the settings you configure: [Gradle](gradle.md), [Maven](maven.md), [IntelliJ IDEA Project](intellij-idea-project.md), [Python](python.md), [Node.js](nodejs.md), [NUnit](nunit.md) and others.

>You can [assign a user to investigate a failed test](investigating-and-muting-build-failures.md#Assigning+Investigations+of+Build+Problems+and+Failed+Tests), or [mute this test](investigating-and-muting-build-failures.md#Muting+Tests), so it doesn't affect the build status.

## Code Inspection in TeamCity

TeamCity comes with code analysis tools capable of inspecting your source code on the fly, finding and reporting common problems and anti-patterns.

The following inspections tools are bundled with TeamCity:
* [Inspections (IntelliJ IDEA)](inspections.md): runs code analysis based on [IntelliJ IDEA inspections](https://www.jetbrains.com/idea/documentation/inspections.jsp). More than 600 Java, HTML, CSS, JavaScript inspections are performed by this runner.
* [Inspections (ReSharper)](inspections-resharper.md): gathers results of [JetBrains ReSharper](https://www.jetbrains.com/resharper) [Code Analysis](https://www.jetbrains.com/resharper/webhelp/Code_Analysis__Index.html) in your C#, VB.NET, XAML, XML, ASP.NET, JavaScript, CSS, and HTML code.  
  Inspection results are reported in the __[Code Inspection](build-results-page.md#Code+Inspection+Tab)__ tab of the __Build Results__ page.
* [](qodana.md): runs static analysis checks, tracks how the code quality changes over time, detects incompatible dependency licenses, and much more. See the following Qodana help article for the detailed setup instructions: [Integration with TeamCity](https://www.jetbrains.com/help/qodana/teamcity.html).

TeamCity can also be [integrated with external reporting tools](how-to.md#Integrate+with+Build+and+Reporting+Tools).

## Code Coverage in TeamCity

Code coverage is a number of metrics that measure how your code is covered by unit tests. TeamCity supports the following coverage engines out of the box:

* __Java,__ see [Configuring Java Code Coverage](configuring-java-code-coverage.md)
  * [IntelliJ IDEA](intellij-idea.md) coverage (bundled)
  * [EMMA](http://emma.sourceforge.net/) open-source toolkit (bundled)
  * [JaCoCo](https://www.eclemma.org/jacoco/) open-source (bundled)

* __.NET__: see [Configuring .NET Code Coverage](configuring-.net-code-coverage.md)
  * [JetBrains dotCover](jetbrains-dotcover.md) (bundled)
  * [NCover](ncover.md)
  * [PartCover](partcover.md)

See how to [import reports](how-to.md#Integrate+with+Build+and+Reporting+Tools) or [coverage results](importing-arbitrary-coverage-results-to-teamcity.md) from other tools.

To get the code coverage information displayed in TeamCity for the supported tools, you need to configure it in the dedicated section of a [build runner's](build-runner.md) settings page. The following build runners include code coverage support:

* [Ant](ant.md)
* [IntelliJ IDEA Project](intellij-idea-project.md)
* [Maven](maven.md)
* [MSBuild](msbuild.md)
* [NAnt](nant.md)
* [NUnit](nunit.md)
* [MSpec](mspec.md)
* [.NET](net.md) (either stand-alone or combined with the [](dotcover-runner.md) runner)
* [Python](python.md)

Note that currently the Maven2 runner supports only [IntelliJ IDEA](intellij-idea.md) and [JaCoCo](jacoco.md) coverage engines.

The code coverage results can be viewed on the __Overview__ tab of __[Build Results](working-with-build-results.md)__. A detailed report is displayed on the dedicated __[Code Coverage](working-with-build-results.md)__ tab.

The chart for code coverage is also available on the __[Statistic Charts](statistic-charts.md)__ tab of a build configuration.

For the details on configuring code coverage, refer to the dedicated pages: [Configuring Java Code Coverage](configuring-java-code-coverage.md), [Configuring .NET Code Coverage](configuring-.net-code-coverage.md).
