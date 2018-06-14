
##### In this lab, you will create and host multiple child bots orchestrated by a master bot on top of Azure Service Fabric for OneBank Corp. Ltd. These child bots will serve different domains of a banking sector. 
You will see then how we leveraged Actor programming model to store the bot state reliably within the cluster itself.


## Excercise 1 : Host Bot

Since every service inside Azure Service Fabric is a console application, first we have to

**Task I :** Add OWIN Communication Listener

1. In Visual Studio Solution explorer, locate the `OneBank.Common` project and create a new C# class by Right-Clicking on the project.
2. Name this class as `OwinCommunicationListener` and replace the existing code with below class.

~~~csharp
namespace OneBank.Common
{
    using System;
    using System.Fabric;
    using System.Threading;
    using System.Threading.Tasks;
    using Autofac;
    using Microsoft.Owin.Hosting;
    using Microsoft.ServiceFabric.Services.Communication.Runtime;
    using Owin;

    public class OwinCommunicationListener : ICommunicationListener
    {
        private readonly string endpointName;

        private readonly ServiceContext serviceContext;

        private readonly Action<IAppBuilder> startup;

        private string listeningAddress;

        private string publishAddress;

        private IDisposable webAppHandle;

        public OwinCommunicationListener(Action<IAppBuilder> startup, ServiceContext serviceContext, string endpointName)
        {
            this.startup = startup ?? throw new ArgumentNullException(nameof(startup));
            this.serviceContext = serviceContext ?? throw new ArgumentNullException(nameof(serviceContext));
            this.endpointName = endpointName ?? throw new ArgumentNullException(nameof(endpointName));
        }

        public void Abort()
        {
            this.StopHosting();
        }

        public Task CloseAsync(CancellationToken cancellationToken)
        {
            this.StopHosting();
            return Task.FromResult(true);
        }

        public Task<string> OpenAsync(CancellationToken cancellationToken)
        {
            string ipAddress = FabricRuntime.GetNodeContext().IPAddressOrFQDN;
            var serviceEndpoint = this.serviceContext.CodePackageActivationContext.GetEndpoint(this.endpointName);
            var protocol = serviceEndpoint.Protocol;
            int port = serviceEndpoint.Port;

            if (this.serviceContext is StatefulServiceContext)
            {
                StatefulServiceContext statefulServiceContext = this.serviceContext as StatefulServiceContext;
                this.listeningAddress = $"{protocol}://+:{port}/{statefulServiceContext.PartitionId}/{statefulServiceContext.ReplicaId}/{Guid.NewGuid()}";
            }
            else if (this.serviceContext is StatelessServiceContext)
            {
                this.listeningAddress = $"{protocol}://+:{port}";
            }
            else
            {
                throw new InvalidOperationException();
            }

            this.publishAddress = this.listeningAddress.Replace("+", ipAddress);

            try
            {
                this.webAppHandle = WebApp.Start(this.listeningAddress, appBuilder => this.startup.Invoke(appBuilder));
                return Task.FromResult(this.publishAddress);
            }
            catch (Exception)
            {
                this.StopHosting();
                throw;
            }
        }

        private void StopHosting()
        {
            if (this.webAppHandle != null)
            {
                try
                {
                    this.webAppHandle.Dispose();
                }
                catch (ObjectDisposedException)
                {
                }
            }
        }
    }
}
~~~

3. In Visual Studio Solution explorer, locate the `OneBank.MasterBot` project and double click on `MasterBot.cs` file.
4. Find the method `CreateServiceInstanceListeners`, and replace the definition with following code.

~~~csharp
protected override IEnumerable<ServiceInstanceListener> CreateServiceInstanceListeners()
{
    var endpoints = this.Context.CodePackageActivationContext.GetEndpoints()
                            .Where(endpoint => endpoint.Protocol == EndpointProtocol.Http || endpoint.Protocol == EndpointProtocol.Https)
                            .Select(endpoint => endpoint.Name);

    return endpoints.Select(endpoint => new ServiceInstanceListener(
        context => new OwinCommunicationListener(Startup.ConfigureApp, this.Context, endpoint), endpoint));
}
~~~

5. In `OneBank.MasterBot` project, locate the `ServiceManifest.xml` file and add an HTTP endpoint inside the `<Endpoints>` element 

