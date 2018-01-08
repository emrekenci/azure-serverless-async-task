# A serverless approach to async task management
## Overview
Here we present an easy way to:
* Create an API that accepts requests for task completion.
* A queue mechanism to decouple the app that requests the task and the app that completes the task.
* Create a way to manage tasks which could not be completed.

...using serverless technologies on Azure.

This approach can be applied to any scenario where a requested task is completed asynchrounously such as:

* Image processing
* File conversion
* Invoice generation
* Sending a non-transactional sms/email

...or any other task which doesn’t have to be completed at the time it was requested.

Here is an overview of what we’ll be covering:
![alt text](https://github.com/emrekenci/azure-serverless-async-task/raw/master/images/A%20serverless%20approach%20to%20async%20task%20management.png "A serverless approach to async task management")

The interesting thing about this approach isn't the architecture itself but how little **_code & effort_**  it takes to implement this on Azure. The rest of the article describes the implementation details.

## Queueing and dequeueing functions

We'll create two Azure Functions using Visual Studio. We **_could_** code the functions on the portal in Javascript or C# Script but the samples I'll share are written in C# on Visual Studio.

You can see the full solution for the functions [here](
https://github.com/emrekenci/azure-functions-servicebus-sample).

For some simplyfied code samples for Service Bus, you can check out [this repository](https://github.com/emrekenci/azure-servicebus-sample).

### HTTP Triggered Function (Queue tasks)

Simply writes the request body to Service Bus Queue as a message. Here is the code:
```csharp
[FunctionName("enqueue")]
public static async Task<HttpResponseMessage> Run([HttpTrigger(AuthorizationLevel.Function, "post")]HttpRequestMessage req, TraceWriter log)
{
    string requestBody = await req.Content.ReadAsStringAsync();

    string serviceBusConnectionString = Environment.GetEnvironmentVariable("ServiceBusConnectionString", EnvironmentVariableTarget.Process);
    string serviceBusQueueName = Environment.GetEnvironmentVariable("ServiceBusQueueName", EnvironmentVariableTarget.Process);

    IQueueClient queueClient = new QueueClient(serviceBusConnectionString, serviceBusQueueName);

    byte[] messageBytes = Encoding.ASCII.GetBytes(requestBody);

    var message = new Message(messageBytes);

    await queueClient.SendAsync(message);

    return req.CreateResponse(HttpStatusCode.OK);
}
```
The code above assumes your ServiceBus endpoint is already created. You can learn how to setup your ServiceBus Queue [here](
https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dotnet-get-started-with-queues).

Note that as soon as we deploy this function to Azure, we will have a functioning HTTP endpoint that's ready to be used.

Here is what this function **_saves_** us from doing:

* Creating an API project
* Setting up routing and the API endpoints
* Writing API authentication code
* Provisioning a web server or a web app.
 
 ### Queue Triggered Function (Complete tasks)
