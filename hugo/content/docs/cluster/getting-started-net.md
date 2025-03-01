---
layout: docs.hbs
title: Getting Started With Grains / Virtual Actors (.NET)
---

# Getting Started With Grains / Virtual Actors (.NET)

In this tutorial we will:

1. Model smart bulbs and a smart house using virtual actors/grains.
1. Run these grains in a cluster of members (nodes).
1. Send messages to and between these grains.
1. Host everything in a simple ASP.NET Core app.

The code from this tutorial is avaliable [on GitHub](https://github.com/asynkron/protoactor-grains-tutorial).

## Setting up the project

First things first, let's get the project setup and basic configuration out of the way, so we can later focus on grains and clustering.

### Required packages

Create an ASP.NET Core Web Application named `ProtoClusterTutorial`. For simplicity, this tutorial will use a Minimal API.

We'll need the following NuGet packages:

- `Proto.Actor`
- `Proto.Remote`
- `Proto.Cluster`
- `Proto.Cluster.CodeGen`
- `Proto.Cluster.TestProvider`
- `Grpc.Tools` - for compiling Protobuf messages

This tutorial was prepared using:

- .NET 6
- Proto.Actor 0.27.0 (all `Proto.*` packages share the same version number)
- `Grpc.Tools` 2.43.0

### Base web app

Let's establish what our base web app code should look like:

`Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);

var app = builder.Build();

app.MapGet("/", () => Task.FromResult("Hello, Proto.Cluster!"));

app.Run();
```

Try running your app to see if everything works so far.

### Basic Proto.Cluster infrastructure and configuration

First, we'll get the basic infrastructure of the cluster going.

We need to create, configure and register an `ActorSystem` instance. To keep it clean, we will create an `IServiceCollection` extension in another file to do that:

`ActorSystemConfiguration.cs`:

```csharp
using Proto;
using Proto.Cluster;
using Proto.Cluster.Partition;
using Proto.Cluster.Testing;
using Proto.DependencyInjection;
using Proto.Remote;
using Proto.Remote.GrpcNet;

namespace ProtoClusterTutorial;

public static class ActorSystemConfiguration
{
    public static void AddActorSystem(this IServiceCollection serviceCollection)
    {
        serviceCollection.AddSingleton(provider =>
        {
            // actor system configuration

            var actorSystemConfig = ActorSystemConfig
                .Setup();

            // remote configuration

            var remoteConfig = GrpcNetRemoteConfig
                .BindToLocalhost();

            // cluster configuration

            var clusterConfig = ClusterConfig
                .Setup(
                    clusterName: "ProtoClusterTutorial",
                    clusterProvider: new TestProvider(new TestProviderOptions(), new InMemAgent()),
                    identityLookup: new PartitionIdentityLookup()
                );

            // create the actor system

            return new ActorSystem(actorSystemConfig)
                .WithServiceProvider(provider)
                .WithRemote(remoteConfig)
                .WithCluster(clusterConfig);
        });
    }
}
```

Now we can register it in our web app:

`Program.cs`:

```csharp
builder.Services.AddActorSystem();
```

It is also suggested to turn on Proto.Actor logging. It will help with resolving all the issues. To do that it is needed to resolve `ILoggerFactory` dependency in `Program.cs` and use it for Proto.Actor logging config.

```csharp
...

var app = builder.Build();

var loggerFactory = app.Services.GetRequiredService<ILoggerFactory>();
Proto.Log.SetLoggerFactory(loggerFactory);

...

```

Let's go through each configuration section one by one:

#### Actor System configuration

This is a standard Proto.Actor configuration. It's out of the scope for this tutorial; if you want to learn more, you should check out the [Actors](../actors.md) section of Proto.Actor's documentation.

#### Remote configuration

Proto.Cluster uses Proto.Remote for transport, usually GRPC (Proto.Remote.GrpcNet). Again, its configuration is out of scope for this tutorial; if you want to learn more, you should check out the [Remote](../remote.md) section of Proto.Actor's documentation.

#### Cluster configuration

This is where we configure Proto.Cluster. Let's explain its parameters:

1. `clusterName` - any name will do.
1. `clusterProvider` - a Cluster Provider is an abstraction that provides information about currently available members (nodes) in a cluster. Since right now our cluster only has one member, it's ok to use a [Test Provider](test-provider-net.md).
   In the future, we will switch to other implementations, like [Consul Provider](consul-net.md) or [Kubernetes Provider](kubernetes-provider-net.md). You can read more about Cluster Providers [here](cluster-providers-net.md).
1. `identityLookup` - an Identity Lookup is an abstraction that allows a cluster to locate grains. `PartitionIdentityLookup` is generally a good choice for most cases. You can read more about Identity Lookup [here](identity-lookup-net.md).

### Cluster object

Most of the time we'll want to interact with the cluster, we will use a `Cluster` object. You can get it from an `ActorSystem` instance:

```csharp
using Proto;
using Proto.Cluster;

// ...

Cluster cluster = actorSystem.Cluster();
```

<!-- todo: link Cluster object documentation when available -->

### Starting a cluster member

Cluster members need to be explicitly started and shut down. You can do it in the following way:

```csharp
await _actorSystem
    .Cluster()
    .StartMemberAsync();

await _actorSystem
    .Cluster()
    .ShutdownAsync();
```

Since we're creating a web app, it's best if we start our cluster using a [hosted service](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-6.0&tabs=visual-studio):

`ActorSystemClusterHostedService.cs`:

```csharp
using Proto;
using Proto.Cluster;

namespace ProtoClusterTutorial;

public class ActorSystemClusterHostedService : IHostedService
{
    private readonly ActorSystem _actorSystem;

    public ActorSystemClusterHostedService(ActorSystem actorSystem)
    {
        _actorSystem = actorSystem;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("Starting a cluster member");

        await _actorSystem
            .Cluster()
            .StartMemberAsync();
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("Shutting down a cluster member");

        await _actorSystem
            .Cluster()
            .ShutdownAsync();
    }
}
```

Register the hosted service in our web app:

`Program.cs`:

```csharp
builder.Services.AddHostedService<ActorSystemClusterHostedService>();
```

At this point, our cluster is not doing much, but it won't hurt to run it and check if nothing breaks. You should see a `Starting a cluster member` line in your app's console.

## Creating a smart bulb grain

Now that we're done with the basic configuration, it's time to implement some features.

In this tutorial, we'll use grains to model smart bulbs. Their functionality will be as follows:

1. A smart bulb has a state, which is either: "unknown", "on" or "off".
1. Initially, smart bulb's state is "unknown".
1. We can turn a smart bulb on or off, which will write a message to a console.
1. Turning a smart bulb on when it's already on or turning it off when it's already off will not do anything.

### Virtual Actors / Grains

To avoid confusion, in this tutorial we'll refer to virtual actors as grains.

To recap:

1. Grains are essentially actors, meaning they will process messages one at a time.
1. Grains are not explicitly crated (activated). Instead, they are created when they receive the first message.
1. Each grain lives in _one_ of the cluster members.
1. Grain's location is transparent, meaning we don't need to know in which cluster member grain lives to call it.
1. Communication with grains should almost always be a request/response. <!-- todo: explain why? -->
1. Grains are identified by a _kind_ and _identity_, e.g. `airport`/`AMS` or `user`/`53`. It's important to distinguish kind/identity pair with an actor's ID, which in the case of grains might change between activations.

### Generating a grain

<!-- todo: link grains documentation page when it's created -->

The recommended way of creating a grain is by using a `Proto.Cluster.CodeGen` package, which generates most of the grain's code for use from a `.proto` file.

You can create it manually without that package, but:

1. It requires much more boilerplate code.
1. It's easy to make a mistake, e.g. respond to a message with a wrong type of message or not respond at all.

Read more about generating grains [here](codegen-net.md).

Let's create two `.proto` files: one for grains, and the other for messages used by these grains:

`Grains.proto`:

```protobuf
syntax = "proto3";

option csharp_namespace = "ProtoClusterTutorial";

import "google/protobuf/empty.proto";

service SmartBulbGrain {
  rpc TurnOn (google.protobuf.Empty) returns (google.protobuf.Empty);
  rpc TurnOff (google.protobuf.Empty) returns (google.protobuf.Empty);
}
```

In order for code generation to work (for both grains and messages), we need to handle them properly in the project file:

`ProtoClusterTutorial.csproj`

```xml
<ItemGroup>
    <ProtoGrain Include="Grains.proto" />
</ItemGroup>
```

`ProtoGrain` is an MSBuild task provided by `Proto.Cluster.CodeGen`.

This is a good moment to build a project and see if code generation doesn't produce any errors.

### Implementing a grain

If everything works correctly, we should implement our grain. `Proto.Cluster.Codegen` only created an abstract base class for our grain, so we need to implement it:

`SmartBulbGrain.cs`:

```csharp
using Proto;
using Proto.Cluster;

namespace ProtoClusterTutorial;

public class SmartBulbGrain : SmartBulbGrainBase
{
    private readonly ClusterIdentity _clusterIdentity;

    private enum SmartBulbState { Unknown, On, Off }
    private SmartBulbState _state = SmartBulbState.Unknown;

    public SmartBulbGrain(IContext context, ClusterIdentity clusterIdentity) : base(context)
    {
        _clusterIdentity = clusterIdentity;

        Console.WriteLine($"{_clusterIdentity.Identity}: created");
    }

    public override async Task TurnOn()
    {
        if (_state != SmartBulbState.On)
        {
            Console.WriteLine($"{_clusterIdentity.Identity}: turning smart bulb on");

            _state = SmartBulbState.On;
        }
    }

    public override async Task TurnOff()
    {
        if (_state != SmartBulbState.Off)
        {
            Console.WriteLine($"{_clusterIdentity.Identity}: turning smart bulb off");

            _state = SmartBulbState.Off;
        }
    }
}
```

<!-- todo: explain context and cluster identity -->

### Registering a grain

Remember, that grains are not explicitly activated, but only when they receive the first message. In other words, Proto.Cluster needs to know how to create new instances of your grains.
More specifically, they need be registered when configuring Cluster with a `WithClusterKind` method.

`ActorSystemConfiguration.cs`:

```csharp
var clusterConfig = ClusterConfig
    .Setup(
        clusterName: "ProtoClusterTutorial",
        clusterProvider: new TestProvider(new TestProviderOptions(), new InMemAgent()),
        identityLookup: new PartitionIdentityLookup()
    )
    .WithClusterKind(
        kind: SmartBulbGrainActor.Kind,
        prop: Props.FromProducer(() =>
            new SmartBulbGrainActor(
                (context, clusterIdentity) => new SmartBulbGrain(context, clusterIdentity)
            )
        )
    );
```

As with actors, we need to provide a `Props` describing how our grain is created.

`SmartBulbGrainActor` is another class generated by `Proto.Cluster.Codegen`, which is a wrapper for our grain code.

<!-- todo: more explanation? -->

## Communicating with grains

### Grain client

We can communicate with grains using a `Cluster` object:

```csharp
private readonly ActorSystem _actorSystem;

public async Task TurnTheLightOnInTheKitchen(CancellationToken ct)
{
    SmartBulbGrainClient smartBulbGrainClient = _actorSystem
        .Cluster()
        .GetSmartBulbGrain(identity: "kitchen");

    await smartBulbGrainClient.TurnOn(ct);
}
```

Both `GetSmartBulbGrain` extention method and `SmartBulbGrainClient` class were generated by `Proto.Cluster.Codegen`.

Mind, that `smartBulbGrainClient` is a client for a _specific_ grain, in this case, a smart bulb that's located in the kitchen.

### Smart bulb simulator

To see how grains behave in our system, we'll create a simulator that will send random messages to random smart bulbs.

We'll do that by creating another hosted service:

```csharp
using Proto;
using Proto.Cluster;

namespace ProtoClusterTutorial;

public class SmartBulbSimulator : BackgroundService
{
    private readonly ActorSystem _actorSystem;

    public SmartBulbSimulator(ActorSystem actorSystem)
    {
        _actorSystem = actorSystem;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var random = new Random();

        var lightBulbs = new[] { "living_room_1", "living_room_2", "bedroom", "kitchen" };

        while (!stoppingToken.IsCancellationRequested)
        {
            var randomIdentity = lightBulbs[random.Next(lightBulbs.Length)];

            var smartBulbGrainClient = _actorSystem
                .Cluster()
                .GetSmartBulbGrain(randomIdentity);

            if (random.Next(2) > 0)
            {
                await smartBulbGrainClient.TurnOn(stoppingToken);
            }
            else
            {
                await smartBulbGrainClient.TurnOff(stoppingToken);
            }

            await Task.Delay(TimeSpan.FromMilliseconds(500), stoppingToken);
        }
    }
}
```

Register the simulator in `Program.cs`:

```csharp
builder.Services.AddHostedService<SmartBulbSimulator>();
```

When you ran the app, you should see console output similar to:

```log
Starting a cluster member
smart bulb simulator: turning on smart bulb 'living_room_1'
living_room_1: created
living_room_1: turning smart bulb on
smart bulb simulator: turning on smart bulb 'bedroom'
bedroom: created
bedroom: turning smart bulb on
smart bulb simulator: turning on smart bulb 'living_room_2'
living_room_2: created
living_room_2: turning smart bulb on
smart bulb simulator: turning off smart bulb 'bedroom'
bedroom: turning smart bulb off
smart bulb simulator: turning on smart bulb 'living_room_2'
```

As you can see in the first few lines, a `living_room_1` grain is created only after a first message is sent to it.

## Using custom messages

Right now communication with our grain is quite simple: both `TurnOn` and `TurnOff` methods accept and return a predefined `google.protobuf.Empty` message. In this section, we will try to receive a custom message from a grain.

### Creating a custom message

Let's say we want to get a smart bulb's state. For simplicity, let's create a `GetSmartBulbStateResponse` that only contains a smart bulb's state:

`Messages.proto`:

```protobuf
syntax = "proto3";

option csharp_namespace = "ProtoClusterTutorial";

message GetSmartBulbStateResponse {
  string state = 1;
}
```

In a project file:

`ProtoClusterTutorial.csproj`

```xml
<ItemGroup>
    <Protobuf Include="Messages.proto" />
</ItemGroup>
```

### Importing a custom message

<!-- todo: either use "grain definition file" earlier of think of sth else -->

To use this message in a grain, we need to do three things:

1. Let `Proto.Cluster.CodeGen` know where to look for messages.
1. Import these messages in a `Grains.proto` file.
1. Register that message in `Proto.Remote`.

ad 1) We need to configure the `ProtoGrain` MSBuild task by adding `AdditionalImportDirs` attribute:

