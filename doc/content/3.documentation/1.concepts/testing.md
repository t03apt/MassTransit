# Testing

MassTransit is an asynchronous framework that enables the development of high-performance and flexible distributed applications. Because of MassTransit's asynchronous underpinning, unit testing consumers, sagas, and routing slip activities can be significantly more complex. To simplify the creation of unit and integration tests, MassTransit includes a [Test Harness](/documentation/configuration/test-harness) that simplifies test creation.

## Test Harness Features

- Simplifies configuration for a majority of unit test scenarios
- Provides an in-memory transport, saga repository, and message scheduler 
- Exposes published, sent, and consumed messages
- Supports Web Application Factory for testing ASP.NET Applications

## Test Harness Concepts

As stated above, MassTransit is an asynchronous framework. In most cases, developers want to test that message consumption is successful, consumer behavior is as expected, and messages are published and/or sent. Because these actions are performed asynchronously, MassTransit's test harness exposes several asynchronous collections allowing test assertions verifying developer expectations. These asynchronous collections are backed by an over test timer and an inactivity timer, so it's important to use a test harness only once for a given scenario. Multiple test assertions, messages, and behaviors are normal in a given test, but unrelated scenarios should not share a single test harness.

MassTransit's test harness is built around Microsoft's Dependency Injection and is configured using the `AddMassTransitTestHarness` extension method. An test example is shown below, that verifies a `SubmitOrderConsumer` consumes a request and responds to the requester.

```csharp
[Test] 
public async Task An_example_unit_test() 
{
    await using var provider = new ServiceCollection()
        .AddYourBusinessServices() // register all of your normal business services
        .AddMassTransitTestHarness(x =>
        {
            x.AddConsumer<SubmitOrderConsumer>();
        })
        .BuildServiceProvider(true);

    var harness = provider.GetRequiredService<ITestHarness>();

    await harness.Start();

    var client = harness.GetRequestClient<SubmitOrder>();

    var response = await client.GetResponse<OrderSubmitted>(new
    {
        OrderId = InVar.Id,
        OrderNumber = "123"
    });

    Assert.IsTrue(await harness.Sent.Any<OrderSubmitted>());

    Assert.IsTrue(await harness.Consumed.Any<SubmitOrder>());

    var consumerHarness = harness.GetConsumerHarness<SubmitOrderConsumer>();

    Assert.That(await consumerHarness.Consumed.Any<SubmitOrder>());

    // test side effects of the SubmitOrderConsumer here
}
```

In the example above, the `AddMassTransitTestHarness` method is used to configure MassTransit, the in-memory transport, and the test harness on the service collection using all the default settings. This simple method is the same as the configuration shown below. The default settings eliminate all the extra code, simplifying the test set up.

```csharp
.AddMassTransitTestHarness(x =>
{
    x.AddDelayedMessageScheduler();
    
    x.AddConsumer<SubmitOrderConsumer>();
    
    x.UsingInMemory((context, cfg) =>
    {
        cfg.UseDelayedMessageScheduler();
        
        cfg.ConfigureEndpoints(context);
    });
})
```

#### Wait and Timeout

