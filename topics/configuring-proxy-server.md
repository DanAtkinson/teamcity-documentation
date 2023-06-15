[//]: # (title: Configuring Proxy Server)
[//]: # (auxiliary-id: Configuring Proxy Server)

This article gives general recommendations on configuring the following proxy types: 
* [Reverse proxy installed in front of the TeamCity Server web UI](#Set+Up+TeamCity+Server+Behind+Proxy)
* [Proxy for outgoing connections](#Use+Proxy+for+Outgoing+Connections)
* [Proxy on the TeamCity agent side for agent-to-server connections](#Use+Proxy+to+Connect+Agents+to+TeamCity+Server)

## Set Up TeamCity Server Behind Proxy

>TeamCity now allows [configuring HTTPS access](https-server-settings.md) easily. These settings affect the built-in Tomcat server configuration and may break your existing proxy configuration.
>
>For large public facing TeamCity installations, it is still recommended that you set up TeamCity behind a proxy.



Consider the example:   
TeamCity server is installed at a _local_ URL [`http://teamcity.local:8111/tc`](http://teamcity.local:8111/tc){nullable="true"}.   
It is visible to the outside world as a _public_ URL [`http://teamcity.public:400/tc`](http://teamcity.public:400/tc){nullable="true"}.

To make sure TeamCity "knows" the actual absolute URLs used by the client to access the resources, you need to:
* Set up a reverse proxy as described below.
* Configure the [Tomcat server](#TeamCity+Tomcat+Configuration) bundled with TeamCity.

These URLs are used to generate absolute URLs in the client redirects and other responses.

After configuring the proxy, you will also need to change the _Server URL_ value in TeamCity __Global Settings__ to the proxy URL.

Note: An internal TeamCity server should work under the __same context__ (that is part of the URL after the host name) as it is visible from outside by an external address. See also the TeamCity server [context changing instructions](configure-server-installation.md#Changing+Server+Context).   
If you need to run the server under a different context, note that the context-changing proxy should conceal this fact from the TeamCity: for example, it should map server redirect URLs as well as cookies setting paths to the original (external) context.

The proxy should be configured with the generic web security in mind. Headers like `Referer` and `Origin` and all unknown headers should be passed to the TeamCity web application in the unmodified form. For example, TeamCity relies on the `X-TC-CSRF-Token` header added by the clients.

### Apache

Versions 2.4.5 or later are recommended. Earlier versions do not support the WebSocket protocol.

When using Apache, make sure to use the [Dedicated "Connector" node approach](#TeamCity+Tomcat+Configuration) for configuring TeamCity server.

```Shell

LoadModule  proxy_module          /usr/lib/apache2/modules/mod_proxy.so
LoadModule  proxy_http_module     /usr/lib/apache2/modules/mod_proxy_http.so
LoadModule  headers_module        /usr/lib/apache2/modules/mod_headers.so
LoadModule  proxy_wstunnel_module /usr/lib/apache2/modules/mod_proxy_wstunnel.so

ProxyRequests       Off
ProxyPreserveHost   On
ProxyPass           /tc/app/subscriptions ws://teamcity.local:8111/tc/app/subscriptions connectiontimeout=240 timeout=1200
ProxyPassReverse    /tc/app/subscriptions ws://teamcity.local:8111/tc/app/subscriptions

ProxyPass           /tc http://teamcity.local:8111/tc connectiontimeout=240 timeout=1200
ProxyPassReverse    /tc http://teamcity.local:8111/tc

```

Note the order of the [ProxyPass rules](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html#proxypass): conflicting ProxyPass rules must be sorted starting with the longest URLs first.

By default, Apache allows only a limited number of parallel connections that may be insufficient when using the WebSocket protocol. For instance, it may result in the TeamCity server not responding when a lot of clients open the web UI. To fix it, you may need to fine-tune the Apache configuration.

For example, on Unix, you should switch to [mpm_worker](http://httpd.apache.org/docs/2.4/mod/worker.html) and configure the maximum number of simultaneous connections:

```Shell
<IfModule mpm_worker_module>
    ServerLimit 100
    StartServers 3
    MinSpareThreads 25
    MaxSpareThreads 75
    ThreadLimit 64
    ThreadsPerChild 25
    MaxClients 2500
    MaxRequestsPerChild 0
</IfModule>

```

On Windows, you may need to increase the [ThreadsPerChild](http://httpd.apache.org/docs/2.4/mod/mpm_common.html#threadsperchild) value as described in the [Apache documentation](http://httpd.apache.org/docs/2.4/platform/windows.html).

### NGINX

Versions 1.3 or later are recommended. Earlier versions do not support the WebSocket protocol.


```Java

http {
    # ... default settings here
    proxy_read_timeout     1200;
    proxy_connect_timeout  240;
    client_max_body_size   0;    # maximum size of an HTTP request. 0 allows uploading large artifacts to TeamCity
 
    map $http_upgrade $connection_upgrade { # WebSocket support
        default upgrade;
        '' '';
    }
     
    server {
        listen       400; # public server port
        server_name  teamcity.public; # public server host name
     
        location / { # public context (should be the same as internal context)
            proxy_pass http://teamcity.local:8111; # full internal address
            proxy_http_version  1.1;
            proxy_set_header    Host $server_name:$server_port;
            proxy_set_header    X-Forwarded-Host $http_host;    # necessary for proper absolute redirects and TeamCity CSRF check
            proxy_set_header    X-Forwarded-Proto $scheme;
            proxy_set_header    X-Forwarded-For $remote_addr;
            proxy_set_header    Upgrade $http_upgrade; # WebSocket support
            proxy_set_header    Connection $connection_upgrade; # WebSocket support
        }
    }
}

```

[//]: # (Internal note. Do not delete. "How To...d160e1234.txt")

<anchor name="OtherServers"/>

### Other Servers

Make sure to use a performant proxy with due (high) limits on request (upload) and response (download) size and timeouts (at least tens of minutes and gigabyte, according to the sizes of the codebase and artifacts).

It is recommended to use a proxy capable of working with the WebSocket protocol as this helps the UI to refresh quicker.

Generally, you need to configure the TeamCity server so that it "knows" about the original URL used by the client and can generate correct absolute URLs accessible for the client. The preferred way to achieve that is to pass the original `Host` header to TeamCity. An alternative method is to set `X-Forwarded-Host` header to the original `Host` header's value.

Note that whenever the value of the `Host` header is changed by the proxy (while it is recommended to preserve the original `Host` header value) and no `X-Forwarded-Host` header with the original `Host` value is provided, the values of the `Origin` and `Referer` headers should be mapped correspondingly if they contain the original `Host` header value (if they do not, they should not be set in order not to circumvent TeamCity [CSRF protection](csrf-protection.md)).

Select a proper approach from the section below and configure the proxy accordingly.

### Common Misconfigurations

Check that your reverse proxy (or a similar tool) conforms to the following requirements:
* URLs with paths starting with a dot (`.`) are supported (path to hidden artifacts start contain the `.teamcity` directory).
* URLs with a colon (`:`) are supported (many TeamCity resources use the colon). Related [IIS setting](https://docs.microsoft.com/en-us/dotnet/api/system.web.configuration.httpruntimesection.requestpathinvalidcharacters?view=netframework-4.7.2). Symptom: build has no artifacts with the "_No user-defined artifacts in this build_" text even if there are artifacts.
* Settings that limit the maximum request name, response length, and response time are not too restrictive. See this article for more information: [](known-issues.md#IIS-Related+Issues).
* gzip Content-Encoding is fully supported. For example, certain IIS configurations can result in the "Loading data..." in the UI and 500 HTTP responses (see the related [issue](https://youtrack.jetbrains.com/issue/TW-56218)).

<anchor name="Proxy-Tomcat-Connector"/>
<anchor name="Proxy-Tomcat-RemoteIpValve"/>

## TeamCity Tomcat Configuration

For a TeamCity Tomcat configuration, use the __dedicated "Connector" node__ in the server configuration with hard-coded public URL details and make sure the port configured in the connector is used only by the requests to the public URL configured.

This approach can be used with any proxy configuration, provided the configured port is receiving requests only to the configured public URL.

You need to change the "Connector" node in `<[TeamCity Home](teamcity-home-directory.md)>/conf/server.xml` file as below.

```Shell

<Connector port="8111" protocol="org.apache.coyote.http11.Http11NioProtocol"
connectionTimeout="60000"
useBodyEncodingForURI="true"
socket.txBufSize="64000"
socket.rxBufSize="64000"
tcpNoDelay="1"
secure="false"
scheme="http"
/>
```

When the public server address is __HTTPS__, use the `secure="true"` and `scheme="https"` attributes. If these attributes are missing, TeamCity will show a respective health report.


[//]: # (Internal note. Do not delete. "How To...d160e1383.txt")

## Use Proxy for Outgoing Connections

This section describes configuring TeamCity to use a proxy server for outgoing HTTP connections. To connect TeamCity behind a proxy to Amazon EC2 cloud agents, see [this section](setting-up-teamcity-for-amazon-ec2.md#Proxy+settings).

A TeamCity server can use a proxy server for certain outgoing HTTP connections to other services like issues trackers.

To point TeamCity to your proxy server, set the following [internal properties](server-startup-properties.md#TeamCity+Internal+Properties):

```Shell

# For HTTP protocol
## The domain name or the IP address of the proxy host and the port:
teamcity.http.proxyHost=proxy.domain.com
teamcity.http.proxyPort=8080
 
## The hosts that should be accessed without going through the proxy, usually internal hosts. Provide a list of hosts, separated by the '|' character. The wildcard '*' can be used:
teamcity.http.nonProxyHosts=localhost|*.mydomain.com
 
## For an authenticated proxy add the following properties:
### Authentication type. "basic" and "ntlm" values are supported. The default is basic.
teamcity.http.proxyAuthenticationType=basic
### Login and Password for the proxy:
teamcity.http.proxyLogin=username
teamcity.http.proxyPassword=password
 
# For HTTPS protocol
## The domain name or the IP address of the proxy host and the port:
teamcity.https.proxyHost=proxy.domain.com
teamcity.https.proxyPort=8080
 
## The hosts that should be accessed without going through the proxy, usually internal hosts. Provide a list of hosts, separated by the '|' character. The wildcard '*' can be used:
teamcity.https.nonProxyHosts=localhost|*.mydomain.com
 
## For an authenticated proxy add the following properties:
### Authentication type. "basic" and "ntlm" values are supported. The default is basic.
teamcity.https.proxyAuthenticationType=basic
### Login and Password for the proxy:
teamcity.https.proxyLogin=login
teamcity.https.proxyPassword=password

```

> For external Git repositories, the aforementioned settings resolve connection issues only if you switch to the bundled JGit. If TeamCity uses the [native Git](git.md#Native+Git+for+VCS-related+operations+on+the+server), you need to manually setup proxy settings in your Git configuration. See the following articles for more information:
> 
> * [Configure Git to use a proxy](https://gist.github.com/evantoli/f8c23a37eb3558ab8765)
> * [The `git config` reference](https://git-scm.com/docs/git-config)
> 
{type="warning"}

## Use Proxy to Connect Agents to TeamCity Server

This section covers the configuration of a proxy server for TeamCity agent-to-server connections.

<chunk include-id="agent-proxy-server">

On the TeamCity agent side, specify the proxy to connect to the TeamCity server using the following properties in the [`buildAgent.properties`](configure-agent-installation.md) file:


```Shell

## The domain name or the IP address of the proxy host and the port
teamcity.http.proxyHost=123.45.678.9
teamcity.http.proxyPort=8080
 
## If the proxy requires authentication, specify the login and password
teamcity.http.proxyLogin=login
teamcity.http.proxyPassword=password
```

Note that the proxy has to be configured not to cache any TeamCity server responses. For example, if you use Squid, add "cache deny all" line to the `squid.conf` file.

</chunk>


## Proxy Server for Multinode Setup

The TeamCity server can be configured to use multiple nodes (or servers) for high availability and flexible load distribution. Refer to this article for the examples of NGINX and HAProxy configurations: [Multinode Setup for High Availability](multinode-setup.md#Proxy+Configuration).

<seealso>
        <category ref="admin-guide">
            <a href="multinode-setup.md#Proxy+Configuration">Configure proxy for a multinode setup</a>
        </category>
</seealso>