`ProtoClusterTutorial.csproj`

```xml
<ItemGroup>
    <ProtoGrain Include="Grains.proto" AdditionalImportDirs="." />
</ItemGroup>
```

ad 2) We need to add the following line to `Grains.proto`:

```protobuf
import "Messages.proto";
```

ad 3) We need to use `WithProtoMessages` on `Proto.Remote` configuration:

`ActorSystemConfiguration.cs`:

```csharp
using Proto.Remote;

// ...

// remote configuration

var remoteConfig = GrpcNetRemoteConfig
    .BindToLocalhost()
    .WithProtoMessages(MessagesReflection.Descriptor);
```

### Extending a grain

Let's add a new method to our grain. It should look like this:

`Grains.proto`:

```protobuf
syntax = "proto3";

option csharp_namespace = "ProtoClusterTutorial";

import "Messages.proto";
import "google/protobuf/empty.proto";

service SmartBulbGrain {
  rpc TurnOn (google.protobuf.Empty) returns (google.protobuf.Empty);
  rpc TurnOff (google.protobuf.Empty) returns (google.protobuf.Empty);
  rpc GetState (google.protobuf.Empty) returns (GetSmartBulbStateResponse);
}
```

Implement this method:

`SmartBulbGrain.cs`

```csharp
public override Task<GetSmartBulbStateResponse> GetState()
{
    return Task.FromResult(new GetSmartBulbStateResponse
    {
        State = _state.ToString()
    });
}
```

