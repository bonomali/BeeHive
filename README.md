[![NuGet version](https://img.shields.io/nuget/v/BeeHive-common.svg)](https://www.nuget.org/packages/BeeHive-common/)
[![Build status](https://ci.appveyor.com/api/projects/status/7rj53liopu8lqy8n?svg=true)](https://ci.appveyor.com/project/aliostad/beehive)



![BeeHive](https://raw.githubusercontent.com/aliostad/PublicImages/master/diagrams/BeeHive-164px.png)

BeeHive
=======

A **Zero Friction** mini-Framework for cloud Actors - currently for Windows Azure only. Implementation is very simple - if you need a complex implementation of the Actor Model, you are probably doing it wrong. It supports **.NET Framework 4.5.2+ and .NET Standard 2.0+.**

This library helps implementing purely the business logic and the rest is taken care of.

Business process is broken down into a series/cascade/tree of events where each step only knows the event it is consuming and the event(s) it is publishing. Actors define the name of the queue they are supposed to read from (in the form of *QueueName* for Azure Service Bus Queues or *TopicName*-*SubscriptionName* for Azure Topics) and system will feed these processor actors via **Factory Actors**. 

Error handling and retry mechanism is under development.

### No-non-sense Microservices Actor Framework for the cloud

BeeHive makes it rediculously easy to build highly decoupled and evolvable event-based cloud systems.

For theoretical discussions, please refer to the [infoq article](http://www.infoq.com/articles/reactive-cloud-actors).

### Version 3.0.0 Breaking Changes and Notes
Version 3.0.0 features and changes include:
 - Supporting cross-platform development with .NET Core as well as supporting .NET Framework 4.5.2+ (`BeeHive.Azure` package depends on .NET 4.6.1+)
 - Upgrading all Azure dependencies (Storage and ServiceBus) to their latest version which removes some complexity and code from BeeHive since some features were not available in previous versions. For example, Service Bus now supports a polling queue/topic reader and keeps extending the lease, all of which were done as part of BeeHive itself.
 - There is now an Azure Function helper library which allows you to turn your actors to Azure Functions. This is particularly useful to move your existing actors to Azure Functions, but can equally be used for fresh projects. For more info on BeeHive and Azure Functions please see below.

## Getting Started

BeeHive makes it ridiculously *simple* (and not necessarily easy) to build decoupled actors that together achieve a business goal. By using topic-based subscription, you can easily add actors that feed on an existing event and do something extra.

Here we will build such systems will a few lines of code. You can find the full solution in the samples folder in the [source](https://github.com/aliostad/BeeHive). The code below is mainly is a snippet (e.g does not have the Dispose() methods for removing clutter).

## Scenario 1: News Feed Keyword Notification

Let's imagine you are interested in to know all breaking news from one or several news feeds and would like to be notified when a certain keyword occurs in the news. In this case we design a series of reactive actors that achieve this while allowing for other functionality to be built on top of existing topic based queues. We use Windows Azure for this example. In order to achieve this we need to regularly (e.g. every 5 minutes), check the news feed (RSS, ATOM, etc) and examine new items arrived and look for the interested keyword(s) and then perhaps tweet, send email or SMS text.

### Pulsers: Activities on regular intervals

BeeHive works on a reactive event-based model. Instead of building components that regularly do some work, we can have generic components that regularly fire an event. These events then can be subscribed to by one or more actors to fire off the processing by a chain of decoupled actors.

BeeHive **Pulsers** do exactly that. On regular intervals, they are woken up to fire off their events. Simplest of pulsers, are assembly attribute ones:

``` csharp
[assembly: SimpleAutoPulserDescription("NewsPulse", 5 * 60)]

```

The code above sets up a pulser that every 5 minutes sends an event of type `NewsPulse` with empty body.

### Dissemination from a list: Fan-out pattern

Next we set up an actor to look up a list of feeds (one per each line) and send an event per each feed. We have stored this list in a blob storage which is abstracted as `IKeyValueStore` in BeeHive.

``` csharp
[ActorDescription("NewsPulse-Capture")]
public class NewsPulseActor : IProcessorActor
{
  private IKeyValueStore _keyValueStore;
  public NewsPulseActor(IKeyValueStore keyValueStore)
  {
    _keyValueStore = _keyValueStore;
  }
  public async Task<IEnumerable<Event>> ProcessAsync(Event evnt)
  {
    var events = new List<Event>();
    var blob = await _keyValueStore.GetAsync("newsFeeds.txt");
    var reader = new StreamReader(blob.Body);
    string line = string.Empty;
    while ((line = reader.ReadLine())!=null)
    {
      if (!string.IsNullOrEmpty(line))
      {
        events.Add(new Event(new NewsFeedPulsed(){Url = line}));
      }
    }
    return events;
  }
}
```

`ActorDescription` attribute used above signifies that the actor will receive its events from the `Capture` subscription of the `NewsPulse` topic. We will be setting up all topics later using a single line of code.

So we are publishing `NewsFeedPulsed` event for each news feed. When we construct an `Event` object using the typed event instance, `EventType` and `QueueName` will be set to the type of the object passed - which is the preferred approach for consistency.

### Feed Capture: another Fan-out

Now we will be consuming these events in the next actor in the chain. `NewsFeedPulseActor` will subscribe to `NewsFeedPulsed` event and will use the URL to get the lastest RSS feed and look for latest news. To prevent from duplicate notifications, we need to know what was the most recent tem we checked last time. We will store this *offset* in a storage. For this use case, we choose `ICollectionStore<T>` which its Azure implementation uses Azure Table Storage.

``` csharp
[ActorDescription("NewsFeedPulsed-Capture")]
public class NewsFeedPulseActor : IProcessorActor
{
    private ICollectionStore<FeedChannel> _channelStore;
    public NewsFeedPulseActor(ICollectionStore<FeedChannel> channelStore)
    {
        _channelStore = channelStore;
    }
    public async Task<IEnumerable<Event>> ProcessAsync(Event evnt)
    {
      var newsFeedPulsed = evnt.GetBody<NewsFeedPulsed>();
      var client = new HttpClient();
      var stream = await client.GetStreamAsync(newsFeedPulsed.Url);
      var feedChannel = new FeedChannel(newsFeedPulsed.Url);
      var feed = SyndicationFeed.Load(XmlReader.Create(stream));
      var offset = DateTimeOffset.MinValue;
      if (await _channelStore.ExistsAsync(feedChannel.Id))
      {
        feedChannel = await _channelStore.GetAsync(feedChannel.Id);
        offset = feedChannel.LastOffset;
      }
      feedChannel.LastOffset = feed.Items.Max(x => x.PublishDate);
      await _channelStore.UpsertAsync(feedChannel);
      return feed.Items.OrderByDescending(x => x.PublishDate)
        .TakeWhile(y => offset < y.PublishDate)
        .Select(z => new Event(new NewsItemCaptured(){Item = z}));
    }
  }
}
```

Here we read the URL from the event and capture the RSS and then get the last offset from the strorage. We then send the captured feed items back as events for whoever is interested. At the end, we set the offset.

### Keyword filtering and Notification

At this stage we need to subscribe to `NewsItemCaptured` and check the content for specific keywords. This is only one potential subscription out of many. For example one actor could be subscribing to the event to store these for further retrieval, another to process for trend analysis, etc.

So for the sake of simplicity, let's hardcode the keyword but it could have been equally loaded from a storage or a file - as we did with the list of feeds.

``` csharp
[ActorDescription("NewsItemCaptured-KeywordCheck")]
public class NewsItemKeywordActor : IProcessorActor
{
    private const string Keyword = "Ukraine";
    public async Task<IEnumerable<Event>> ProcessAsync(Event evnt)
    {
        var newsItemCaptured = evnt.GetBody<NewsItemCaptured>();
        if (newsItemCaptured.Item.Title.Text.ToLower()
            .IndexOf(Keyword.ToLower()) >= 0)
        {
            return new Event[]
            {
                new Event(new NewsItemContainingKeywordIdentified()
                {
                    Item = newsItemCaptured.Item,
                    Keyword = Keyword
                }),
            };
        }
        return new Event[0];
    }
}

```
Now we can have several actors listening for `NewsItemContainingKeywordIdentified` and send different notifications, here we implement a simple Trace-based one:

``` csharp
[ActorDescription("NewsItemContainingKeywordIdentified-TraceNotification")]
public class TraceNotificationActor : IProcessorActor
{
  public async Task<IEnumerable<Event>> ProcessAsync(Event evnt)
  {
    var keywordIdentified = evnt.GetBody<NewsItemContainingKeywordIdentified>();
    Trace.TraceInformation("Found {0} in {1}",
        keywordIdentified.Keyword, keywordIdentified.Item.Links[0].Uri);
    return new Event[0];
  }
}
```
### Setting up the worker role

Using a Cloud Service worker role has been the standard means to host compute for BeeHive workloads. Having said that, BeeHive can be hosted in any compute model (including Service Fabric or even non-cloud) and also has built-in support for Azure Functions - see below.

If you have an Azure account, you need a storage account, Azure Service Bus and a worker role (even an *Extra Small* instance would suffice). If not, you can use development emulators although for the Service Bus you need to use Service Bus for windows. Just bear in mind, with local emulators and Service Bus for Windows, you have to use special versions of Azure SDK - latest versions usually do not work.

We can get a list of assembly pulsers by the code below:

``` csharp
_pulsers = Pulsers.FromAssembly(Assembly.GetExecutingAssembly())
    .ToList();
```

Also we need to create an `Orchestartor` to set up factory actors. We need to call `SetupAsync()` to set up all the topics and subscriptions.

Also we need to register our classes against Dependency Injection framework.

Now we are ready!

After running the application in debug mode, here is what you see in the output window:

![Found Ukraine](https://raw.githubusercontent.com/aliostad/BeeHive/master/assets/NewsFeedOutput.png)

Obviously, we can add another actor to subscribe to this event to send email, SMS messages, you name it. Point being, it is a piece of cake.

### Controlling actor parallelism by configuration

BeeHive by default looks for configuration values (both app.config and Azure config) `Beehive.ActorParallelism.<Actor Type Name>` and if it finds it, uses the count as the degree of parallelism. For example, this sets number of actors to 5 for class MyActor:

``` XML
  <add key="Beehive.ActorParallelism.MyActor" value="5" />
```

### Using BeeHive with Azure Functions
BeeHive provides an intuitive and unified programming model to build event-driven applications. With Azure Cloud Services now considered legacy and availability of Azure Functions as a cheaper compute model, it is only natural that BeeHive should support running actors in Azure Functions.

Now there is a question, why would you even need to use BeeHive? Well, reality is you don't. Having said that, even before Azure Functions, benefits of BeeHive was to provide a consistent programming model and abstractions for common distributed computing primitives such as Queues, Topics, Key Value stores, Collections, etc - nothing would prevent you to simply build directly on top of cloud features providing these primitives. And this has not changed with Azure Functions.

So if you have existing BeeHive actors to move to Azure Functions - or equally would like to build new system using BeeHive and Azure Functions - you simply design your actor as usual and add the dependency to the package `BeeHive.Azure.Functions` which contains an extension method for `IProcessorActor`. Then you would create a Service Bus Azure Function as below using the extension method to *direct your calls* to the *actor*:

``` csharp
[FunctionName("<name your function>")]
public static async Task My_Function(
    [ServiceBusTrigger("<topic>", "<subscription>", Connection = "<name of the setting for service bus connection>")] Message msg,
    ILogger log)
{
    var actor = new MyBeeHiveActor();
    await actor.Invoke(
      new QueueName("<topic>-<subscription>"), 
      msg, 
      new ServiceBusOperator(Environment.GetEnvironmentVariable("<name of the setting for service bus connection>")));
}      
```

Since in Azure Function all settings are loaded as environment variables, the class below sometimes could be useful if you had used Azure Configuration Management in the Cloud Service - you can replace it with below:

``` csharp
class EnvVarConfigProvider : IConfigurationValueProvider
{
    public string GetValue(string name)
    {
        return Environment.GetEnvironmentVariable(name);
    }
}
```

As for pulsers, you simply create a `TimerTrigger` and send a message to the topic - this is essentially what a pulser does - something like below (assuming `GetOperator()` creates a service bus operator):

``` csharp
[FunctionName("PulserExample")] // fires every 10 minutes
public async static Task My_Pulser_Function([TimerTrigger("0 1/10 * * * *")]TimerInfo info, ILogger log)
{
    await GetOperator().PushAsync(new Event("this content does not matter")
    {
        QueueName = QueueName.FromTopicName("<topic name>").ToString()
    });
}
```

For an example please refer to Demo project in samples and look at [PrismoEcommerce Azure Function project](https://github.com/aliostad/BeeHive/tree/master/demo/BeeHive.Demo.PrismoEcommerce.Functions).