~~~xml
<Resources>
    <Endpoints>
      <Endpoint Name="ServiceEndpoint" Type="Input" Protocol="http" Port="8770" />
    </Endpoints>
</Resources>
~~~

> Notice the `Type` and `Port` of the Master bot endpoint. These values should be different for all child bots as shown in the next step

6. Similarly, locate the `OneBank.AccountsBot`project and double click on AccountsBot.cs file.
7. Find the method `CreateServiceInstanceListeners`, and replace the definition with following code.

~~~csharp
protected override IEnumerable<ServiceInstanceListener> CreateServiceInstanceListeners()
{
    var endpoints = this.Context.CodePackageActivationContext.GetEndpoints()
                            .Where(endpoint => endpoint.Protocol == EndpointProtocol.Http || endpoint.Protocol == EndpointProtocol.Https)
                            .Select(endpoint => endpoint.Name);

    return endpoints.Select(endpoint => new ServiceInstanceListener(
        context => new OwinCommunicationListener(Startup.ConfigureApp, this.Context, endpoint), endpoint));
}
~~~

8. In `OneBank.AccountsBot` project, locate the `ServiceManifest.xml` file and add an HTTP endpoint inside the `<Endpoints>` element

~~~xml
<Resources>
    <Endpoints>
      <Endpoint Name="ServiceEndpoint" Type="Internal" Protocol="http" Port="8771" />
    </Endpoints>
</Resources>
~~~

9. Again, you would do the same for the InsuranceBot by replacing the definition for `CreateServiceInstanceListeners` with following code

~~~csharp
protected override IEnumerable<ServiceInstanceListener> CreateServiceInstanceListeners()
{
    var endpoints = this.Context.CodePackageActivationContext.GetEndpoints()
                            .Where(endpoint => endpoint.Protocol == EndpointProtocol.Http || endpoint.Protocol == EndpointProtocol.Https)
                            .Select(endpoint => endpoint.Name);

    return endpoints.Select(endpoint => new ServiceInstanceListener(
        context => new OwinCommunicationListener(Startup.ConfigureApp, this.Context, endpoint), endpoint));
}
~~~
  
10. And then, add an endpoint in the ServiceManifest.xml file under `<Endpoints>` element

~~~xml
<Resources>
    <Endpoints>
      <Endpoint Name="ServiceEndpoint" Type="Internal" Protocol="http" Port="8772" />
    </Endpoints>
</Resources>
~~~

> Http port for the AccountsBot & InsuranceBot must be different than MasterBot. Also the `Type` should also be `Internal` so that you don't expose the child bots directly outside of the Service Fabric cluster. Only MasterBot should be exposed to a publicily accessible endpoint.

**Task II :** Create a basic master root dialog.

1. In `OneBank.MasterBot` project, locate the `Dialogs` folder, and add a new C# class.
2. Name this class as `MasterRootDialog` and replace the existing code with below class.

~~~csharp
using Microsoft.Bot.Builder.Dialogs;
using Microsoft.Bot.Connector;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace OneBank.MasterBot.Dialogs
{
    [Serializable]
    public class MasterRootDialog : IDialog<object>
    {
        public Task StartAsync(IDialogContext context)
        {
            context.Wait(this.MessageReceivedAsync);
            return Task.CompletedTask;
        }

        public async Task MessageReceivedAsync(IDialogContext context, IAwaitable<IMessageActivity> result)
        {
            await context.PostAsync("Hello there! Welcome to OneBank.");
            await context.PostAsync("I am the Master bot");

            PromptDialog.Choice(context, ResumeAfterChoiceSelection, new List<string>() { "Account Management", "Buy Insurance" }, "What would you like to do today?");           
        }

        private async Task ResumeAfterChoiceSelection(IDialogContext context, IAwaitable<string> result)
        {
            var choice = await result;

            if (choice.Equals("Account Management", StringComparison.OrdinalIgnoreCase))
            {
                await context.PostAsync("Forward me to AccountsBot");
            }
            else if (choice.Equals("Buy Insurance", StringComparison.OrdinalIgnoreCase))
            {
                await context.PostAsync("Forward me to InsuranceBot");
            }
            else
            {
                context.Done(1);
            }
        }
    }
}
~~~

**Task III:** Observe the application by running it.