Let's create an API method to call it:

`Program.cs`

```csharp
app.MapGet("/smart-bulbs/{identity}", async (ActorSystem actorSystem, string identity) =>
{
    return await actorSystem
        .Cluster()
        .GetSmartBulbGrain(identity)
        .GetState(CancellationToken.None);
});
```

Run the app and try navigating to `/smart-bulbs/bedroom` in your browser. You should get results similar to the following:

```json
{ "state": "On" }
```

### Side note: grain activation

Let's use this moment to emphasise how grains work. Try navigating to: `/smart-bulbs/made-up-identity` or `/smart-bulbs/xyz123`. In both cases you should get:

```json
{ "state": "Unknown" }
```

Proto.Custer will activate any grain you send a message to, even the ones you haven't anticipated. Sometimes this might require some additional handling, e.g. checking if a given identity is valid, is present in some sort of a database, etc.
It's important to have this in the back of your head when designing a system using grains.

## Communicating between grains

To present how grains can communicate with each other, we'll create a new grain that will represent a smart house. It will be responsible for counting how many smart bulbs are on.
For simplicity, we'll assume there's only one smart house with identity `my-house`. Each bulb will report its status to this smart house when it changes.

### Creating a new grain

Create a definition for a new grain:

`Grains.proto`:

```protobuf
service SmartHouseGrain {
  rpc SmartBulbStateChanged (SmartBulbStateChangedRequest) returns (google.protobuf.Empty);
}
```

