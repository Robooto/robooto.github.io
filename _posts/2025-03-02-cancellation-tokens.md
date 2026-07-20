---
layout: post
title: "Cancellation Tokens: Best Practices"
---

# [](cancellation-tokens-best-practices) Cancellation Tokens: Best Practices

In .net core the framework supplies a cancellation token `HttpContext.RequestAborted or ServerCallContext.CancellationToken` that is automatically canceled if the client disconnects.  This token is rarely used and thus the .net core authors defaulted it to `CancellationToken.None`.  I was curious what happens when we don't use this token and when we do use this token when a user decides to not wait around for a call to go through.  I created a demo project that has a client api that then communicates with a gRPC service that can read from a db.  This involves 2 network calls and a db call [Example Project](https://github.com/Robooto/cancel-token)

What happens when a user leaves the page while a task is running?


### [](#what-happens)What Happens?

In this example project there is a simple grpc service that connects to a db, and an api using the grpc client.
Get the app running by using `docker-compose up --build`.
Head over to `http://localhost:8080/swagger/index.html` and hit the `GET /processWithout` endpoint.
![swagger](/assets/images/cancellation/swagger.png)
Now we want to refresh the page to simulate what happens when a user leaves.

![no-token](/assets/images/cancellation/no%20token.png)
```c#
app.MapGet("/processWithout", async (ILogger<Program> logger) =>
    {
        logger.LogInformation("Starting task delay without CancellationToken. ${DateTime.Now}", DateTime.Now);
        await Task.Delay(5000);
        logger.LogInformation("Task delay completed successfully. ${DateTime.Now}", DateTime.Now);
        return Results.Ok("Completed without CancellationToken");
    })
    .WithName("ProcessWithoutCancellation")
    .WithOpenApi();
```
Even though the user left the work still happens.  

Now lets see what happens with the cancellation token.

```c#
app.MapGet("/process", async (CancellationToken ct, ILogger<Program> logger) =>
    {
        try
        {
            logger.LogInformation("Starting task delay with CancellationToken.");
            await Task.Delay(10000, ct);
            return Results.Ok("Completed using CancellationToken");
        }
        catch (OperationCanceledException)
        {
            logger.LogWarning("Task delay was canceled.");
            return Results.Problem("Operation was canceled.");
        }
    })
    .WithName("ProcessWithCancellation")
    .WithOpenApi();
```


![token](/assets/images/cancellation/token.png)

What we can see this time is that our request did get canceled when the page was refreshed.  

### Propagating Cancellation to gRPC

Now that we've seen how `CancellationToken` works within a single HTTP request, what happens if that API client needs to make a network call to another microservice? 

In our demo project, the frontend API communicates with a backend gRPC service. To make sure cancellation propagates across the network, we must pass the token to our gRPC client call.

Here is the code in our API client endpoint:

```c#
app.MapGet("/grpc-hello-with-token", async (CancellationToken ct, ILogger<Program> logger) =>
    {
        try
        {
            using var channel = GrpcChannel.ForAddress("http://resource-access:5001");
            var client = new Greeter.GreeterClient(channel);
            
            // Forward the client's token here
            var response = await client.SayHelloAsync(new HelloRequest { Name = "World" }, cancellationToken: ct);
            return Results.Ok(response.Message);
        }
        catch (RpcException)
        {
            logger.LogWarning("Task delay was canceled.");
            return Results.Problem("Operation was canceled.");
        }
    })
    .WithName("GrpcHelloWithToken")
    .WithOpenApi();
```

By passing `cancellationToken: ct`, ASP.NET Core will automatically notify the gRPC client if the caller cancels their connection. The gRPC library then serializes this cancellation request and sends it over HTTP/2 to the backend server.

On the gRPC server implementation side (`GreeterService`), we can access this token directly through the `ServerCallContext`:

```c#
public override async Task<HelloReply> SayHello(HelloRequest request, ServerCallContext context)
{
    // Check for cancellation or pass the token to a long-running task.
    try
    {
        _logger.LogInformation("Processing gRPC request with cancellation capability.");
        
        // Pass context.CancellationToken down to any downstream async tasks
        await Task.Delay(2000, context.CancellationToken);

        return new HelloReply
        {
            Message = "Hello " + request.Name
        };
    }
    catch (OperationCanceledException)
    {
        _logger.LogWarning("Operation was canceled by the client.");
        throw new RpcException(new Status(StatusCode.Cancelled, "Operation was canceled by the client."));
    }
}
```