1. On top of the Visual Studio, click on `Start` button to run the application.
    >Please make sure the start project in the Solution must be `OneBank.FabricApp`.

    >As you are running the application for the first time, it may take a couple of minutes to boot up the cluster.

![startApp]

2. A pop-up may appear to seek permission to Refresh Application on the cluster. Click `Yes`

![refreshApp]

3. Navigate to Desktop, and double click on Bot Framework Emulator

![startBotEmulator]

4. Set the URL of the MasterBot in the Address bar. The URL must be `http://localhost:8770/api/messages`. And click `Connect`

![setBotUrl]

5. Type in `Hi` and see the response. Thats how it should ideally look like.

![sayHi]

### Excercise 2 : Forward request to child bots

**Task 1:** Add a new class in Common project and name it as HttpCommunicationClient, then copy/paste this code

~~~csharp
using Newtonsoft.Json.Linq;
using System;
using System.Collections.Generic;
using System.Fabric;
using System.Linq;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;

namespace OneBank.Common
{
    public class HttpCommunicationClient : ICommunicationClient
    {
        public HttpCommunicationClient(HttpClient httpClient)
        {
            this.HttpClient = httpClient;
        }

        public HttpClient HttpClient { get; private set; }

        public ResolvedServicePartition ResolvedServicePartition { get; set; }

        public string ListenerName { get; set; }

        public ResolvedServiceEndpoint Endpoint { get; set; }

        public string HttpEndPoint
        {
            get
            {
                JObject addresses = JObject.Parse(this.Endpoint.Address);
                return (string)addresses["Endpoints"].First();
            }
        }
    }
}
~~~
**Task 2:** Add an empty interface by the name of IHttpCommunicationClientFactory

~~~csharp
namespace Gorenje.DA.Fabric.Communication.HttpCommunication
{
    using Microsoft.ServiceFabric.Services.Communication.Client;

    public interface IHttpCommunicationClientFactory : ICommunicationClientFactory<HttpCommunicationClient>
    {
    }
}
~~~
**Task 3:** Add the HttpCommunicationClientFactory class
~~~csharp
using Microsoft.ServiceFabric.Services.Communication.Client;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.Http;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

namespace OneBank.Common
{
    [Serializable]
    public class HttpCommunicationClientFactory : CommunicationClientFactoryBase<HttpCommunicationClient>, IHttpCommunicationClientFactory
    {
        private readonly HttpClient httpClient;

        public HttpCommunicationClientFactory(HttpClient httpClient)
        {
            this.httpClient = httpClient;
        }

        protected override void AbortClient(HttpCommunicationClient client)
        {
        }

        protected override Task<HttpCommunicationClient> CreateClientAsync(string endpoint, CancellationToken cancellationToken)
        {
            return Task.FromResult(new HttpCommunicationClient(this.httpClient));
        }

        protected override bool ValidateClient(HttpCommunicationClient client)
        {
            return true;
        }

        protected override bool ValidateClient(string endpoint, HttpCommunicationClient client)
        {
            return true;
        }
    }
}

~~~

**Task 4:** Register the communication client in the Startup class of the master bot

~~~csharp
Conversation.UpdateContainer(
                builder =>
                {
                    builder.Register(c => new HttpCommunicationClientFactory(new HttpClient()))
                     .As<IHttpCommunicationClientFactory>().SingleInstance();
                });

            config.DependencyResolver = new AutofacWebApiDependencyResolver(Conversation.Container);
~~~

**Task 5** Add this method in the MasterRootDialog class
~~~csharp
public async Task<HttpResponseMessage> ForwardToChildBot(string serviceName, string path, object model, IDictionary<string, string> headers = null)
        {
            var clientFactory = Conversation.Container.Resolve<IHttpCommunicationClientFactory>();
            var client = new ServicePartitionClient<HttpCommunicationClient>(clientFactory, new Uri(serviceName));

            HttpResponseMessage response = null;

            await client.InvokeWithRetry(async x =>
            {
                var targetRequest = new HttpRequestMessage
                {
                    Method = HttpMethod.Post,
                    Content = new StringContent(JsonConvert.SerializeObject(model), Encoding.UTF8, "application/json"),
                    RequestUri = new Uri($"{x.HttpEndPoint}/{path}")
                };

                if (headers != null)
                {
                    foreach (var key in headers.Keys)
                    {
                        targetRequest.Headers.Add(key, headers[key]);
                    }
                }

                response = await x.HttpClient.SendAsync(targetRequest);
            });

            return response;
        }
