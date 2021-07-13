### Hi there 👋

[![Blog](https://img.shields.io/badge/Blog-FFA500?style=for-the-badge&logo=rss&logoColor=white)](https://northern-dev.net)
[![Twitter](https://img.shields.io/badge/Twitter-1DA1F2?style=for-the-badge&logo=twitter&logoColor=white)](https://twitter.com/rafaelldi)
[![Linkedin](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/rivalabdrakhamnov)

---

### 📓 Recent blog posts:

<!--START_SECTION:feed-->
#### [Async request processing](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;async-request-processing&#x2F;) 
*Recently, I’ve talked about state machines and routing slips. In this post, I am going to show how to combine these approaches.

This post came out of a real-life scenario. Imagine that you have a list of items, and you can somehow process each of them. For example, a list of pull requests on GitHub, every one of them you can automatically review. Another key feature is that each operation can take a long time. So, you don’t want to perform them synchronously.
As in the previous posts, let’s start with a simple web application and add MassTransit packages.

$ dotnet new web
$ dotnet add package MassTransit.AspNetCore
$ dotnet add package MassTransit.RabbitMQ


Next, I need some mechanism to process each element separately, one by one. I don’t know how many items will be in a request, so I have to add them dynamically. I think that the routing slip will be appropriate for our case.
I don’t go much into details about implementing routing slip (and saga) in MassTransit because I have separate posts about them. And the library has perfect documentation.
I’ve created a simple activity that logs some information and pauses for a random amount of time. After the delay, it publishes a message about completed item processing.

public record ProcessItemArgument(Guid ItemId);
public record ItemProcessed(Guid Id, Guid TrackingNumber);

public class ProcessItemActivity : IExecuteActivity&lt;ProcessItemArgument&gt;
{
    private readonly RandomService _randomService;
    private readonly ILogger&lt;ProcessItemActivity&gt; _logger;

    public ProcessItemActivity(RandomService randomService, ILogger&lt;ProcessItemActivity&gt; logger)
    {
        _randomService &#x3D; randomService;
        _logger &#x3D; logger;
    }

    public async Task&lt;ExecutionResult&gt; Execute(ExecuteContext&lt;ProcessItemArgument&gt; context)
    {
        _logger.LogInformation(&quot;Processing item with id&#x3D;{ItemId}&quot;, context.Arguments.ItemId);

        var delay &#x3D; _randomService.GetDelay();
        await Task.Delay(TimeSpan.FromSeconds(delay));

        await context.Publish(new ItemProcessed(context.Arguments.ItemId, context.TrackingNumber));
        
        return context.Completed();
    }
}


The next step is to build the routing slip with these activities. I will use a state machine to save the status of the process and check it via http request. When a command to process items arrives, we need to create the routing slip, go to the Processing state and respond that we accepted the request. Also, we save into the state all ids that we need to process.

Initially(
    When(ProcessRequestCommand)
        .Then(context &#x3D;&gt;
        {
            logger.LogInformation(&quot;Start processing request with id&#x3D;{RequestId}&quot;, context.Data.RequestId);
            context.Instance.RequestId &#x3D; context.Data.RequestId;
            context.Instance.ToProcess.AddRange(context.Data.ItemsIds);
        })
        .CreateRoutingSlip()
        .TransitionTo(Processing)
        .Respond(context &#x3D;&gt; new RequestAccepted(context.Instance.RequestId)));



public static EventActivityBinder&lt;RequestProcessingState, ProcessRequestCommand&gt; CreateRoutingSlip(
    this EventActivityBinder&lt;RequestProcessingState, ProcessRequestCommand&gt; binder)
{
    return binder.ThenAsync(async context &#x3D;&gt;
    {
        var trackingNumber &#x3D; Guid.NewGuid();
        context.Instance.TrackingNumber &#x3D; trackingNumber;
        var builder &#x3D; new RoutingSlipBuilder(trackingNumber);
        var consumeContext &#x3D; context.CreateConsumeContext();

        foreach (var item in context.Data.ItemsIds)
        {
            builder.AddActivity(&quot;ProcessItem&quot;, new Uri(&quot;queue:ProcessItem_execute&quot;), new ProcessItemArgument(item));
        }

        var routingSlip &#x3D; builder.Build();
        await consumeContext.Execute(routingSlip);
    });
}


After an item has been processed, we need to update the state.

During(Processing,
    When(ItemProcessed)
        .Then(context &#x3D;&gt;
        {
            logger.LogInformation(&quot;Item with id&#x3D;{ItemId} is processed&quot;, context.Data.Id);
            context.Instance.ToProcess.Remove(context.Data.Id);
            context.Instance.Processed.Add(context.Data.Id);
        }));


Finally, after the routing slip is completed, the state machine goes to the Completed state.

During(Processing,
    When(RequestProcessingCompleted)
        .Then(context &#x3D;&gt;
        {
            logger.LogInformation(&quot;Process completed for request with id&#x3D;{RequestId}&quot;,
                context.Instance.RequestId);
        })
        .TransitionTo(Completed));


Our command side is ready; let’s take a look at query one. How to monitor the process? I’m going to add a new message and handle it within the state machine.

public record RequestStatusQuery(Guid RequestId);
public record RequestStatus(string Status, List&lt;Guid&gt; ToProcess, List&lt;Guid&gt; Processed);

During(Processing,
    When(RequestStatusQueried)
        .Respond(context &#x3D;&gt; new RequestStatus(
            <!--START_SECTION:feed-->
* [Async request processing](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;async-request-processing&#x2F;)
* [Making a plugin isn’t so hard…](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;plugin-for-rider&#x2F;)
* [Choreography](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;choreography&#x2F;)
* [Orchestration](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;orchestration&#x2F;)
* [Coordination in the distributed systems](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;coordination-in-the-distributed-systems&#x2F;)
<!--END_SECTION:feed-->quot;{nameof(Processing)}&quot;,
            context.Instance.ToProcess,
            context.Instance.Processed
        )));

During(Completed,
    When(RequestStatusQueried)
        .Respond(context &#x3D;&gt; new RequestStatus(
            <!--START_SECTION:feed-->
* [Async request processing](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;async-request-processing&#x2F;)
* [Making a plugin isn’t so hard…](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;plugin-for-rider&#x2F;)
* [Choreography](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;choreography&#x2F;)
* [Orchestration](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;orchestration&#x2F;)
* [Coordination in the distributed systems](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;coordination-in-the-distributed-systems&#x2F;)
<!--END_SECTION:feed-->quot;{nameof(Completed)}&quot;,
            context.Instance.ToProcess,
            context.Instance.Processed
        )));


As you can see, we answer with a state and a number of the completed items. So, now let’s create a controller to send messages to the state machine.

public record Request(Guid Id, List&lt;Guid&gt; Items);

[ApiController]
[Route(&quot;requests&quot;)]
public class RequestsController : ControllerBase
{
    private readonly IRequestClient&lt;ProcessRequestCommand&gt; _processRequestClient;
    private readonly IRequestClient&lt;RequestStatusQuery&gt; _getRequestStatusClient;

    public RequestsController(
        IRequestClient&lt;ProcessRequestCommand&gt; processRequestClient,
        IRequestClient&lt;RequestStatusQuery&gt; getRequestStatusClient)
    {
        _processRequestClient &#x3D; processRequestClient;
        _getRequestStatusClient &#x3D; getRequestStatusClient;
    }

    [HttpPost]
    public async Task&lt;IActionResult&gt; Process(Request request)
    {
        var response &#x3D; await _processRequestClient.GetResponse&lt;RequestAccepted&gt;(new ProcessRequestCommand(request.Id, request.Items));
        return Accepted(response.Message.RequestId);
    }

    [HttpGet(&quot;{id:guid}&#x2F;status&quot;)]
    public async Task&lt;IActionResult&gt; Status(Guid id)
    {
        var response &#x3D; await _getRequestStatusClient.GetResponse&lt;RequestStatus&gt;(new RequestStatusQuery(id));
        return Ok(new
        {
            response.Message.Status,
            toProcess &#x3D; string.Join(&quot;,&quot;, response.Message.ToProcess),
            processed &#x3D; string.Join(&quot;,&quot;, response.Message.Processed)
        });
    }
}


This way, we can send a command and monitor the process by polling the status.

POST http:&#x2F;&#x2F;localhost:5000&#x2F;requests
Content-Type: application&#x2F;json

{
  &quot;id&quot;: &quot;B41DEB0B-D088-458C-9241-53375B12117D&quot;,
  &quot;items&quot;: [
    &quot;BF1FAE17-070A-473D-9866-C20E594A4CB1&quot;,
    &quot;8668F382-25B2-4C08-AB02-84B99F9BBCB6&quot;,
    &quot;74A7C363-AF80-4EA6-83BF-CD415ADD7F0C&quot;
  ]
}



GET http:&#x2F;&#x2F;localhost:5000&#x2F;requests&#x2F;B41DEB0B-D088-458C-9241-53375B12117D&#x2F;status

HTTP&#x2F;1.1 200 OK
Content-Type: application&#x2F;json; charset&#x3D;utf-8
Server: Kestrel
Transfer-Encoding: chunked

{
  &quot;status&quot;: &quot;Processing&quot;,
  &quot;toProcess&quot;: &quot;8668f382-25b2-4c08-ab02-84b99f9bbcb6,74a7c363-af80-4ea6-83bf-cd415add7f0c&quot;,
  &quot;processed&quot;: &quot;bf1fae17-070a-473d-9866-c20e594a4cb1&quot;
}


You can find the project in my repository.
Conclusion
This post shows how to combine state machines and routing slips to handle requests with multiple items asynchronously.
References
MassTransit
State machines
Routing slips
Image: Photo by Marvin Castelino on Unsplash*
#### [Making a plugin isn’t so hard…](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;plugin-for-rider&#x2F;) 
*In this post, I describe my experience with plugin creation for JetBrains Rider

It was always interesting for me is it difficult to build your plugin for IDE. Recently, I’ve decided to try. It turns out it’s not that complicated, and I would like to share my experience.
Disclaimer: I’m not a Kotlin or Java developer. Moreover, this is my first experience with these languages. So, the code below might be not fully optimized or doesn’t follow best practices.
In this post, I want to show that plugin creation isn’t rocket science and maybe inspire you to build your own. The more plugins we have, the more productive the whole community will be. This post isn’t a how-to because JetBrains has detailed documentation, and you can find answers there. My goal is to demonstrate that there is a short step from an idea to a worth solution.
Template
To start with, you could create a repository from an IntelliJ Platform Plugin Template. It simplifies the first setup, creates GitHub Actions for building and publishing plugin and more.
IntelliJ Platform Plugin Template
After that, you’ll have a new repository for your plugin. Let’s clone it and add some logic.
Project Tye
Lately, I published a post about Project Tye. It’s an experimental tool from Microsoft to develop distributed applications locally and deploy them to the Kubernetes cluster. So, I want to add support for this tool to the Rider.
Distributed application with Project Tye
I’ll briefly remind you how this tool works. First of all, let’s create a test solution for our experiments.

$ dotnet new sln --name TyeExperiments
$ dotnet new web --name TyeApi
$ dotnet new worker --name TyeWorker
$ dotnet sln TyeExperiments.sln add TyeApi&#x2F;TyeApi.csproj
$ dotnet sln TyeExperiments.sln add TyeWorker&#x2F;TyeWorker.csproj


Now, we have a solution with two projects. Next, install tye .NET tool. Note that you need first install .NET Core 3.1. After that, run the following command:

$ dotnet tool install -g Microsoft.Tye --version &quot;0.6.0-alpha.21070.5&quot;


Finally, go to the TyeExperiments solution folder and execute a command:

$ tye run


Tye starts your projects, and you can find them on the dashboard http:&#x2F;&#x2F;127.0.0.1:8000&#x2F;.

Run Configuration
Excellent, now what I want to accomplish is to run this command directly from my IDE. IntelliJ Platform SDK gives us the ability to add custom Run Configuration. So, I think that it’s the best place to extend basic functionality with our plugin. There is a tutorial in the IntelliJ Platform documentation, and I show my implementation bellow.
Start a new extension from the plugin.xml file and register configurationType.

&lt;extensions defaultExtensionNs&#x3D;&quot;com.intellij&quot;&gt;
  &lt;configurationType implementation&#x3D;&quot;com.github.rafaelldi.tyeplugin.run.TyeConfigurationType&quot;&#x2F;&gt;
&lt;&#x2F;extensions&gt; 


Implement this ConfigurationType with a new class.

class TyeConfigurationType : ConfigurationType {
    override fun getDisplayName() &#x3D; &quot;Tye&quot;

    override fun getConfigurationTypeDescription() &#x3D; &quot;Tye run command&quot;

    override fun getIcon() &#x3D; AllIcons.General.Information

    override fun getId() &#x3D; &quot;TYE_RUN_CONFIGURATION&quot;

    override fun getConfigurationFactories(): Array&lt;ConfigurationFactory&gt; &#x3D; arrayOf(TyeConfigurationFactory(this))
}


Introduce a new ConfigurationFactory to be able to produce a run configuration.

class TyeConfigurationFactory(type: TyeConfigurationType) : ConfigurationFactory(type) {
    companion object {
        private const val FACTORY_NAME &#x3D; &quot;Tye configuration factory&quot;
    }

    override fun createTemplateConfiguration(project: Project): RunConfiguration {
        return TyeRunConfiguration(project, this, &quot;Tye&quot;)
    }

    override fun getName() &#x3D; FACTORY_NAME

    override fun getId() &#x3D; FACTORY_NAME
}


Create a RunConfiguration class. With it, you can execute different actions, for example, call tye commands.

class TyeRunConfiguration(project: Project, factory: TyeConfigurationFactory, name: String) :
    RunConfigurationBase&lt;TyeCommandLineState&gt;(project, factory, name) {

    override fun getConfigurationEditor(): SettingsEditor&lt;out RunConfiguration&gt; &#x3D; TyeSettingsEditor()

    override fun checkConfiguration() {
    }

    override fun getState(executor: Executor, environment: ExecutionEnvironment): RunProfileState {
        return TyeCommandLineState(environment, this, project)
    }
}


Additionally, I create a TyeCommandLineState class to locate operations with a command line. We’ll come back to this later.

open class TyeCommandLineState(
    environment: ExecutionEnvironment,
    private val runConfig: TyeRunConfiguration,
    private val project: Project
) : CommandLineState(environment) {

    override fun startProcess(): ProcessHandler {
        val commandLine &#x3D; GeneralCommandLine()
        val handler &#x3D; OSProcessHandler(commandLine)
        return handler
    }
}


After all, add SettingsEditor class to handle UI form.

class TyeSettingsEditor : SettingsEditor&lt;TyeRunConfiguration&gt;() {
    private lateinit var panel: JPanel

    override fun createEditor(): JComponent {
        createUIComponents()
        return panel
    }

    override fun resetEditorFrom(runConfig: TyeRunConfiguration) {
    }

    override fun applyEditorTo(runConfig: TyeRunConfiguration) {
    }

    private fun createUIComponents() {
        panel &#x3D; JPanel().apply {
            layout &#x3D; VerticalFlowLayout(VerticalFlowLayout.TOP)
        }
    }
}


Pretty straightforward implementation. I won’t go into detail about each file. As I said before, you can find their description in the documentation. Eventually, if you run the plugin, you would see the new Run Configuration type. For now, it doesn’t do anything; let’s go to the next section and add some behaviour.

Tye run command
To call tye run command, we’ll modify TyeCommandLineState class.

override fun startProcess(): ProcessHandler {
    val arguments &#x3D; mutableListOf&lt;String&gt;()
    arguments.add(&quot;run&quot;)

    val commandLine &#x3D; GeneralCommandLine()
        .withParentEnvironmentType(GeneralCommandLine.ParentEnvironmentType.CONSOLE)
        .withWorkDirectory(project.basePath)
        .withExePath(&quot;tye&quot;)
        .withParameters(arguments)

    val handler &#x3D; OSProcessHandler(commandLine)
    handler.startNotify()
    ProcessTerminatedListener.attach(handler, environment.project)
    return handler
}


Start the plugin again, select TyeExperiments solution folder and run the new tye configuration type. You’ll see the same logs as before when we ran tye independently.

Conclusion
This post shows how to create a primitive plugin to envelope some command-line tool calls. Of course, the capabilities of the platform are much more significant. You can find a lot of complicated plugins in the marketplace. I’ve tried to show that if you want to add some simple functionality, it’s not hard to do.
The documentation is an excellent starting point for your exploration. If it isn’t enough, you can find how to implement some functionality in the other plugins.
I hope I was able to motivate you to try creating your own plugin. My source code you can find on GitHub.
Tye Plugin
References
Project Tye on GitHub
Introducing Project Tye
IntelliJ Platform Plugin Template
IntelliJ Platform SDK
IntelliJ Platform Explorer
Image: Photo by Johanneke Kroesbergen-Kamps on Unsplash*
#### [Choreography](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;choreography&#x2F;) 
*This post shows you an example of the choreography pattern to coordinate services in a system.

In the previous post, we created a food delivery application and applied the orchestration pattern. In this one, I’m going to modify that solution to follow the choreography pattern.
Orchestration
Let me remind the process:
A user places online order from the website;
The manager receives a notification about the new order and accepts (or denies) it;
The kitchen gets the order details and starts cooking;
The courier delivers food to the user’s address.

Routing slip
Sometimes you don’t want to couple to one central service (orchestrator) as we did in the previous post. In that case, it’s possible to connect your services via pub&#x2F;sub mechanism. It’s not so complicated and has some pitfalls, especially if you want to add compensating transactions. So, today I’ll additionally apply a routing slip pattern. I described it in the post about coordination. In short, you put all steps (and compensations if needed) in a routing slip and attach it to the message. Hence, every service knows where to send this message next.
Coordination in the distributed systems
Modify application
Notice that after the Place Order step, our pipeline stops and waits for the manager reaction. So, we won’t include this step in the routing slip.
Let’s modify the PlaceOrder method in the OrdersController. We need to send OrderPlaced event to reuse the corresponding consumer, which will send a notification to the manager. Also, we save order details and address to cash because we’ll need them in the AcceptOrder method.

[ApiController]
[Route(&quot;[controller]&quot;)]
public class OrdersController : ControllerBase
{
    private readonly IBus _bus;
    private static readonly Dictionary&lt;Guid, (string OrderDetails, string Address)&gt; Cash &#x3D; new();

    public OrdersController(IBus bus)
    {
        _bus &#x3D; bus;
    }

    [HttpPost]
    public async Task&lt;IActionResult&gt; PlaceOrder(OrderDto dto)
    {
        var orderId &#x3D; Guid.NewGuid();

        Cash.Add(orderId, (dto.OrderDetails, dto.Address));

        await _bus.Publish(new OrderPlaced
        {
            OrderId &#x3D; orderId,
            OrderDetails &#x3D; dto.OrderDetails
        });

        return Ok();
    }

    &#x2F;&#x2F; ...
}


That’s all with the first step.
Next, to build a routing slip, we need to create an activity for each step in the pipeline. So, we end with CookDishActivity and DeliverOrderActivity. They will send messages to the consumers (again, to reuse functionality from the previous part) and wait for the responses from them. After the activity is done, we call context.Completed() method to advance the routing slip to the next one. Each activity is located in the related service.

public class CookDishActivity : IExecuteActivity&lt;CookDishArgument&gt;
{
    private readonly IBus _bus;

    public CookDishActivity(IBus bus)
    {
        _bus &#x3D; bus;
    }

    public async Task&lt;ExecutionResult&gt; Execute(ExecuteContext&lt;CookDishArgument&gt; context)
    {
        var client &#x3D; context.CreateRequestClient&lt;CookDish&gt;(_bus);
        await client.GetResponse&lt;DishCooked&gt;(new CookDish
        {
            OrderId &#x3D; context.Arguments.OrderId,
            OrderDetails &#x3D; context.Arguments.OrderDetails
        });
        return context.Completed();
    }
}



public class DeliverOrderActivity : IExecuteActivity&lt;DeliverOrderArgument&gt;
{
    private readonly IBus _bus;
    
    public DeliverOrderActivity(IBus bus)
    {
        _bus &#x3D; bus;
    }

    public async Task&lt;ExecutionResult&gt; Execute(ExecuteContext&lt;DeliverOrderArgument&gt; context)
    {
        var client &#x3D; context.CreateRequestClient&lt;DeliverOrder&gt;(_bus);
        await client.GetResponse&lt;OrderDelivered&gt;(new DeliverOrder
        {
            OrderId &#x3D; context.Arguments.OrderId,
            Address &#x3D; context.Arguments.Address
        });
        return context.Completed();
    }
}


After that, register them in the Startup.

services.AddMassTransit(x &#x3D;&gt;
    {
        &#x2F;&#x2F; ...

        x.AddExecuteActivity&lt;CookDishActivity, CookDishArgument&gt;();
        x.AddExecuteActivity&lt;DeliverOrderActivity, DeliverOrderArgument&gt;();

        &#x2F;&#x2F;...
    })


Finally, build the routing slip. As you see, we don’t include any message types or consumer details. All we need are the addresses of activities. Therefore, we reduce coupling between components in the system. As I said early, this routing slip will be attached to the message, and each service will know where to send it next.

[HttpPost(&quot;{id}&#x2F;accept&quot;)]
public async Task&lt;IActionResult&gt; AcceptOrder(Guid id)
{
    if (!Cash.TryGetValue(id, out var order))
    {
        throw new ArgumentException(&quot;Can&#39;t find order details&quot;);
    }

    var builder &#x3D; new RoutingSlipBuilder(Guid.NewGuid());

    builder.AddActivity(&quot;CookDish&quot;, new Uri(&quot;queue:CookDish_execute&quot;), new CookDishArgument
    {
        OrderId &#x3D; id,
        OrderDetails &#x3D; order.OrderDetails
    });

    builder.AddActivity(&quot;DeliverOrder&quot;, new Uri(&quot;queue:DeliverOrder_execute&quot;), new DeliverOrderArgument
    {
        OrderId &#x3D; id,
        Address &#x3D; order.Address
    });

    var routingSlip &#x3D; builder.Build();

    await _bus.Execute(routingSlip);
    return Ok();
}


Our choreography-based system is ready. We’ve removed the central component, which instructs others what to do and oversees the process. Now, we initially create instructions and send them with a message.
If you test the application, you’ll see similar logs.

info: Microsoft.AspNetCore.Hosting.Diagnostics[1]
      Request starting HTTP&#x2F;1.1 POST http:&#x2F;&#x2F;localhost:5000&#x2F;choreography&#x2F;orders application&#x2F;json 62
info: Microsoft.AspNetCore.Hosting.Diagnostics[2]
      Request finished HTTP&#x2F;1.1 POST http:&#x2F;&#x2F;localhost:5000&#x2F;choreography&#x2F;orders application&#x2F;json 62 - 200 0 - 393.6161ms
info: CommunicationFoodDelivery.Consumers.OrderPlacedConsumer[0]
      OrderPlaced event received
info: CommunicationFoodDelivery.Consumers.OrderPlacedConsumer[0]
      Order with id &#x3D; 98641c8b-4ec5-4859-a0a1-8732b4ef600b and details &#x3D; Pizza was placed
info: CommunicationFoodDelivery.Consumers.OrderPlacedConsumer[0]
      Sending notification to the manager...

info: Microsoft.AspNetCore.Hosting.Diagnostics[1]
      Request starting HTTP&#x2F;1.1 POST http:&#x2F;&#x2F;localhost:5000&#x2F;orders&#x2F;98641c8b-4ec5-4859-a0a1-8732b4ef600b&#x2F;accept application&#x2F;json 3
info: Microsoft.AspNetCore.Hosting.Diagnostics[2]
      Request finished HTTP&#x2F;1.1 POST http:&#x2F;&#x2F;localhost:5000&#x2F;orders&#x2F;98641c8b-4ec5-4859-a0a1-8732b4ef600b&#x2F;accept application&#x2F;json 3 - 200 0 - 70.4028ms
info: CommunicationFoodDelivery.Consumers.CookDishConsumer[0]
      CookDish command received
info: CommunicationFoodDelivery.Consumers.CookDishConsumer[0]
      Dish for order with id &#x3D; 98641c8b-4ec5-4859-a0a1-8732b4ef600b was cooked
info: CommunicationFoodDelivery.Consumers.DeliverOrderConsumer[0]
      DeliverOrder command received
info: CommunicationFoodDelivery.Consumers.DeliverOrderConsumer[0]
      Order with id &#x3D; 98641c8b-4ec5-4859-a0a1-8732b4ef600b was delivered


All code is available on GitHub. You can compare two approaches side-by-side.
Link to GitHub Project
Conclusion
Today, we’ve transformed the application with a choreography approach to reduce coupling and increase the autonomy of components in the system. Each pattern has advantages and disadvantages. It depends on a lot of factors which one you want to apply. I hope these posts give you some ideas about how to coordinate services in a distributed system.
References
Coordination in the distributed systems
https:&#x2F;&#x2F;masstransit-project.com&#x2F;
https:&#x2F;&#x2F;masstransit-project.com&#x2F;advanced&#x2F;courier&#x2F;
Image: Photo by Gaelle Marcel on Unsplash*
#### [Orchestration](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;orchestration&#x2F;) 
*In this post, I want to show you an example of how to apply an orchestration pattern in your system.

The previous post was about two main patterns for coordinating services in a distributed system. So, today we’ll take a look at the straightforward application for food delivery. Within this system, we’ll try to connect different parts with orchestration pattern.
Coordination in the distributed systems
Spoiler: this post has a lot of code. If you want to look at the project yourself, I left a link to GitHub at the end of the post.
Let’s pretend we’re developing an application for the restaurant. As you may notice, food delivery is popular nowadays, so we want to implement this functionality in the app.

The whole process will be consists of four steps:
A user places online order from the website;
The manager receives a notification about the new order and accepts (or denies) it;
The kitchen gets the order details and cooks it;
The courier delivers food to the user’s address.
Application
Let’s start with an empty ASP.NET template.

$ dotnet new web


We’ll use an excellent library MassTransit, so we need to install this package.

$ dotnet add package MassTransit.AspNetCore


MassTransit
Controller and consumers
First of all, I’m going to add a new controller with two methods PlaceOrder and AcceptOrder. The first one will be used by customers and the second one by managers. Also, we need to register the controller in the Startup class.

[ApiController]
[Route(&quot;[controller]&quot;)]
public class OrdersController : ControllerBase
{
    [HttpPost]
    public async Task&lt;IActionResult&gt; PlaceOrder(OrderDto dto)
    {
        return Ok();
    }

    [HttpPost(&quot;{id}&#x2F;accept&quot;)]
    public async Task&lt;IActionResult&gt; AcceptOrder(Guid id)
    {
        return Ok();
    }
}



public record OrderDto(string OrderDetails, string Address);



public class Startup
 {
     public void ConfigureServices(IServiceCollection services)
     {
         services.AddControllers();
     }

     public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
     {
         app.UseRouting();

         app.UseEndpoints(endpoints &#x3D;&gt;
         {
             endpoints.MapControllers();
         });
     }
 }


Now, create commands and events. We’ll use them to trigger steps from our pipeline. You can easily relate them to the diagram above. Commands are sent to perform some action, events to report that the action has happened.

public static class Commands
{
    public record PlaceOrder
    {
        public Guid OrderId { get; init; }
        public string OrderDetails { get; init; }
        public string Address { get; init; }
    }

    public record AcceptOrder
    {
        public Guid OrderId { get; init; }
    }

    public record CookDish
    {
        public Guid OrderId { get; init; }
        public string OrderDetails { get; init; }
    }

    public record DeliverOrder
    {
        public Guid OrderId { get; init; }
        public string Address { get; init; }
    }
}



public static class Events
{
    public record OrderPlaced
    {
        public Guid OrderId { get; init; }
        public string OrderDetails { get; init; }
    }

    public record DishCooked
    {
        public Guid OrderId { get; init; }
    }

    public record OrderDelivered
    {
        public Guid OrderId { get; init; }
    }
}


After that, let’s add consumers. Consumers are similar to controllers but used for messaging. Our consumers are straightforward; they just log some information. Of course, in the real application, logic is more complicated. Moreover, they might be located in different microservices, but I’ll create them in our API project for simplicity. I recently showed the approach with multiple containers connected via RabbitMQ.
Distributed application with Project Tye

public class OrderPlacedConsumer : IConsumer&lt;OrderPlaced&gt;
{
    private readonly ILogger&lt;OrderPlacedConsumer&gt; _logger;

    public OrderPlacedConsumer(ILogger&lt;OrderPlacedConsumer&gt; logger)
    {
        _logger &#x3D; logger;
    }

    public async Task Consume(ConsumeContext&lt;OrderPlaced&gt; context)
    {
        _logger.LogInformation(<!--START_SECTION:feed-->
* [Async request processing](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;async-request-processing&#x2F;)
* [Making a plugin isn’t so hard…](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;plugin-for-rider&#x2F;)
* [Choreography](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;choreography&#x2F;)
* [Orchestration](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;orchestration&#x2F;)
* [Coordination in the distributed systems](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;coordination-in-the-distributed-systems&#x2F;)
<!--END_SECTION:feed-->quot;{nameof(OrderPlaced)} event received&quot;);

        await Task.Delay(500);

        _logger.LogInformation(&quot;Order with id &#x3D; {id} and details &#x3D; {details} was placed&quot;,
            context.Message.OrderId.ToString(), context.Message.OrderDetails);
        _logger.LogInformation(&quot;Sending notification to the manager...&quot;);
    }
}



public class CookDishConsumer : IConsumer&lt;CookDish&gt;
{
    private readonly ILogger&lt;CookDishConsumer&gt; _logger;

    public CookDishConsumer(ILogger&lt;CookDishConsumer&gt; logger)
    {
        _logger &#x3D; logger;
    }

    public async Task Consume(ConsumeContext&lt;CookDish&gt; context)
    {
        _logger.LogInformation(<!--START_SECTION:feed-->
* [Async request processing](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;async-request-processing&#x2F;)
* [Making a plugin isn’t so hard…](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;plugin-for-rider&#x2F;)
* [Choreography](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;choreography&#x2F;)
* [Orchestration](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;orchestration&#x2F;)
* [Coordination in the distributed systems](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;coordination-in-the-distributed-systems&#x2F;)
<!--END_SECTION:feed-->quot;{nameof(CookDish)} command received&quot;);

        await Task.Delay(500);

        var orderId &#x3D; context.Message.OrderId;
        _logger.LogInformation(&quot;Dish for order with id &#x3D; {id} was cooked&quot;, orderId.ToString());
        await context.RespondAsync(new DishCooked {OrderId &#x3D; orderId});
    }
}



public class DeliverOrderConsumer : IConsumer&lt;DeliverOrder&gt;
{
    private readonly ILogger&lt;DeliverOrderConsumer&gt; _logger;

    public DeliverOrderConsumer(ILogger&lt;DeliverOrderConsumer&gt; logger)
    {
        _logger &#x3D; logger;
    }

    public async Task Consume(ConsumeContext&lt;DeliverOrder&gt; context)
    {
        _logger.LogInformation(<!--START_SECTION:feed-->
* [Async request processing](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;async-request-processing&#x2F;)
* [Making a plugin isn’t so hard…](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;plugin-for-rider&#x2F;)
* [Choreography](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;choreography&#x2F;)
* [Orchestration](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;orchestration&#x2F;)
* [Coordination in the distributed systems](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;coordination-in-the-distributed-systems&#x2F;)
<!--END_SECTION:feed-->quot;{nameof(DeliverOrder)} command received&quot;);

        await Task.Delay(500);

        var orderId &#x3D; context.Message.OrderId;
        _logger.LogInformation(&quot;Order with id &#x3D; {id} was delivered&quot;, orderId.ToString());
        await context.RespondAsync(new OrderDelivered {OrderId &#x3D; orderId});
    }
}


Notice that consumers perform their actions in response to the messages they receive.
Possible implementations of consumers in different services are shown in the diagram below.

Next, register the consumers and the library itself. For testing purposes, I’m using in-memory message bus.

services.AddMassTransit(x &#x3D;&gt;
    {
        x.AddConsumer&lt;CookDishConsumer&gt;();
        x.AddConsumer&lt;OrderPlacedConsumer&gt;();
        x.AddConsumer&lt;DeliverOrderConsumer&gt;();

        x.UsingInMemory((context, cfg) &#x3D;&gt; { cfg.ConfigureEndpoints(context); });
    })
    .AddMassTransitHostedService();


Now, all groundwork is done, let’s connect different parts.
State machine
The heart of our system is an orchestrator, which will be inside the API project. It will follow the business process, keep the state and send messages to other components. It’s natural to implement our pipeline as a state machine. MassTransit gives us a wide range of possibilities to customize it; we’ll use only a few of them. Here you can find the documentation about state machines in MassTransit.
Automatonymous
Firstly, let’s describe a state that we’ll keep for each instance of the state machine.

public class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public int CurrentState { get; set; }

    public Guid OrderId { get; set; }
    public string OrderDetails { get; set; }
    public string Address { get; set; }
    public DateTime Placed { get; set; }
    public DateTime Accepted { get; set; }
    public DateTime Cooked { get; set; }
    public DateTime Delivered { get; set; }
}


CorrelationId and CurrentState are required fields; OrderId, OrderDetails and Address come from user request; Placed, Accepted, Cooked and Delivered fields we’ll use to save the time of the events.
Next, create a state machine, add possible states and events.

public class OrderStateMachine : MassTransitStateMachine&lt;OrderState&gt;
{
    public OrderStateMachine()
    {
        InstanceState(x &#x3D;&gt; x.CurrentState, Placed, Accepted, Cooked);
    }

    public Event&lt;PlaceOrder&gt; PlaceOrder { get; private set; }
    public Event&lt;AcceptOrder&gt; AcceptOrder { get; private set; }
    public Event&lt;DishCooked&gt; DishCooked { get; private set; }
    public Event&lt;OrderDelivered&gt; OrderDelivered { get; private set; }

    public State Placed { get; private set; }
    public State Accepted { get; private set; }
    public State Cooked { get; private set; }
}


After that, we should specify the fields to correlate our messages. It tells a state machine which instance should be chosen to apply incoming event.

public OrderStateMachine()
{
    &#x2F;&#x2F; ...

    Event(() &#x3D;&gt; PlaceOrder, x &#x3D;&gt; x.CorrelateById(m &#x3D;&gt; m.Message.OrderId));
    Event(() &#x3D;&gt; AcceptOrder, x &#x3D;&gt; x.CorrelateById(m &#x3D;&gt; m.Message.OrderId));
    Event(() &#x3D;&gt; DishCooked, x &#x3D;&gt; x.CorrelateById(m &#x3D;&gt; m.Message.OrderId));
    Event(() &#x3D;&gt; OrderDelivered, x &#x3D;&gt; x.CorrelateById(m &#x3D;&gt; m.Message.OrderId));
}


Then, add reactions to the incoming messages. You can see that I describe how to change states, publish other messages and save something to the instance of the state machine.

public OrderStateMachine()
{
    &#x2F;&#x2F; ...

    Initially(
        When(PlaceOrder)
            .SetOrderDetails()
            .TransitionTo(Placed)
            .PublishOrderPlaced());

    During(Placed,
        When(AcceptOrder)
            .SetAcceptedTime()
            .TransitionTo(Accepted)
            .PublishCookDish());

    During(Accepted,
        When(DishCooked)
            .SetCookedTime()
            .TransitionTo(Cooked)
            .PublishDeliverOrder());

    During(Cooked,
        When(OrderDelivered)
            .SetDeliveredTime()
            .Finalize());
}

&#x2F;&#x2F; ...

public static EventActivityBinder&lt;OrderState, PlaceOrder&gt; SetOrderDetails(
    this EventActivityBinder&lt;OrderState, PlaceOrder&gt; binder)
{
    return binder.Then(x &#x3D;&gt;
    {
        x.Instance.OrderId &#x3D; x.Data.OrderId;
        x.Instance.OrderDetails &#x3D; x.Data.OrderDetails;
        x.Instance.Address &#x3D; x.Data.Address;
        x.Instance.Placed &#x3D; DateTime.UtcNow;
    });
}

public static EventActivityBinder&lt;OrderState, PlaceOrder&gt; PublishOrderPlaced(
    this EventActivityBinder&lt;OrderState, PlaceOrder&gt; binder)
{
    return binder.PublishAsync(context &#x3D;&gt; context.Init&lt;OrderPlaced&gt;(new OrderPlaced
    {
        OrderId &#x3D; context.Data.OrderId,
        OrderDetails &#x3D; context.Data.OrderDetails
    }));
}

&#x2F;&#x2F; ...


Eventually, modify controllers to send corresponding commands and register state machine in the Startup class.

&#x2F;&#x2F; ...
private readonly IPublishEndpoint _publishEndpoint;

public OrdersController(IPublishEndpoint publishEndpoint)
{
    _publishEndpoint &#x3D; publishEndpoint;
}

[HttpPost]
public async Task&lt;IActionResult&gt; PlaceOrder(OrderDto dto)
{
    await _publishEndpoint.Publish(new PlaceOrder
    {
        OrderId &#x3D; Guid.NewGuid(),
        OrderDetails &#x3D; dto.OrderDetails,
        Address &#x3D; dto.Address
    });
    return Ok();
}

[HttpPost(&quot;{id}&#x2F;accept&quot;)]
public async Task&lt;IActionResult&gt; AcceptOrder(Guid id)
{
    await _publishEndpoint.Publish(new AcceptOrder
    {
        OrderId &#x3D; id
    });
    return Ok();
}



x.AddSagaStateMachine&lt;OrderStateMachine, OrderState&gt;()
    .InMemoryRepository();


The whole process is as follows. The user sends a request to create an order. The controller transforms this request into a PlaceOrder command and sends it to the message bus. The state machine receives the command, sets itself to Placed status and sends the OrderPlaced event. The corresponding OrderPlacedConsumer responds to this event and sends a notification to the manager about the new order. At this point, the state machine pauses and waits for action from the user. After the manager approves the order with a request, the controller sends an AcceptOrder command. The state machine responds, sends a CookDish command, waits for a message from the DishCookedConsumer and sends a DeliverOrder command to deliver the order. After the order is delivered, the message OrderDelivered comes, the state machine passes to the final state.
After all this work, you can finally test our project. Send POST request to the controller and follow all the steps of our process. In the logs you will see the next entries.

info: Microsoft.AspNetCore.Hosting.Diagnostics[1]
      Request starting HTTP&#x2F;1.1 POST http:&#x2F;&#x2F;localhost:5000&#x2F;orders application&#x2F;json 63
info: Microsoft.AspNetCore.Hosting.Diagnostics[2]
      Request finished HTTP&#x2F;1.1 POST http:&#x2F;&#x2F;localhost:5000&#x2F;orders application&#x2F;json 63 - 200 0 - 563.1161ms
info: CommunicationFoodDelivery.Consumers.OrderPlacedConsumer[0]
      OrderPlaced event received
info: CommunicationFoodDelivery.Consumers.OrderPlacedConsumer[0]
      Order with id &#x3D; 423c53af-0e3f-4659-820b-75b1c6bac2ba and details &#x3D; Burger was placed
info: CommunicationFoodDelivery.Consumers.OrderPlacedConsumer[0]
      Sending notification to the manager...

info: Microsoft.AspNetCore.Hosting.Diagnostics[1]
      Request starting HTTP&#x2F;1.1 POST http:&#x2F;&#x2F;localhost:5000&#x2F;orders&#x2F;423c53af-0e3f-4659-820b-75b1c6bac2ba&#x2F;accept application&#x2F;json 3
iinfo: Microsoft.AspNetCore.Hosting.Diagnostics[2]
      Request finished HTTP&#x2F;1.1 POST http:&#x2F;&#x2F;localhost:5000&#x2F;orders&#x2F;423c53af-0e3f-4659-820b-75b1c6bac2ba&#x2F;accept application&#x2F;json 3 - 200 0 - 43.5609ms
info: CommunicationFoodDelivery.Consumers.CookDishConsumer[0]
      CookDish command received
info: CommunicationFoodDelivery.Consumers.CookDishConsumer[0]
      Dish for order with id &#x3D; 423c53af-0e3f-4659-820b-75b1c6bac2ba was cooked
info: CommunicationFoodDelivery.Consumers.DeliverOrderConsumer[0]
      DeliverOrder command received
info: CommunicationFoodDelivery.Consumers.DeliverOrderConsumer[0]
      Order with id &#x3D; 423c53af-0e3f-4659-820b-75b1c6bac2ba was delivered


It is worth noting that you don’t wait for the process to complete inside the controller. The web request was processed faster than the end of the state machine. Mostly, long-running processes are handled asynchronously. I’m going to show you how to proper handle it in future posts.
I didn’t show all the code because the post is long enough as it is. You can find the project on GitHub.
Link to GitHub Project
Conclusion
In this post, I’ve shown a possible implementation of the orchestration pattern. We built the application for food delivery which have the orchestrator and three consumers. I didn’t separate them into different services because the example is already long. But with MassTransit, it’s straightforward to do that. If there is too much code for you, you can easily download the project from GitHub and explore it by yourself. If you have any question, I will be happy to answer them in the comments below.
In the next post, I will show how to modify this solution towards the choreography pattern. I hope it will contain a much smaller amount of code 😉.
References
Coordination in the distributed systems
https:&#x2F;&#x2F;masstransit-project.com&#x2F;
https:&#x2F;&#x2F;masstransit-project.com&#x2F;usage&#x2F;sagas&#x2F;automatonymous.html
Image: Photo by Andrey Konstantinov on Unsplash*
#### [Coordination in the distributed systems](https:&#x2F;&#x2F;rafaelldi.github.io&#x2F;&#x2F;posts&#x2F;coordination-in-the-distributed-systems&#x2F;) 
*Recently, I had a talk about distributed systems. This type of architecture is popular nowadays. In this post, I want to discuss one of such systems’ main problems: how to coordinate different parts.

Some user requests might be handled by a group of services in your system. For example, you take an order, process payment and ship that order. And these steps might be located in different parts of your application because each service is responsible for a small piece of the domain. Of course, these services must process the order in a certain sequence. Let’s take a look at how to organize this interservice communication.
There are two main patterns: choreography and orchestration.
Choreography

In this pattern, each service subscribes to events from the others. When an event comes, the service executes its action and produces a new event. Eventually, you have a chain of the services which handles the request sequentially.
Choreography reduces coupling in your system because the services don’t know anything about each other. All they know is the events to come to them.
Some services may not handle events but enrich them. These services add some useful information from their databases to the message and resend it. Thus, the next service in the chain can use more data. This approach is similar to the pattern Pipes an Filters.
Pipes and Filters pattern
The downside of the Choreography is that it’s possible to describe only elementary processes with the subscribing to the events. If you have many steps with conditions, cycles and parallel steps, Choreography isn’t for you.
Another weakness is a large number of services. It’s troublesome to maintain systems with many events and subscriptions. You don’t exactly know which services will be triggered by certain events, and a vast amount of traffic is passed through the message bus.
Also, it’s challenging to recover from failure. If a service falls and doesn’t send any event, your process will break down, and other services won’t know about it.
Orchestration

In this pattern, you have one central service called a conductor. It knows the workflow and sends commands to the other services. In its turn, the service executes the command and responds to the conductor.
With orchestration, you can create complex and tricky workflows by describing them in your conductor. Also, the conductor deals with all failures in the system. It waits for the services responds, retries the requests, notifies system administrators.
The main drawback is that the conductor is a single point of failure. If it falls, your system won’t handle users requests. Furthermore, there is a high coupling between the conductor and services. In the distributed systems, you should avoid any coupling between components. Eventually, most of the domain logic ends up in this central service, and it becomes responsible for everything.
Saga
Sometimes you need to accomplish all steps in your pipeline or undo everything if one of the steps failed. If you have one database, it’s simple to achieve by using a database transaction. But in the distributed systems each service has its own database, and this straightforward solution doesn’t work.
Another way is a distributed transaction or two-phase commit (2PC). However, this approach doesn’t work either because many modern databases or queues don’t support this protocol, and this pattern reduces system availability. In distributed systems, you have to choose between availability and consistency (see CAP theorem).
Fortunately, there is a pattern called Saga. It was initially formulated by Hector Garcaa-Molrna and Kenneth Salem in the article in 1987.
In short, the saga is a sequence of steps, and all steps have compensating transactions. If something breaks down, this transaction undoes the effect of the corresponding action. So, in the end, all steps will be accomplished or compensated.
Also, each action affects only one service. Therefore, within a step, we can use a local transaction. In summary, we might say, that saga is a sequence of local transactions with appropriate compensating transactions.

In some situations, it’s impossible to revert some actions. For example, sending an email, you can’t return it back. You should send a new email with excuses. So, compensating transactions might be tricky.
One more thing to consider is that sagas are eventually consistent. Changes in different services will be available at different times. So, it’s a good idea to mark some objects as pending during the saga. It will prevent other processes from reading or modifying these objects.
This pattern is often built on the choreography and orchestration patterns. After all, you need to establish an interaction between services. In the orchestration model, the orchestrator executes the saga and compensates actions if required. Choreography one is not so obvious, because with elementary publish&#x2F;subscribe mechanism you can’t guarantee, that all steps will be undone.
Routing Slip
The solution here is to attach the list of steps and compensations to the message — this pattern called Routing Slip. Each service performs its action and sends to the next one from the list. Same with compensation. Thus, you use the network as a database. With this pattern, you can check if all steps complete or compensated.
Conclusion
Today, I’ve shown you how to coordinate services in the distributed system. Mainly, we have two options: choreography and orchestration. Also, we’ve considered an analogue of transaction in the distributed world - pattern Saga. In the next posts, I will demonstrate to you some examples of these pattern’s implementation.
Orchestration
Choreography
References
https:&#x2F;&#x2F;docs.microsoft.com&#x2F;en-us&#x2F;azure&#x2F;architecture&#x2F;patterns&#x2F;choreography
https:&#x2F;&#x2F;docs.microsoft.com&#x2F;en-us&#x2F;azure&#x2F;architecture&#x2F;reference-architectures&#x2F;saga&#x2F;saga
https:&#x2F;&#x2F;docs.microsoft.com&#x2F;en-us&#x2F;azure&#x2F;architecture&#x2F;patterns&#x2F;compensating-transaction
https:&#x2F;&#x2F;www.enterpriseintegrationpatterns.com&#x2F;ramblings&#x2F;18_starbucks.html
https:&#x2F;&#x2F;www.enterpriseintegrationpatterns.com&#x2F;patterns&#x2F;messaging&#x2F;ProcessManager.html
https:&#x2F;&#x2F;www.enterpriseintegrationpatterns.com&#x2F;patterns&#x2F;messaging&#x2F;RoutingTable.html
Image: Photo by Daria Nepriakhina on Unsplash*
<!--END_SECTION:feed-->

---

### 📊 Stats

[![GitHub stats](https://github-readme-stats.vercel.app/api?username=rafaelldi&show_icons=true&theme=nord)](https://github.com/rafaelldi/)

[![Top Langs](https://github-readme-stats.vercel.app/api/top-langs/?username=rafaelldi&layout=compact&theme=nord)](https://github.com/rafaelldi/)