Define the `SmartBulbStateChangedRequest` message:

`Messages.proto`:

```protobuf
message SmartBulbStateChangedRequest {
  string smart_bulb_identity = 1;
  bool is_on = 2;
}
```

Implement the grain:

`SmartHouseGrain.cs`:

```csharp
using Proto;
using Proto.Cluster;

namespace ProtoClusterTutorial;

public class SmartHouseGrain : SmartHouseGrainBase
{
    private readonly ClusterIdentity _clusterIdentity;

    private readonly SortedSet<string> _turnedOnSmartBulbs = new();

    public SmartHouseGrain(IContext context, ClusterIdentity clusterIdentity) : base(context)
    {
        _clusterIdentity = clusterIdentity;

        Console.WriteLine($"{_clusterIdentity.Identity}: created");
    }

    public override Task SmartBulbStateChanged(SmartBulbStateChangedRequest request)
    {
        if (request.IsOn)
        {
            _turnedOnSmartBulbs.Add(request.SmartBulbIdentity);
        }
        else
        {
            _turnedOnSmartBulbs.Remove(request.SmartBulbIdentity);
        }

        Console.WriteLine($"{_clusterIdentity.Identity}: {_turnedOnSmartBulbs.Count} smart bulbs are on");

        return Task.CompletedTask;
    }
}
```