~~~ 
**Task 6:** Replace the line 

~~~charp
await context.PostAsync("Forward me to AccountsBot"); 
~~~

with
~~~csharp
await ForwardToChildBot("fabric:/OneBank.FabricApp/OneBank.AccountsBot", "api/messages", context.Activity);
~~~

**Task 6:** Replace the line 

~~~charp
await context.PostAsync("Forward me to InsuranceBot"); 
~~~

with
~~~csharp
await ForwardToChildBot("fabric:/OneBank.FabricApp/OneBank.InsuranceBot", "api/messages", context.Activity);
~~~
**Task 7:** Add the EchoDialog in AccountsBot

~~~csharp
using Microsoft.Bot.Builder.Dialogs;
using Microsoft.Bot.Connector;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace OneBank.AccountsBot.Dialogs
{
    [Serializable]
    public class AccountsEchoDialog
    {
        private int count = 1;

        public async Task StartAsync(IDialogContext context)
        {
            await Task.CompletedTask;
            context.Wait(this.MessageReceivedAsync);
        }

        public async Task MessageReceivedAsync(IDialogContext context, IAwaitable<IMessageActivity> argument)
        {
            var message = await argument;

            await context.PostAsync($"[From AccountsBot] - You said {message.Text} - Count {count++}");
            context.Wait(this.MessageReceivedAsync);
        }
    }
}

~~~
**Task 8:** Call the echo dialog from the Controllers Post method.
~~~csharp
await Conversation.SendAsync(activity, () => new AccountsEchoDialog());
~~~

**Task 9:** Add the EchoDialog in InsuranceBot

~~~csharp
using Microsoft.Bot.Builder.Dialogs;
using Microsoft.Bot.Connector;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace OneBank.AccountsBot.Dialogs
{
    [Serializable]
    public class InsuranceEchoDialog
    {
        private int count = 1;

        public async Task StartAsync(IDialogContext context)
        {
            await Task.CompletedTask;
            context.Wait(this.MessageReceivedAsync);
        }

        public async Task MessageReceivedAsync(IDialogContext context, IAwaitable<IMessageActivity> argument)
        {
            var message = await argument;

            await context.PostAsync($"[From InsuranceBot] - You said {message.Text} - Count {count++}");
            context.Wait(this.MessageReceivedAsync);
        }
    }
}

~~~
**Task 8:** Call the echo dialog from the Controllers Post method.
~~~csharp
await Conversation.SendAsync(activity, () => new InsuranceEchoDialog());
~~~

**Task 9:** Lets rerun the bot again and lets see what happens this time

![botStateError]

Excercise 3 : Service Fabric Bot State

**Task 1** In IBotStateActor add the following methods

~~~csharp
Task<BotStateContext> GetBotStateAsync(string key, CancellationToken cancellationToken);

Task<BotStateContext> SaveBotStateAsync(string key, BotStateContext dialogState, CancellationToken cancellationToken);

Task InsertBotStateAsync(string key, BotStateContext dialogState, CancellationToken cancellationToken);

Task<bool> DeleteBotStateAsync(string key, CancellationToken cancellationToken);
~~~

**Task 3** Create a new class to store the bot state context

~~~csharp
using System;

namespace OneBank.BotStateActor
{
    [Serializable]
    public class BotStateContext
    {
        public string BotId { get; set; }

        public string UserId { get; set; }

        public string ChannelId { get; set; }

        public string ConversationId { get; set; }

        public DateTime TimeStamp { get; set; }

        public byte[] Data { get; set; }

        public string ETag { get; set; }
    }
}

~~~

**Task 3** In BotStateActor.cs class add the following methods with thier definition

~~~csharp
public async Task<BotStateContext> GetBotStateAsync(string key, CancellationToken cancellationToken)
{
    ActorEventSource.Current.ActorMessage(this, $"Getting bot state from actor key - {key}");
    ConditionalValue<BotStateContext> result = await this.StateManager.TryGetStateAsync<BotStateContext>(key, cancellationToken);

    if (result.HasValue)
    {
        return result.Value;
    }
    else
    {
        return null;
    }
}

