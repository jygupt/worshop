
## In this lab, you will create multiple child bots orchestrated by a master bot

### Excercise 1 : Host Bot

**Task 1 :** Add a new OWIN Listener class in common project and paste the below code

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

    /// <summary>
    /// OWIN Communication Listener
    /// </summary>
    /// <seealso cref="Microsoft.ServiceFabric.Services.Communication.Runtime.ICommunicationListener" />
    public class OwinCommunicationListener : ICommunicationListener
    {
        /// <summary>
        /// Defines the endpointName
        /// </summary>
        private readonly string endpointName;

        /// <summary>
        /// Defines the serviceContext
        /// </summary>
        private readonly ServiceContext serviceContext;

        /// <summary>
        /// The startup
        /// </summary>
        private readonly Action<IAppBuilder> startup;

        /// <summary>
        /// Defines the listeningAddress
        /// </summary>
        private string listeningAddress;

        /// <summary>
        /// Defines the publishAddress
        /// </summary>
        private string publishAddress;

        /// <summary>
        /// Defines the webAppHandle
        /// </summary>
        private IDisposable webAppHandle;

        /// <summary>
        /// Initializes a new instance of the <see cref="OwinCommunicationListener" /> class.
        /// </summary>
        /// <param name="startup">The <see cref="Action{IAppBuilder}" /></param>
        /// <param name="serviceContext">The <see cref="ServiceContext" /></param>
        /// <param name="endpointName">The <see cref="string" />Endpoint name</param>
        public OwinCommunicationListener(Action<IAppBuilder> startup, ServiceContext serviceContext, string endpointName)
        {
            this.startup = startup ?? throw new ArgumentNullException(nameof(startup));
            this.serviceContext = serviceContext ?? throw new ArgumentNullException(nameof(serviceContext));
            this.endpointName = endpointName ?? throw new ArgumentNullException(nameof(endpointName));
        }

        /// <summary>
        /// The Abort
        /// </summary>
        public void Abort()
        {
            this.StopHosting();
        }

        /// <summary>
        /// The CloseAsync
        /// </summary>
        /// <param name="cancellationToken">The <see cref="CancellationToken"/></param>
        /// <returns>The <see cref="Task"/></returns>
        public Task CloseAsync(CancellationToken cancellationToken)
        {
            this.StopHosting();
            return Task.FromResult(true);
        }

        /// <summary>
        /// The OpenAsync
        /// </summary>
        /// <param name="cancellationToken">The <see cref="CancellationToken"/></param>
        /// <returns>The endpoints</returns>
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

        /// <summary>
        /// The StopWebServer
        /// </summary>
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
In the master bot, Add the below code inside the CreateServiceInstanceListeners method

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

and then, add an endpoint in the ServiceManifest.xml file under Resource -> Endpoints tag 

~~~xml
<Resources>
    <Endpoints>
      <!-- This endpoint is used by the communication listener to obtain the port on which to 
           listen. Please note that if your service is partitioned, this port is shared with 
           replicas of different partitions that are placed in your code. -->
      <Endpoint Name="ServiceEndpoint" Type="Input" Protocol="http" Port="8770" />
    </Endpoints>
  </Resources>
~~~

In accounts bot, Add the below code inside the CreateServiceInstanceListeners method
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

and then, add an endpoint in the ServiceManifest.xml file under Resource -> Endpoints tag 

~~~xml
<Resources>
    <Endpoints>
      <!-- This endpoint is used by the communication listener to obtain the port on which to 
           listen. Please note that if your service is partitioned, this port is shared with 
           replicas of different partitions that are placed in your code. -->
      <Endpoint Name="ServiceEndpoint" Type="Internal" Protocol="http" Port="8771" />
    </Endpoints>
  </Resources>
~~~
In insurance bot, Add the below code inside the CreateServiceInstanceListeners method

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
  

and then, add an endpoint in the ServiceManifest.xml file under Resource -> Endpoints tag 

~~~xml
<Resources>
    <Endpoints>
      <!-- This endpoint is used by the communication listener to obtain the port on which to 
           listen. Please note that if your service is partitioned, this port is shared with 
           replicas of different partitions that are placed in your code. -->
      <Endpoint Name="ServiceEndpoint" Type="Internal" Protocol="http" Port="8772" />
    </Endpoints>
  </Resources>
~~~

**Task 2 :** Create a basic master root dialog inside dialogs folder.

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
        /// <summary>
        /// starts async
        /// </summary>
        /// <param name="context">sends the context</param>
        /// <returns>returns nothing</returns>
        public Task StartAsync(IDialogContext context)
        {
            context.Wait(this.MessageReceivedAsync);
            return Task.CompletedTask;
        }

        /// <summary>
        /// MessageReceivedAsync method
        /// </summary>
        /// <param name="context">sends context data</param>
        /// <param name="result">sends result data</param>
        /// <returns>returns result based on condition</returns>
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

**Task 3:** Start the application

![startApp]

![refreshApp]

![startBotEmulator]

![setBotUrl]

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
        /// <summary>
        /// Initializes a new instance of the <see cref="HttpCommunicationClient"/> class.
        /// </summary>
        /// <param name="httpClient">The HTTP client.</param>
        public HttpCommunicationClient(HttpClient httpClient)
        {
            this.HttpClient = httpClient;
        }

        /// <summary>
        /// Gets the HTTP client.
        /// </summary>
        /// <value>
        /// The HTTP client.
        /// </value>
        public HttpClient HttpClient { get; private set; }

        /// <summary>
        /// Gets or sets the ResolvedServicePartition
        /// </summary>
        public ResolvedServicePartition ResolvedServicePartition { get; set; }

        /// <summary>
        /// Gets or sets the ListenerName
        /// </summary>
        public string ListenerName { get; set; }

        /// <summary>
        /// Gets or sets the Endpoint
        /// </summary>
        public ResolvedServiceEndpoint Endpoint { get; set; }

        /// <summary>
        /// Gets the HTTP end point.
        /// </summary>
        /// <value>
        /// The HTTP end point.
        /// </value>
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

    /// <summary>
    /// Defines the <see cref="IHttpCommunicationClientFactory" />
    /// </summary>
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
        /// <summary>
        /// The HTTP client
        /// </summary>
        private readonly HttpClient httpClient;

        /// <summary>
        /// Initializes a new instance of the <see cref="HttpCommunicationClientFactory" /> class.
        /// </summary>
        /// <param name="httpClient">The HTTP client.</param>
        public HttpCommunicationClientFactory(HttpClient httpClient)
        {
            this.httpClient = httpClient;
        }

        /// <summary>
        /// The AbortClient
        /// </summary>
        /// <param name="client">The <see cref="HttpCommunicationClient"/></param>
        protected override void AbortClient(HttpCommunicationClient client)
        {
        }

        /// <summary>
        /// The CreateClientAsync
        /// </summary>
        /// <param name="endpoint">The <see cref="string"/></param>
        /// <param name="cancellationToken">The <see cref="CancellationToken"/></param>
        /// <returns>The <see cref="Task{HttpCommunicationClient}"/></returns>
        protected override Task<HttpCommunicationClient> CreateClientAsync(string endpoint, CancellationToken cancellationToken)
        {
            return Task.FromResult(new HttpCommunicationClient(this.httpClient));
        }

        /// <summary>
        /// The ValidateClient
        /// </summary>
        /// <param name="client">The <see cref="HttpCommunicationClient"/></param>
        /// <returns>The <see cref="bool"/></returns>
        protected override bool ValidateClient(HttpCommunicationClient client)
        {
            return true;
        }

        /// <summary>
        /// The ValidateClient
        /// </summary>
        /// <param name="endpoint">The <see cref="string"/></param>
        /// <param name="client">The <see cref="HttpCommunicationClient"/></param>
        /// <returns>The <see cref="bool"/></returns>
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
        /// <summary>
        /// Defines the count
        /// </summary>
        private int count = 1;

        /// <summary>
        /// The StartAsync
        /// </summary>
        /// <param name="context">The <see cref="IDialogContext"/></param>
        /// <returns>The <see cref="Task"/></returns>
        public async Task StartAsync(IDialogContext context)
        {
            await Task.CompletedTask;
            context.Wait(this.MessageReceivedAsync);
        }

        /// <summary>
        /// The MessageReceivedAsync
        /// </summary>
        /// <param name="context">The <see cref="IDialogContext"/></param>
        /// <param name="argument">The <see cref="IAwaitable{IMessageActivity}"/></param>
        /// <returns>The <see cref="Task"/></returns>
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
        /// <summary>
        /// Defines the count
        /// </summary>
        private int count = 1;

        /// <summary>
        /// The StartAsync
        /// </summary>
        /// <param name="context">The <see cref="IDialogContext"/></param>
        /// <returns>The <see cref="Task"/></returns>
        public async Task StartAsync(IDialogContext context)
        {
            await Task.CompletedTask;
            context.Wait(this.MessageReceivedAsync);
        }

        /// <summary>
        /// The MessageReceivedAsync
        /// </summary>
        /// <param name="context">The <see cref="IDialogContext"/></param>
        /// <param name="argument">The <see cref="IAwaitable{IMessageActivity}"/></param>
        /// <returns>The <see cref="Task"/></returns>
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
 /// <summary>
        /// Gets last stored dialog state asynchronously.
        /// </summary>
        /// <param name="key">The key.</param>
        /// <param name="cancellationToken">Cancellation token</param>
        /// <returns>
        /// Bot State
        /// </returns>
        Task<BotStateContext> GetBotStateAsync(string key, CancellationToken cancellationToken);

        /// <summary>
        /// Ensures last stored dialog state asynchronously.
        /// </summary>
        /// <param name="key">The key.</param>
        /// <param name="dialogState">Dialog State</param>
        /// <param name="cancellationToken">Cancellation Token</param>
        /// <returns>
        /// Bot State
        /// </returns>
        Task<BotStateContext> SaveBotStateAsync(string key, BotStateContext dialogState, CancellationToken cancellationToken);

        /// <summary>
        /// Inserts the bot state asynchronous.
        /// </summary>
        /// <param name="key">The key.</param>
        /// <param name="dialogState">State of the dialog.</param>
        /// <param name="cancellationToken">The cancellation token.</param>
        /// <returns>Returns task</returns>
        Task InsertBotStateAsync(string key, BotStateContext dialogState, CancellationToken cancellationToken);

        /// <summary>
        /// Deletes the bot state asynchronous.
        /// </summary>
        /// <param name="key">The key.</param>
        /// <param name="cancellationToken">The cancellation token.</param>
        /// <returns>True if deleted otherwise false</returns>
        Task<bool> DeleteBotStateAsync(string key, CancellationToken cancellationToken);
~~~

**Task 3** Create a new class to store the bot state context

~~~csharp
using System;

namespace OneBank.BotStateActor
{
    /// <summary>
    /// The dialog context.
    /// </summary>
    [Serializable]
    public class BotStateContext
    {
        /// <summary>
        /// Gets or sets the bot identifier.
        /// </summary>
        /// <value>
        /// The bot identifier.
        /// </value>
        public string BotId { get; set; }

        /// <summary>
        /// Gets or sets the user identifier.
        /// </summary>
        /// <value>
        /// The user identifier.
        /// </value>
        public string UserId { get; set; }

        /// <summary>
        /// Gets or sets the channel identifier.
        /// </summary>
        /// <value>
        /// The channel identifier.
        /// </value>
        public string ChannelId { get; set; }

        /// <summary>
        /// Gets or sets the conversation identifier.
        /// </summary>
        /// <value>
        /// The conversation identifier.
        /// </value>
        public string ConversationId { get; set; }

        /// <summary>
        /// Gets or sets the time stamp.
        /// </summary>
        /// <value>
        /// The time stamp.
        /// </value>
        public DateTime TimeStamp { get; set; }

        /// <summary>
        /// Gets or sets the data.
        /// </summary>
        /// <value>
        /// The data.
        /// </value>
        public byte[] Data { get; set; }

        /// <summary>
        /// Gets or sets the e tag.
        /// </summary>
        /// <value>
        /// The e tag.
        /// </value>
        public string ETag { get; set; }
    }
}

~~~

**Task 3** In BotStateActor.cs class add the following methods with thier definition

~~~csharp
 /// <summary>
        /// Gets last stored dialog state asynchronously.
        /// </summary>
        /// <param name="key">The key.</param>
        /// <param name="cancellationToken">Cancellation token</param>
        /// <returns>
        /// Bot State
        /// </returns>
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

        /// <summary>
        /// Ensures last stored dialog state asynchronously.
        /// </summary>
        /// <param name="key">The key.</param>
        /// <param name="dialogState">Dialog State</param>
        /// <param name="cancellationToken">Cancellation Token</param>
        /// <returns>
        /// Asynchronous Task
        /// </returns>
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

        /// <summary>
        /// Inserts the bot state asynchronous.
        /// </summary>
        /// <param name="key">The key.</param>
        /// <param name="dialogState">State of the dialog.</param>
        /// <param name="cancellationToken">The cancellation token.</param>
        /// <returns>Async Task</returns>
        public async Task InsertBotStateAsync(string key, BotStateContext dialogState, CancellationToken cancellationToken)
        {
            ActorEventSource.Current.ActorMessage(this, $"Inserting bot state for actor key - {key}");
            await this.StateManager.AddStateAsync(key, dialogState, cancellationToken);
        }

        /// <summary>
        /// Deletes the bot state asynchronous.
        /// </summary>
        /// <param name="key">The key.</param>
        /// <param name="cancellationToken">The cancellation token.</param>
        /// <returns>True if deleted otherwise false</returns>
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

    /// <summary>
    /// Defines the <see cref="ServiceFabricBotDataStore" />
    /// </summary>
    public class ServiceFabricBotDataStore : IBotDataStore<BotData>
    {
        /// <summary>
        /// Defines the serializationSettings
        /// </summary>
        private static readonly JsonSerializerSettings SerializationSettings = new JsonSerializerSettings()
        {
            Formatting = Formatting.None,
            NullValueHandling = NullValueHandling.Ignore
        };

        /// <summary>
        /// Defines the botName
        /// </summary>
        private readonly string botName;

        /// <summary>
        /// Initializes a new instance of the <see cref="ServiceFabricBotDataStore"/> class.
        /// </summary>
        /// <param name="userProfileProvider">The <see cref="IUserProfileProvider"/></param>
        /// <param name="botName">The <see cref="string"/></param>
        public ServiceFabricBotDataStore(string botName)
        {
            this.botName = botName;
        }

        /// <summary>
        /// The FlushAsync
        /// </summary>
        /// <param name="key">The <see cref="IAddress"/></param>
        /// <param name="cancellationToken">The <see cref="CancellationToken"/></param>
        /// <returns>Async Task</returns>
        public async Task<bool> FlushAsync(IAddress key, CancellationToken cancellationToken)
        {
            return await Task.FromResult(true);
        }

        /// <summary>
        /// The LoadAsync
        /// </summary>
        /// <param name="key">The <see cref="IAddress"/></param>
        /// <param name="botStoreType">The <see cref="BotStoreType"/></param>
        /// <param name="cancellationToken">The <see cref="CancellationToken"/></param>
        /// <returns>The <see cref="Task{BotData}"/></returns>
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

        /// <summary>
        /// The SaveAsync
        /// </summary>
        /// <param name="key">The <see cref="IAddress"/></param>
        /// <param name="botStoreType">The <see cref="BotStoreType"/></param>
        /// <param name="data">The <see cref="BotData"/></param>
        /// <param name="cancellationToken">The <see cref="CancellationToken"/></param>
        /// <returns>The <see cref="Task"/></returns>
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

        /// <summary>
        /// The Serialize
        /// </summary>
        /// <param name="data">The <see cref="object"/></param>
        /// <returns>Serialized data</returns>
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

        /// <summary>
        /// The Deserialize
        /// </summary>
        /// <param name="bytes">Serialized data</param>
        /// <returns>The <see cref="object"/></returns>
        private static object Deserialize(byte[] bytes)
        {
            using (var stream = new MemoryStream(bytes))
            using (var gz = new GZipStream(stream, CompressionMode.Decompress))
            using (var streamReader = new StreamReader(gz))
            {
                return JsonConvert.DeserializeObject(streamReader.ReadToEnd());
            }
        }

        /// <summary>
        /// The GetActorInstance
        /// </summary>
        /// <param name="userId">The <see cref="string"/>User ID</param>
        /// <param name="channelId">The <see cref="string"/>Channel ID</param>
        /// <returns>The <see cref="Task{IBotStateActor}"/></returns>
        private IBotStateActor GetActorInstance(string userId, string channelId)
        {
            return ActorProxy.Create<IBotStateActor>(new ActorId($"{userId}-{channelId}"), new Uri("fabric:/OneBank.FabricApp/BotStateActorService"));
        }

        /// <summary>
        /// The GetStateKey
        /// </summary>
        /// <param name="key">The <see cref="IAddress"/></param>
        /// <param name="botStoreType">The <see cref="BotStoreType"/></param>
        /// <returns>The <see cref="string"/></returns>
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

### Excersice 3 : Put Authentication

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

**Task 3** Observe the changes

![botAuthenticationError]

![botAuthenticationPassed]


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