Register this grain in the cluster by calling another `WithClusterKind` on `ClusterConfig`:

`ActorSystemConfiguration.cs`:

```csharp
...
.WithClusterKind(
    kind: SmartHouseGrainActor.Kind,
    prop: Props.FromProducer(() =>
        new SmartHouseGrainActor(
            (context, clusterIdentity) => new SmartHouseGrain(context, clusterIdentity)
        )
    )
);
```

### Sending messages between grains

Again, to call a grain, we need to use a `Cluster` object. In a grain, we can get it from an `IContext` instance. In grains generated with `Proto.Cluster.CodeGen`, it's available as a `Context` property.

Modify the smart bulb grain accordingly:

`SmartBulbGrain.cs`:

```csharp
public override async Task TurnOn()
{
    if (_state != SmartBulbState.On)
    {
        Console.WriteLine($"{_clusterIdentity.Identity}: turning smart bulb on");

        _state = SmartBulbState.On;

        await NotifyHouse();
    }
}

public override async Task TurnOff()
{
    if (_state != SmartBulbState.Off)
    {
        Console.WriteLine($"{_clusterIdentity.Identity}: turning smart bulb off");

        _state = SmartBulbState.Off;

        await NotifyHouse();
    }
}

public override Task<GetSmartBulbStateResponse> GetState()
{
    return Task.FromResult(new GetSmartBulbStateResponse
    {
        State = _state.ToString()
    });
}

private async Task NotifyHouse()
{
    await Context
        .GetSmartHouseGrain("my-house")
        .SmartBulbStateChanged(
            new SmartBulbStateChangedRequest
            {
                SmartBulbIdentity = _clusterIdentity.Identity,
                IsOn = _state == SmartBulbState.On
            },
            CancellationToken.None
        );
}
```

Try running the app. You should see console output similar to:

```log
smart bulb simulator: turning off smart bulb 'living_room_2'
living_room_2: created
living_room_2: turning smart bulb off
my-house: created
my-house: 0 smart bulbs are on
smart bulb simulator: turning on smart bulb 'bedroom'
bedroom: created
bedroom: turning smart bulb on
my-house: 1 smart bulbs are on
smart bulb simulator: turning on smart bulb 'living_room_2'
living_room_2: turning smart bulb on
my-house: 2 smart bulbs are on
```

## Running a cluster with multiple members (nodes)

To showcase how grains work in a distributed context, we're going to run two members of our example app. Additionally, `SmartBulbSimulator` will be running as a separate application.

To do that, we'll need a proper Cluster Provider. To recap, a Cluster Provider is an abstraction that provides information about currently available members in a cluster.
In other words, it tells a cluster member what are the other members, thus allowing them to communicate with one another. You can read more about Cluster Providers [here](cluster-providers-net.md).

Until now, we've been using a [Test Provider](test-provider-net.md), which is only suited for running a single-member cluster.
To run a cluster with multiple members, we'll use a [Consul Provider](consul-net.md), which, like the name suggests, utilizes [HashiCorp Consul](https://www.consul.io/).