public async Task<BotStateContext> SaveBotStateAsync(string key, BotStateContext dialogState, CancellationToken cancellationToken)
{
    ActorEventSource.Current.ActorMessage(this, $"Adding bot state for actor key - {key}");
    return await this.StateManager.AddOrUpdateStateAsync(
        key,
        dialogState,
        (k, v) =>
            {
                return dialogState.ETag != "*" && dialogState.ETag != v.ETag ? throw new Exception() : v = dialogState;
            },
        cancellationToken);
}

public async Task InsertBotStateAsync(string key, BotStateContext dialogState, CancellationToken cancellationToken)
{
    ActorEventSource.Current.ActorMessage(this, $"Inserting bot state for actor key - {key}");
    await this.StateManager.AddStateAsync(key, dialogState, cancellationToken);
}

public async Task<bool> DeleteBotStateAsync(string key, CancellationToken cancellationToken)
{
    ActorEventSource.Current.ActorMessage(this, $"Deleting bot state for actor key - {key}");
    return await this.StateManager.TryRemoveStateAsync(key, cancellationToken);
}
~~~

**Task 4** Create a new class in ServiceFabricBotDataStore in common project
~~~csharp
namespace Gorenje.DA.Common.StateManager
{
    using System;
    using System.IO;
    using System.IO.Compression;
    using System.Threading;
    using System.Threading.Tasks;
    using Microsoft.Bot.Builder.Dialogs;
    using Microsoft.Bot.Builder.Dialogs.Internals;
    using Microsoft.Bot.Connector;
    using Microsoft.ServiceFabric.Actors;
    using Microsoft.ServiceFabric.Actors.Client;
    using Newtonsoft.Json;
    using OneBank.BotStateActor;
    using OneBank.BotStateActor.Interfaces;
    using OneBank.Common;

    public class ServiceFabricBotDataStore : IBotDataStore<BotData>
    {
        private static readonly JsonSerializerSettings SerializationSettings = new JsonSerializerSettings()
        {
            Formatting = Formatting.None,
            NullValueHandling = NullValueHandling.Ignore
        };

        private readonly string botName;

        public ServiceFabricBotDataStore(string botName)
        {
            this.botName = botName;
        }

        public async Task<bool> FlushAsync(IAddress key, CancellationToken cancellationToken)
        {
            return await Task.FromResult(true);
        }

        public async Task<BotData> LoadAsync(IAddress key, BotStoreType botStoreType, CancellationToken cancellationToken)
        {
            var botStateActor = this.GetActorInstance(key.UserId, key.ChannelId);
            BotStateContext botStateContext = await botStateActor.GetBotStateAsync(this.GetStateKey(key, botStoreType), cancellationToken);

            if (botStateContext != null)
            {
                return new BotData(botStateContext.ETag, Deserialize(botStateContext.Data));
            }
            else
            {
                return new BotData(string.Empty, null);
            }
        }

        public async Task SaveAsync(IAddress key, BotStoreType botStoreType, BotData data, CancellationToken cancellationToken)
        {
            var stateKey = this.GetStateKey(key, botStoreType);

            BotStateContext botStateContext = new BotStateContext
            {
                BotId = key.BotId,
                ChannelId = key.ChannelId,
                ConversationId = key.ConversationId,
                UserId = key.UserId,
                Data = Serialize(data.Data),
                ETag = data.ETag,
                TimeStamp = DateTime.UtcNow
            };

            var botStateActor = this.GetActorInstance(key.UserId, key.ChannelId);

            if (string.IsNullOrEmpty(botStateContext.ETag))
            {
                botStateContext.ETag = Guid.NewGuid().ToString();
                await botStateActor.SaveBotStateAsync(stateKey, botStateContext, cancellationToken);
            }
            else if (botStateContext.ETag == "*")
            {
                if (botStateContext.Data != null)
                {
                    await botStateActor.SaveBotStateAsync(stateKey, botStateContext, cancellationToken);
                }
                else
                {
                    await botStateActor.DeleteBotStateAsync(stateKey, cancellationToken);
                }
            }
            else
            {
                if (botStateContext.Data != null)
                {
                    await botStateActor.SaveBotStateAsync(stateKey, botStateContext, cancellationToken);
                }
                else
                {
                    await botStateActor.DeleteBotStateAsync(stateKey, cancellationToken);
                }
            }
        }

