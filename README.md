# Existing Tool for Application Configuration Demo

## Introduction

Situation:
- need to provide different config for different environments.
- some configs might be common across environments.

Rough set up in this demo:
- Provider: Spring Cloud Config Server
  - takes care of reading/merging configs
- Consumer: Steeltoe
  - C# library (OSS by Pivotal)

- Proposed set up
  - decide and assign names to applications, eg. app1, app2.

    On the application side, we will specify this as a value in the respective app's appsettings.json (under `Spring.Application.Name`).

    In the config files side, we will write configs for an app (eg. app1) in a file of its same name (eg. app1.yml).

  - decide on environment names, eg. dev, qas, prod.

    On the application side, we will set ASPNETCORE_ENVIRONMENT to this value.

    In Spring Cloud Config, this corresponds to a "profile". We can place configs for different profiles in a single file (see app1.yml) - this works by default. We can also use a layout like `{profile}/{application}` where configs for a profile is placed in a subdirectory (see dev/qas/prod which contain app2.yml) - for filesystem as the "backend" for Spring Cloud Config, we'll need to add this pattern to the `spring.cloud.config.server.native.searchLocations` setting for Spring Cloud Config server to find them.

  - set up "backend" for config server - git repository, filesystem
  - tell app address of config server
  - usage in C# code:
    - inject `IConfiguration` from `Microsoft.Extensions.Configuration` (built-in), as usual - transparent to the app where the value is coming from

Advanced usage:
- Backends can be combined - eg. filesystem + Git repository.
- If Vault by Hashicorp is used for secrets, Spring Cloud Config server can use it as a backend too.
- No need to specify Spring Cloud Config server url if you use a DiscoveryClient implementation, such as Spring Cloud Netflix and Eureka Service Discovery.
- Spring Cloud Config server can monitor for changes in config (eg. pushes to Git repository, if Git repository is used as source of config) and send push notifications https://cloud.spring.io/spring-cloud-config/reference/html/#_push_notifications_and_spring_cloud_bus

## Provider: Spring Cloud Config Server

A central place to manage external properties for applications across all environments.

It exposes a RESTful API:

```
/{application}/{profile}
/{application}-{profile}.yml
/{application}-{profile}.json
/{application}-{profile}.properties
```

`application`: app1, app2
`profile`: dev, test

For demonstration purposes, we have used the "filesystem" type as the source of config:

```console
$ docker run \
  --publish 8888:8888 \
  --volume $(pwd)a-config-source:/config \
  steeltoeoss/config-server \
  --spring.profiles.active=native \
  --spring.cloud.config.server.native.searchLocations=file:///config,file:///config/{profile}
```


References:
- https://spring.io/guides/gs/centralized-configuration/



## Consumer: Steeltoe

What is Steeltoe?
- Supports writing cloud native applications/microservices on .NET
- .NET is supported on VMware Tanzu! https://docs.google.com/presentation/d/191zDItDYhWWrr37Va1g5LlPt9-jW6Iaeoi2Bb2fuTE4/edit#
- Coverage on MSDN Channel9: https://channel9.msdn.com/Shows/On-NET/NET-Microservices-with-Steeltoe
- a Nuget package, targets .NET Standard (hence runs on .NET Core and .NET Framework 4.x).


References:
- https://spring.io/guides/gs/centralized-configuration/
- https://steeltoe.io/app-configuration/get-started/springconfig

1. Install Steeltoe:

   ```console
   dotnet add package Steeltoe.Extensions.Configuration.ConfigServerCore
   ```

2. Add Steeltoe to app, pick up HostingEnvironment (ie. `ASPNETCORE_ENVIRONMENT`):

  ```diff
  diff --git a/Program.cs b/Program.cs
  --- a/Program.cs
  +++ b/Program.cs
  @@ -6,6 +6,6 @@ using Microsoft.AspNetCore.Hosting;
  using Microsoft.Extensions.Configuration;
  using Microsoft.Extensions.Hosting;
  using Microsoft.Extensions.Logging;
  +using Steeltoe.Extensions.Configuration.ConfigServer;^M

  namespace app1
  {
  diff --git a/Program.cs b/Program.cs
  index 223e9ed..15603e2 100644
  --- a/Program.cs
  +++ b/Program.cs
  @@ -14,4 +14,8 @@ namespace app1
          public static IHostBuilder CreateHostBuilder(string[] args) =>
              Host.CreateDefaultBuilder(args)
  +                .ConfigureAppConfiguration((context, config) =>
  +                {
  +                    config.AddConfigServer(context.HostingEnvironment);
  +                })
                  .ConfigureWebHostDefaults(webBuilder =>
                  {
  ```

3. Tell Steeltoe the name of the app, and where to find Spring Cloud Config:

  ```diff
  diff --git a/appsettings.json b/appsettings.json
  --- a/appsettings.json
  +++ b/appsettings.json
  @@ -1,1 +1,11 @@
  {
  +  "Spring": {
  +    "Application": {
  +      "Name": "app1"
  +    },
  +    "Cloud": {
  +      "Config": {
  +        "Uri": "http://localhost:8888"
  +      }
  +    }
  +  },
  ```

4. Run app with the ASPNETCORE_ENVIRONMENT environment variable set to corresponding environment, eg. dev/qas/prod. (If running with `dotnet run`, remember to specify `--no-launch-profiles` for this to take effect). See [.vscode/tasks.json](.vscode/tasks.json) for the demo apps launched.

  ```console
  # via dotnet run
  $ ASPNETCORE_ENVIRONMENT=dev dotnet run --project app1/app1.csproj urls=https://localhost:5050 --no-launch-profile
  $ ASPNETCORE_ENVIRONMENT=qas dotnet run --project app1/app1.csproj urls=https://localhost:5051 --no-launch-profile
  $ ASPNETCORE_ENVIRONMENT=dev dotnet run --project app2/app2.csproj urls=https://localhost:6060 --no-launch-profile
  $ ASPNETCORE_ENVIRONMENT=qas dotnet run --project app2/app2.csproj urls=https://localhost:6061 --no-launch-profile

  # if dotnet build has already been run
  $ ASPNETCORE_ENVIRONMENT=dev dotnet run app1/bin/Release/netcoreapp3.1/app1.dll urls=https://localhost:5050
  ```

# Potential FAQs

Q: Is it ok to set ASPNETCORE_ENVIRONMENT to dev/qas/prod? I thought you can only set it to Development/Staging/Production.

A: Development/Staging/Production are the defaults, and there is no limitation on what values it is set to:

> IHostEnvironment.EnvironmentName can be set to any value, but the following values are provided by the framework:
>
> Development : The launchSettings.json file sets ASPNETCORE_ENVIRONMENT to Development on the local machine.
> Staging
> Production : The default if DOTNET_ENVIRONMENT and ASPNETCORE_ENVIRONMENT have not been set.

Source: https://docs.microsoft.com/en-us/aspnet/core/fundamentals/environments

Q: I read about "labels" in Spring Cloud Config server. What is that?

A: For a Git repository as the source of configs, labels can correspond to a git label (commit id, branch name, or tag). They can be accessed via:

```
/{application}/{profile}[/{label}]
/{label}/{application}-{profile}.yml
/{label}/{application}-{profile}.json
/{label}/{application}-{profile}.properties
```