Let's also recap, how grains work. Each grain (i.e smart bulbs and a smart house) will live in one of the cluster members:


```mermaid
graph TB
a1{{SmartBulbGrain<br/>living_room_1}}
class a1 blue
a2{{SmartBulbGrain<br/>living_room_2}}
class a2 blue
a3{{SmartBulbGrain<br/>kitchen}}
class a3 blue

a4{{SmartBulbGrain<br/>bedroom}}
class a4 blue
a5{{SmartHouseGrain<br/>my-house}}
class a5 red

subgraph Member 2
    a4
    a5
end

subgraph Member1
    a1
    a2
    a3
end

a2-->a1
a2-->a3

a4-->a5

linkStyle default display:none;
```


### Consul provider

Now we can replace `TestProvider` with Consul provider.

First, we'll need to run Consul:

1. [Download Consul binaries here](https://www.consul.io/downloads).
1. Open a terminal and run the downloaded Consul binary in the development mode: `./consul agent -dev`

Let's now configure the [Consul Provider](consul-net.md) in our example.

Add `Proto.Cluster.Consul` NuGet package to the `ProtoClusterTutorial` project.

Change cluster member provider:

`ActorSystemConfiguration.cs`

```cs
var clusterConfig = ClusterConfig
    .Setup(
        clusterName: "ProtoClusterTutorial",
        clusterProvider: new ConsulProvider(new ConsulProviderConfig()),
        identityLookup: new PartitionIdentityLookup()
    )
    // ...
```

This will connect to Consul using a default port.

For more information on how to configure Consul Provider, read [the Consul provider documentation page](consul-net.md).

Run the `ProtoClusterTutorial` app to check if everything works so far. The app should not process any data since simulator is turned off.

### Running multiple members

Now we're ready to run multiple members. Start a terminal and navigate to the `ProtoClusterTutoral` project directory.

First, make sure your app is up to date:

```sh
dotnet build
```

Then we do the same for `SmartBulbSimulatorApp` project in the new terminal window:

```sh
dotnet build
```

Start the first member (in the first terminal):

```sh
dotnet run --no-build --urls "http://localhost:5161" 
```

At this point, the app shouldn't do much now, as the simulator is turned off.

Open a third terminal with `ProtoClusterTutoral` project directory, start the second member.


```bash
dotnet run --no-build --urls "http://localhost:5162"
dotnet run --no-build --urls "http://localhost:5161" ProtoRemotePort=5000 RunSimulation=false
```

After this we could observe in logs that cluster topology has changed but still the application is not doing much since simulator is off.

Back to the second terminal and run `SmartBulbSimulatorApp` app.

```bash
dotnet run --no-build --urls "http://localhost:5162" ProtoRemotePort=5001 RunSimulation=true
```

When you look at the console output, grains should be distributed between two members.

Sample output from the first member terminal:

```log
living_room_2: created
living_room_2: turning smart bulb off
bedroom: created
bedroom: turning smart bulb off
living_room_2: turning smart bulb on
living_room_2: turning smart bulb off
bedroom: turning smart bulb on
bedroom: turning smart bulb off
living_room_2: turning smart bulb on
bedroom: turning smart bulb on
...
```

Sample output from the second member terminal:

```log
living_room_1: created
living_room_1: turning smart bulb off
my-house: created
my-house: 0 smart bulbs are on
smart bulb simulator: turning off smart bulb 'living_room_1'
smart bulb simulator: turning off smart bulb 'living_room_2'
my-house: 0 smart bulbs are on
smart bulb simulator: turning off smart bulb 'kitchen'
kitchen: created
kitchen: turning smart bulb off
my-house: 0 smart bulbs are on
smart bulb simulator: turning off smart bulb 'living_room_1'
smart bulb simulator: turning on smart bulb 'kitchen'
kitchen: turning smart bulb on
my-house: 1 smart bulbs are on
smart bulb simulator: turning off smart bulb 'bedroom'
my-house: 1 smart bulbs are on
smart bulb simulator: turning on smart bulb 'living_room_2'
my-house: 2 smart bulbs are on
smart bulb simulator: turning on smart bulb 'living_room_1'
living_room_1: turning smart bulb on
my-house: 3 smart bulbs are on
...
```

This is a good opportunity to perform an experiment: turn off the first member. You should see, that all the grains from the first member should be recreated on the second member.

## Running application in Kubernetes

For now, tutorial showed how to run multiple members locally using Consul.
The same setup might be also suitable for some deployment cases, but since modern applications are more often deployed in Kubernetes it is better to select dedicated provider for it.

[Kubernetes provider](cluster/kubernetes-provider-net.md) is another implementation of `IClusterProvider` interface, the same as [Consul provider](cluster/consul-net.md). For more information you can check [Cluster providers section](cluster/cluster-providers-net.md).

### Changes in the application

First thing that needs to be done is to reference `Proto.Cluster.Kubernetes` package where this implementation is provided.
The next step is to replace Consul provider with Kubernetes provider in `ActorSystemConfiguration.cs`.

```csharp

...

var clusterConfig = ClusterConfig
    .Setup(
        clusterName: "ProtoClusterTutorial",
        clusterProvider: new KubernetesProvider(new Kubernetes(KubernetesClientConfiguration.InClusterConfig())),
        identityLookup: new PartitionIdentityLookup()
    )

```

It is also needed to change how remote configuration is prepared. We bind to all interfaces and use `ProtoActor:AdvertisedHost` host address passed in the configuration.

``` csharp

var remoteConfig = GrpcNetRemoteConfig
                    .BindToAllInterfaces(advertisedHost: configuration["ProtoActor:AdvertisedHost"])
                    .WithProtoMessages(MessagesReflection.Descriptor);
        
```

To have `configuration` variable in `AddActorSystem` extension it is needed to change its signature.

```csharp
public static void AddActorSystem(this IServiceCollection serviceCollection, IConfiguration configuration)
{
    ...
}
```

And usage in `ProtoClusterTutorial` `Program.cs`.

```csharp
builder.Services.AddActorSystem(builder.Configuration);
```

The same in the `SmartBulbSimulatorApp`

```csharp
services.AddActorSystem(hostContext.Configuration);
```

At this step both applications should be ready to run in Kubernetes, but first we need to create conainter images.

### Create docker images

To continue next steps it is needed to have container registry where the images will be pushed. In our tutorial we will use Azure Container Registry. You can find instructions how to create it [here](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-acr?tabs=azure-cli).

Add Dockerfile into `ProtoClusterTutorial` directory:

```Dockerfile

# Stage 1 - Build
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS builder

WORKDIR /app/src

# Restore
COPY *.csproj .

RUN dotnet restore -r linux-x64

# Build
COPY . .

RUN dotnet publish -c Release -o /app/publish -r linux-x64 --no-self-contained --no-restore

# Stage 2 - Publish
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app

RUN addgroup --system --gid 101 app \
    && adduser --system --ingroup app --uid 101 app


COPY --from=builder --chown=app:app /app/publish .

USER app
    
ENTRYPOINT ["./ProtoClusterTutorial"]

```

After this you should be able to build docker image for the `ProtoClusterTutorial` app with tag:

```sh

docker build . -t YOUR_ACR_ADDRESS/proto-cluster-tutorial:1.0.0

```

You need to add Dockerfile in the `SmartBulbSimulatorApp` directory too:

```Dockerfile

# Stage 1 - Build
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS builder

WORKDIR /app/src

# Restore
COPY /ProtoClusterTutorial/*.csproj ./ProtoClusterTutorial/
COPY /SmartBulbSimulatorApp/*.csproj ./SmartBulbSimulatorApp/


RUN dotnet restore ./SmartBulbSimulatorApp/SmartBulbSimulatorApp.csproj -r linux-x64

# Build
COPY . .

RUN dotnet publish ./SmartBulbSimulatorApp/SmartBulbSimulatorApp.csproj -c Release -o /app/publish -r linux-x64 --no-self-contained --no-restore

# Stage 2 - Publish
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app

RUN addgroup --system --gid 101 app \
    && adduser --system --ingroup app --uid 101 app


COPY --from=builder --chown=app:app /app/publish .

USER app
 
ENTRYPOINT ["./SmartBulbSimulatorApp"]

```

`SmartBulbSimulatorApp` relies on `ProtoClusterTutorial` sources so you need to run it from the main directory and pass Dockerfile as argument.

```sh

 docker build -f SmartBulbSimulatorApp/Dockerfile . -t YOUR_ACR_ADDRESS/smart-bulb-simulator:1.0.0

```

So now we created images for both applications and they should be visible on the images list:

```sh
docker images
```

Tip: If you encounter strange errors during building images then remove `obj` and `bin` directories. You can also consider adding `.dockerignore` file to skip them.

Both images should be pushed to our container registry:

```sh

docker push YOUR_ACR_ADDRESS/proto-cluster-tutorial:1.0.0

...

docker push YOUR_ACR_ADDRESS/smart-bulb-simulator:1.0.0

```

Now both images are stored in the container registry and we can start application deployment.

### Deployment to Kubernetes cluster

To continue next steps it is needed to have Kubernetes cluster running. In our tutorial we will use Azure Kubernetes Service. You can find instructions how to create it [here](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster?tabs=azure-cli).

To simplify the deployment to Kubernetes we will use [Helm](https://helm.sh/). Ensure that you have installed it locally and `helm` command is available. You can check how to do it [here](https://helm.sh/docs/intro/quickstart/)

Now we are going to prepare Helm chart that will help us with deployment.
To not create everything by hand you can download `chart-tutorial` [folder](https://github.com/asynkron/protoactor-grains-tutorial/tree/5351544702d4905948b45db97202dfb8290a2d25/chart-tutorial) from tutorial's repository on Github.

This chart contains definitions of given resources:

- deployment - the most important part is setting `ProtoActor__AdvertisedHost` variable based on pod's IP

``` yml
env:
    - name: ProtoActor__AdvertisedHost
    valueFrom:
      fieldRef:
        fieldPath: status.podIP

```

- service - to make each member reachable by another members

- role - permissions needed for [Kubernetes cluster provider](kubernetes-provider-net.md)

- service account - [identity in Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) that will be used by pod (Kubernetes provider)

- role binding - connection between role and service account

To continue with deployment you should open `values.yaml` and replace `member.image.repository` with your image. The same with `member.image.tag`. Key `member.replicaCount` determines the number of running replicas. By default it it 2.

After this, you need to open terminal in a chart's folder parent directory. Then you need to run `helm install` to deploy your image.

```sh
helm install proto-cluster-tutorial chart-tutorial
```

Where `proto-cluster-tutorial` is the name of the release and `chart-tutorial` is the name of a folder where the chart is located.

After this you should be able to see in the command line information that deployment is succeeded.

```sh
PS C:\repo\proto\protoTest> helm install proto-cluster-tutorial chart
NAME: proto-cluster-tutorial
LAST DEPLOYED: Tue Feb 15 11:29:48 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

You can also check that there are two running pods:

```sh
PS C:\repo\proto\protoTest> kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
proto-cluster-tutorial-886c9b657-5hgxh   1/1     Running   0          63m
proto-cluster-tutorial-886c9b657-vj8nj   1/1     Running   0          63m
```

If we will look closer to the created pods, we can see labels that were added by Kubernetes provider [here](https://github.com/asynkron/protoactor-dotnet/blob/dev/src/Proto.Cluster.Kubernetes/KubernetesProvider.cs#L114).

![pod-labels](images/pod-labels.png)

At this point these pods do nothing. Now it is needed to deploy simumaltor. We can reuse the same chart because simulator uses cluster client to send data to proto.actor cluster and requires similar permissions.
To do this we will call helm install as before but we will override values saved in `values.yaml` file. So first let's copy `values.yaml` file and rename it to `simulator-values.yaml`.

In the file we will change repository and tag to align with simulator image pushed to container registry. We also change `replicaCount` to 1 because we would like to have only single replica of the simulator.
Then we open again terminal in chart's parent directory and deploy simulator. We are adding `--values` parameter override `values.yaml` stored in chart's folder.

```sh
PS C:\repo\proto\protoactor-grains-tutorial> helm install simulator chart-tutorial --values .\simulator-values.yaml
NAME: simulator
LAST DEPLOYED: Tue Feb 15 13:24:48 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

We can also see that the pod has been deployed and we can see one more pod:

```sh
PS C:\repo\proto\protoactor-grains-tutorial> kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
proto-cluster-tutorial-886c9b657-5hgxh   1/1     Running   0          98m
proto-cluster-tutorial-886c9b657-vj8nj   1/1     Running   0          97m
simulator-68d4c5c4df-zgxv2               1/1     Running   0          6m22s
```

We can look into pods logs to see that both pods started processing data. We can do the same experiment as we did for Consul provider and scale down number of replicas to 1 to see that actors are recreated on the node that is still alive.

## Conclusion

Hopefully, at this point, you know how to build a cluster of grains using Proto.Actor. If you want to learn more, it's highly recommended, that you take a look at [the documentation](../_index.md), especially [the Cluster section](../cluster.md).

Thanks for your interest and good luck!