        private static byte[] Serialize(object data)
        {
            using (var cmpStream = new MemoryStream())
            using (var stream = new GZipStream(cmpStream, CompressionMode.Compress))
            using (var streamWriter = new StreamWriter(stream))
            {
                var serializedJSon = JsonConvert.SerializeObject(data, SerializationSettings);
                streamWriter.Write(serializedJSon);
                streamWriter.Close();
                stream.Close();
                return cmpStream.ToArray();
            }
        }

        private static object Deserialize(byte[] bytes)
        {
            using (var stream = new MemoryStream(bytes))
            using (var gz = new GZipStream(stream, CompressionMode.Decompress))
            using (var streamReader = new StreamReader(gz))
            {
                return JsonConvert.DeserializeObject(streamReader.ReadToEnd());
            }
        }

        private IBotStateActor GetActorInstance(string userId, string channelId)
        {
            return ActorProxy.Create<IBotStateActor>(new ActorId($"{userId}-{channelId}"), new Uri("fabric:/OneBank.FabricApp/BotStateActorService"));
        }

        private string GetStateKey(IAddress key, BotStoreType botStoreType)
        {
            switch (botStoreType)
            {
                case BotStoreType.BotConversationData:
                    return $"{this.botName}:{key.ChannelId}:conversation:{key.ConversationId}";

                case BotStoreType.BotUserData:
                    return $"{this.botName}:{key.ChannelId}:user:{key.ConversationId}";

                case BotStoreType.BotPrivateConversationData:
                    return $"{this.botName}:{key.ChannelId}:private:{key.ConversationId}:{key.UserId}";

                default:
                    throw new ArgumentException("Unsupported bot store type!");
            }
        }
    }
}
~~~
**Task 5** Register bot state in master bot
~~~csharp
var store = new ServiceFabricBotDataStore("Master");
                    builder.Register(c => new CachingBotDataStore(store, CachingBotDataStoreConsistencyPolicy.LastWriteWins))
                        .As<IBotDataStore<BotData>>()
                        .AsSelf()
                        .InstancePerLifetimeScope();
~~~ 
**Task 5** Register bot state in Accounts bot
~~~csharp
Conversation.UpdateContainer(
                builder =>
                {
                    var store = new ServiceFabricBotDataStore("Accounts");
                    builder.Register(c => new CachingBotDataStore(store, CachingBotDataStoreConsistencyPolicy.LastWriteWins))
                        .As<IBotDataStore<BotData>>()
                        .AsSelf()
                        .InstancePerLifetimeScope();
                });

            config.DependencyResolver = new AutofacWebApiDependencyResolver(Conversation.Container);
~~~

**Task 5** Register bot state in Insurance  bot
~~~csharp
Conversation.UpdateContainer(
                builder =>
                {
                    var store = new ServiceFabricBotDataStore("Insurance");
                    builder.Register(c => new CachingBotDataStore(store, CachingBotDataStoreConsistencyPolicy.LastWriteWins))
                        .As<IBotDataStore<BotData>>()
                        .AsSelf()
                        .InstancePerLifetimeScope();
                });

            config.DependencyResolver = new AutofacWebApiDependencyResolver(Conversation.Container);
~~~
**Task 6** Sticky bot state
Replace the ResumeAfterChoiceSelection method with 

~~~csharp
var choice = await result;

            if (choice.Equals("Account Management", StringComparison.OrdinalIgnoreCase))
            {
                var botDataStore = Conversation.Container.Resolve<IBotDataStore<BotData>>();
                var key = Address.FromActivity(context.Activity);
                var conversationData = await botDataStore.LoadAsync(key, BotStoreType.BotConversationData, CancellationToken.None);
                conversationData.SetProperty<string>("CurrentBotContext", "Accounts");
                await botDataStore.SaveAsync(key, BotStoreType.BotConversationData, conversationData, CancellationToken.None);

                await ForwardToChildBot("fabric:/OneBank.FabricApp/OneBank.AccountsBot", "api/messages", context.Activity);
            }
            else if (choice.Equals("Buy Insurance", StringComparison.OrdinalIgnoreCase))
            {
                var botDataStore = Conversation.Container.Resolve<IBotDataStore<BotData>>();
                var key = Address.FromActivity(context.Activity);
                var conversationData = await botDataStore.LoadAsync(key, BotStoreType.BotConversationData, CancellationToken.None);
                conversationData.SetProperty<string>("CurrentBotContext", "Insurance");
                await botDataStore.SaveAsync(key, BotStoreType.BotConversationData, conversationData, CancellationToken.None);

                await ForwardToChildBot("fabric:/OneBank.FabricApp/OneBank.InsuranceBot", "api/messages", context.Activity);
            }
            else
            {
                context.Done(1);
            }