If the user disconnects, the gRPC server cancels the token, which instantly aborts `Task.Delay` and throws an `OperationCanceledException`. We then throw an `RpcException` with a `Cancelled` status back to the client.

---

### Canceling SQL Queries with Dapper

Network calls aren't the only resource-heavy operations we should cancel. Database queries are often the slowest part of a request, and they consume server connections and execution threads. 

Fortunately, `Microsoft.Data.SqlClient` and micro-ORMs like Dapper fully support cancellation.

Here is how we propagate the token all the way to our SQL Server query in the gRPC service:

```c#
public override async Task<SqlReply> SimulateSqlOperationWithCancellation(SqlRequest request, ServerCallContext context)
{
    _logger.LogInformation("Starting SQL operation with cancellation support.");
    try
    {
        using var connection = new SqlConnection(_configuration.GetConnectionString("DefaultConnection"));
        
        // 1. Pass token to OpenAsync
        await connection.OpenAsync(context.CancellationToken);
        
        // 2. Use CommandDefinition to pass the token to Dapper's execution methods
        var command = new CommandDefinition("WAITFOR DELAY '00:00:05'", cancellationToken: context.CancellationToken);
        await connection.ExecuteAsync(command);
        
        _logger.LogInformation("SQL operation completed successfully with cancellation support.");
        return new SqlReply { Message = "SQL operation completed with cancellation support." };
    }
    catch (OperationCanceledException)
    {
        _logger.LogWarning("SQL operation was canceled by the client.");
        return new SqlReply { Message = "SQL operation was canceled gracefully." };
    }
    catch (SqlException ex) when (ex.Message.Contains("cancel", StringComparison.OrdinalIgnoreCase))
    {
        _logger.LogWarning("SQL operation was canceled by the client (SqlException).");
        return new SqlReply { Message = "SQL operation was canceled gracefully." };
    }
}
```

Two things are critical here:
1. We pass `context.CancellationToken` to `OpenAsync` so we don't waste time waiting for a database connection if the user has already left.
2. We wrap our query in Dapper's `CommandDefinition` object, which takes the `cancellationToken`. Under the hood, this sends an attention signal to SQL Server to terminate the query execution, immediately releasing server resources.

Compare this to `SimulateSqlOperationWithoutCancellation`, where no tokens are passed:
```c#
public override async Task<SqlReply> SimulateSqlOperationWithoutCancellation(SqlRequest request, ServerCallContext context)
{
    _logger.LogInformation("Starting SQL operation without cancellation support.");
    using var connection = new SqlConnection(_configuration.GetConnectionString("DefaultConnection"));
    await connection.OpenAsync();
    await connection.ExecuteAsync("WAITFOR DELAY '00:00:05'");
    _logger.LogInformation("SQL operation completed successfully without cancellation support.");
    return new SqlReply { Message = "SQL operation completed without cancellation support." };
}
```
If a client disconnects during this call, SQL Server will continue executing the query for the full 5 seconds, holding up a connection and wasting CPU cycles.

---

### Best Practices Checklist

To ensure your applications remain performant and resilient under load, follow these best practices for cooperative cancellation:

- **Accept Tokens in Endpoints**: Always accept `CancellationToken` in your controller actions or minimal API route handlers. ASP.NET Core will inject `HttpContext.RequestAborted` automatically.
- **Propagate Everywhere**: Pass the token down to all asynchronous methods, including HTTP clients (`HttpClient`), gRPC clients, and databases.
- **Use CommandDefinition in Dapper**: Dapper's standard execution methods (like `ExecuteAsync`) don't have a direct cancellation token overload in some signatures. Always use `CommandDefinition` to bind your tokens.
- **Catch and Log Gracefully**: Always handle `OperationCanceledException` and database-specific query abort exceptions (`SqlException`) to prevent unhandled 500 errors in your application logs.

Happy Coding!

[Example Project](https://github.com/Robooto/cancel-token)
