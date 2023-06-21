[//]: # (title: Agent Docker Images)

Instead of manually installing TeamCity agents and setting up required build software, you can do the following:

* Pull a required JetBrains "TeamCity Agent" Docker image. You can choose between a "minimal" (the basic agent image without any 3rd-party tools) and regular/full (bundled with multiple tools such as Git and .NET Runtime) Docker images.

* Execute the `docker run` command to start a container with a TeamCity agent running within.

> You can also start a TeamCity server inside a container. See instructions on this page for more information: [jetbrains/teamcity-server](https://hub.docker.com/r/jetbrains/teamcity-server).
> 
{type="tip"}

> Since not every 3rd-party tool vendor provides binaries for ARM processors, images for ARM devices have fewer installed components (when compared to images for the AMD x64 architecture). For instance, ARM-based images do not include Perforce CLI (p4).
> 
{type="note"}


|                                                                                                  | Regular Agent Image                                                                                           | Minimal Agent Image                                                                                                   |
|--------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Image registries and available tags                                                              | [Docker Hub](https://hub.docker.com/r/jetbrains/teamcity-agent/tags)                                          | [Docker Hub](https://hub.docker.com/r/jetbrains/teamcity-minimal-agent/tags)                                          |
| Installation instructions, limitations, and customization options                                | [Docker Hub](https://hub.docker.com/r/jetbrains/teamcity-agent)                                               | [Docker Hub](https://hub.docker.com/r/jetbrains/teamcity-minimal-agent)                                               |
| The list of components installed in agent images                                                 | [GitHub](https://github.com/JetBrains/teamcity-docker-images/blob/master/context/generated/teamcity-agent.md) | [GitHub](https://github.com/JetBrains/teamcity-docker-images/blob/master/context/generated/teamcity-minimal-agent.md) |
| Docker Compose samples<br/>(allow you to run a TeamCity server and agents in a single container) |                             [GitHub](https://github.com/JetBrains/teamcity-docker-samples)                    ||