~~~
Replace the MessageRecievedAsync method with

~~~csharp
var currentBotCtx = context.ConversationData.GetValueOrDefault<string>("CurrentBotContext");

            if (currentBotCtx == "Accounts")
            {
                await ForwardToChildBot("fabric:/OneBank.FabricApp/OneBank.AccountsBot", "api/messages", context.Activity);
            }
            else if (currentBotCtx == "Insurance")
            {
                await ForwardToChildBot("fabric:/OneBank.FabricApp/OneBank.InsuranceBot", "api/messages", context.Activity);
            }
            else
            {
                await context.PostAsync("Hello there! Welcome to OneBank.");
                await context.PostAsync("I am the Master bot");
                
                PromptDialog.Choice(context, ResumeAfterChoiceSelection, new List<string>() { "Account Management", "Buy Insurance" }, "What would you like to do today?");
        } 
~~~

**Task 7** Run the bot again and see the difference
![botStateSuccess]
![botStateActorEvents]
![stickyChildBots]

### Excersice 4 : Put Authentication

**Task 1** Modify StartUp.cs class of master bot and replace the MicrosoftAppiId and MicrosoftAppPassword with actual value

~~~csharp
config.Filters.Add(new BotAuthentication() { MicrosoftAppId = "", MicrosoftAppPassword = "" });
            var microsoftAppCredentials = Conversation.Container.Resolve<MicrosoftAppCredentials>();
            microsoftAppCredentials.MicrosoftAppId = "";
            microsoftAppCredentials.MicrosoftAppPassword = "";
~~~

**Task 2** Modify StartUp.cs class of accounts bot and replace the MicrosoftAppiId and MicrosoftAppPassword with actual value

~~~csharp
config.Filters.Add(new BotAuthentication() { MicrosoftAppId = "", MicrosoftAppPassword = "" });
            var microsoftAppCredentials = Conversation.Container.Resolve<MicrosoftAppCredentials>();
            microsoftAppCredentials.MicrosoftAppId = "";
            microsoftAppCredentials.MicrosoftAppPassword = "";
~~~

**Task 2** Modify StartUp.cs class of insurance bot and replace the MicrosoftAppiId and MicrosoftAppPassword with actual value

~~~csharp
config.Filters.Add(new BotAuthentication() { MicrosoftAppId = "", MicrosoftAppPassword = "" });
            var microsoftAppCredentials = Conversation.Container.Resolve<MicrosoftAppCredentials>();
            microsoftAppCredentials.MicrosoftAppId = "";
            microsoftAppCredentials.MicrosoftAppPassword = "";
~~~

**Task 4** Create call context class
~~~csharp
namespace OneBank.Common
{
    using System.Collections.Concurrent;
    using System.Threading;

    public class RequestCallContext
    {
        /// <summary>
        /// Defines the Context
        /// </summary>
        public static AsyncLocal<string> AuthToken { get; set; } = new AsyncLocal<string>();
    }
}
~~~

**Task 5** First line in the master controller

~~~csharp
RequestCallContext.AuthToken.Value = $"Bearer {this.Request.Headers.Authorization.Parameter}";
~~~