In the example above, the `await harness.Consumed.Any<SubmitOrder>()` method call waits for the `SubmitOrder` message to be consumed within the time specified by the `TestTimeout` property of the Test Harness. The default timeout is 30 seconds, and 50 minutes while debugging, configured by the [`TestHarnessOptions`](https://github.com/MassTransit/MassTransit/blob/develop/src/MassTransit/Configuration/DependencyInjection/TestHarnessOptions.cs#L9), and can be set:

```csharp
harness.TestTimeout = TimeSpan.FromSeconds(10);
```

#### Reading the values of messages Sent/Published/Consumed

To read the messages that were sent you can use the `Select` or `SelectAsync` methods:

```csharp
var messageSent = harness.Sent.Select<OrderSubmitted>()
    .FirstOrDefault();

Assert.AreEqual(orderId, messageSent.Context.Message.OrderId);
```

::callout{type="info"}
#summary
If you don't limit the number of messages to wait for, e.g. by using the `.FirstOrDefault()`, then the `Select` and `SelectAsync` methods **wait until the `TestTimeout`.** This means **50 minutes** by default while debugging, making the debugging experience unpleasant.
#content
For example, using Fluent Assertions the line below would be perfectly fine, but in this case it causes the test to block until the Test Timeout expires.

```csharp
// Waits for the full TestTimeout
❌ harness.Sent.Select<OrderSubmitted>()
      .Should().HaveCountGreaterThan(0);
```
::

### Transport Support

MassTransit's test harness can be used with any supported transport, and can also be used write rider integration tests (there is no in-memory rider implementation for unit testing). For example, to create an integration test using RabbitMQ, the RabbitMQ transport can be configured as shown.

```csharp
.AddMassTransitTestHarness(x =>
{
    x.AddConsumer<SubmitOrderConsumer>();
    
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.ConfigureEndpoints(context);
    });
})
```

In this example, RabbitMQ is configured using the default settings (which specifies a broker running on `localhost` using the default username and password of `guest`).


## Web Application Factory

MassTransit's test harness can be used with Microsoft's Web Application Factory, allowing unit and/or integration testing of ASP.NET applications. 

:sample{sample=web-application-factory}

To configure MassTransit's test harness for use with the Web Application Factory, call `AddMassTransitTestHarness` in the set up as shown below.

```csharp
await using var application = new WebApplicationFactory<Program>()
    .WithWebHostBuilder(builder => 
        builder.ConfigureServices(services => 
            services.AddMassTransitTestHarness()));

var testHarness = application.Services.GetTestHarness();

using var client = application.CreateClient();

var orderId = NewId.NextGuid();

var submitOrderResponse = await client.PostAsync("/Order", JsonContent.Create(new Order
{
    OrderId = orderId
}));

var consumerTestHarness = testHarness.GetConsumerHarness<SubmitOrderConsumer>();

Assert.That(await consumerTestHarness.Consumed.Any<SubmitOrder>(x => x.Context.Message.OrderId == orderId), Is.True);
```

## Examples

The following are examples of using the `TestHarness` to test various components.

### Consumer 

To test a consumer using the MassTransit Test Harness:

```csharp
[Test]
public async Task ASampleTest() 
{
    await using var provider = new ServiceCollection()
        .AddMassTransitTestHarness(cfg =>
        {
            cfg.AddConsumer<SubmitOrderConsumer>();
        })
        .BuildServiceProvider(true);

    var harness = provider.GetRequiredService<ITestHarness>();

    await harness.Start();

    var client = harness.GetRequestClient<SubmitOrder>();

    await client.GetResponse<OrderSubmitted>(new
    {
        OrderId = InVar.Id,
        OrderNumber = "123"
    });

    Assert.IsTrue(await harness.Sent.Any<OrderSubmitted>());

    Assert.IsTrue(await harness.Consumed.Any<SubmitOrder>());

    var consumerHarness = harness.GetConsumerHarness<SubmitOrderConsumer>();

    Assert.That(await consumerHarness.Consumed.Any<SubmitOrder>());

    // test side effects of the SubmitOrderConsumer here
}
```

### Request / Response

To test a Request using the MassTransit Test Harness:

```csharp
[Test]
public async Task ASampleTest()
{
    await using var provider = new ServiceCollection()
        .AddMassTransitTestHarness(cfg =>
        {
            cfg.Handler<SubmitOrder>(async cxt => 
            {
                await cxt.RespondAsync(new OrderResponse("OK"));
            });
        })
        .BuildServiceProvider(true);

    var harness = provider.GetRequiredService<ITestHarness>();

    await harness.Start();

    var client = harness.GetRequestClient<SubmitOrder>();

    var response = await client.GetResponse<OrderResponse>(new SubmitOrder
    {
        OrderNumber = "123"
    });

    Assert.That(response.Message.Status, Is.EqualTo("OK"));
}
```
[Short Addresses](https://masstransit.io/documentation/concepts/producers#short-addresses)

### Saga State Machine

To test a saga state machine using the MassTransit Test Harness:

```csharp
[Test]
public async Task ASampleTest()
{
    await using var provider = new ServiceCollection()
        .AddMassTransitTestHarness(cfg =>
        {
            cfg.AddSagaStateMachine<OrderStateMachine, OrderState>();
        })
        .BuildServiceProvider(true);

    var harness = provider.GetRequiredService<ITestHarness>();

    await harness.Start();

    var sagaId = Guid.NewGuid();
    var orderNumber = "ORDER123";

    await harness.Bus.Publish(new OrderSubmitted
    {
        CorrelationId = sagaId,
        OrderNumber = orderNumber
    });

    Assert.That(await harness.Consumed.Any<OrderSubmitted>());

    var sagaHarness = harness.GetSagaStateMachineHarness<OrderStateMachine, OrderState>();

    Assert.That(await sagaHarness.Consumed.Any<OrderSubmitted>());

    Assert.That(await sagaHarness.Created.Any(x => x.CorrelationId == sagaId));

    var instance = sagaHarness.Created.ContainsInState(sagaId, sagaHarness.StateMachine, sagaHarness.StateMachine.Submitted);
    Assert.IsNotNull(instance, "Saga instance not found");
    Assert.That(instance.OrderNumber, Is.EqualTo(orderNumber));

    Assert.IsTrue(await harness.Published.Any<OrderApprovalRequired>());

    // test side effects of OrderState here
}
```
