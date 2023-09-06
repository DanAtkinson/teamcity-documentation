[//]: # (title: Qodana)
[//]: # (auxiliary-id: Qodana)

The _Qodana_ build runner lets you add static analysis to your build chain. 
It is based on the [Qodana](https://www.jetbrains.com/help/qodana/teamcity.html) code quality monitoring platform.

<warning>

*Current limitations*
The Qodana runner requires TeamCity agents to have a Docker server installed and the TeamCity agents must not be deployed on a Windows OS.

</warning>

You can enable advanced code quality inspections and do the following:

- Run static analysis checks.
- Find duplicates in your code.
- Track how the code quality changes over time, and much more.

Refer to [Configuring Build Steps](configuring-build-steps.md) for a description of common build steps' settings. 
With Qodana, you can use flexible [build failure conditions](build-failure-conditions.md).

The _Qodana_ build runner provides exhaustive data about your code quality. You can:

- View an interactive build report.
- Assign [investigations](investigating-and-muting-build-failures.md) of the reported issues to the team members.
- Compare problems and checks applied between builds.
- View aggregated statistics for static code analysis metrics.