**Task 6** First line Master root dialog (MessageRecieved and Resume)
~~~csharp
public async Task MessageReceivedAsync(IDialogContext context, IAwaitable<IMessageActivity> result)
        {
            var currentBotCtx = context.ConversationData.GetValueOrDefault<string>("CurrentBotContext");

            Dictionary<string, string> headers = new Dictionary<string, string>();
            headers.Add("Authorization", RequestCallContext.AuthToken.Value);

            if (currentBotCtx == "Accounts")
            {
                await ForwardToChildBot("fabric:/OneBank.FabricApp/OneBank.AccountsBot", "api/messages", context.Activity, headers);
            }
            else if (currentBotCtx == "Insurance")
            {
                await ForwardToChildBot("fabric:/OneBank.FabricApp/OneBank.InsuranceBot", "api/messages", context.Activity, headers);
            }
            else
            {
                await context.PostAsync("Hello there! Welcome to OneBank.");
                await context.PostAsync("I am the Master bot");

                PromptDialog.Choice(context, ResumeAfterChoiceSelection, new List<string>() { "Account Management", "Buy Insurance" }, "What would you like to do today?");
            }                      
        }

        private async Task ResumeAfterChoiceSelection(IDialogContext context, IAwaitable<string> result)
        {
            var choice = await result;

            Dictionary<string, string> headers = new Dictionary<string, string>();
            headers.Add("Authorization", RequestCallContext.AuthToken.Value);

            if (choice.Equals("Account Management", StringComparison.OrdinalIgnoreCase))
            {
                var botDataStore = Conversation.Container.Resolve<IBotDataStore<BotData>>();
                var key = Address.FromActivity(context.Activity);
                var conversationData = await botDataStore.LoadAsync(key, BotStoreType.BotConversationData, CancellationToken.None);
                conversationData.SetProperty<string>("CurrentBotContext", "Accounts");
                await botDataStore.SaveAsync(key, BotStoreType.BotConversationData, conversationData, CancellationToken.None);

                await ForwardToChildBot("fabric:/OneBank.FabricApp/OneBank.AccountsBot", "api/messages", context.Activity, headers);
            }
            else if (choice.Equals("Buy Insurance", StringComparison.OrdinalIgnoreCase))
            {
                var botDataStore = Conversation.Container.Resolve<IBotDataStore<BotData>>();
                var key = Address.FromActivity(context.Activity);
                var conversationData = await botDataStore.LoadAsync(key, BotStoreType.BotConversationData, CancellationToken.None);
                conversationData.SetProperty<string>("CurrentBotContext", "Insurance");
                await botDataStore.SaveAsync(key, BotStoreType.BotConversationData, conversationData, CancellationToken.None);

                await ForwardToChildBot("fabric:/OneBank.FabricApp/OneBank.InsuranceBot", "api/messages", context.Activity, headers);
            }
            else
            {
                context.Done(1);
            }
        }
~~~

**Task 3** Observe the changes

![botAuthenticationError]

![botAuthenticationPassed]

### Excercise 5 : Use Application Insights

Install newget package 31.png

Add this in Start up
~~~csharp
TelemetryConfiguration.Active.InstrumentationKey = "8f6e4be0-d0b7-4165-a068-3ccff2c328ad";
            TelemetryConfiguration.Active.TelemetryInitializers.Add(new OperationIdTelemetryInitializer());
            appBuilder.UseApplicationInsights(null, new OperationIdContextMiddlewareConfiguration { OperationIdFactory = IdFactory.FromHeader("X-My-Operation-Id") });
~~~

~~~csharp
// ADD THIS LINE
targetRequest.Headers.Add("X-My-Operation-Id", OperationContext.Get().OperationId);]
// ADD THIS LINE
~~~

[startApp]: https://asfabricstorage.blob.core.windows.net:443/images/19.png
[refreshApp]: https://asfabricstorage.blob.core.windows.net:443/images/18.png
[startBotEmulator]: https://asfabricstorage.blob.core.windows.net:443/images/20.png
[setBotUrl]: https://asfabricstorage.blob.core.windows.net:443/images/21.png
[sayHi]: https://asfabricstorage.blob.core.windows.net:443/images/22.png
[botStateError]: https://asfabricstorage.blob.core.windows.net:443/images/23.png
[botStateSuccess]: https://asfabricstorage.blob.core.windows.net:443/images/24.png
[botStateActorEvents]: https://asfabricstorage.blob.core.windows.net:443/images/25.png
[stickyChildBots]: https://asfabricstorage.blob.core.windows.net:443/images/26.png
[botAuthenticationError]: https://asfabricstorage.blob.core.windows.net:443/images/27.png
[botAuthenticationPassed]: https://asfabricstorage.blob.core.windows.net:443/images/28.png
