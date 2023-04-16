---
layout: post
title: "GRPC Errors and Open Telemetry"
---

# [](#C#-grpc-errors-and-open-telemetry) C# GRPC Errors and Open Telemetry

Open Telemetry is a great tool for any systems using microservices.
The ability to see the flow of requests through your system is invaluable.
However, when you start using GRPC,
you'll find that the default Open Telemetry instrumentation doesn't capture the errors that are thrown by GRPC.
This is due to how the dot net team's grpc implementation.
To get around this, let's implement our own middleware to address this issue.

I've created a simple project that you can use to get started with this.
You can find it here: [example project](https://github.com/Robooto/grpc-open-tel-exceptions)

### [](#example-project)Example Project

In this example project, we have a jaeger instance for visualizing open telemetry,
a simple grpc service, and an api using the grpc client.
Get the app running by using `docker-compose up --build`.
Head over to `http://localhost:5001/swagger/index.html` and hit the `GET /Hello` endpoint.
Next open up jaeger at `http://localhost:16686/` and search for the `Hello` service.
You should see a trace with a single span.
If you click on the span, you should see the following:

![grpc-success](/assets/images/jaeger-success.png)

Great we can see our request.  Now let's implement the middleware to capture the errors.

```c#
public class ErrorInterceptor : Interceptor
{
    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        try
        {
            return await continuation(request, context).ConfigureAwait(false);
        }
        catch (Exception ex)
        {
            // This is a workaround for https://github.com/grpc/grpc-dotnet/issues/1407
            Activity.Current?.RecordException(ex);
            throw;
        }
    }
}
```

```c#
builder.Services.AddGrpc(options => { options.Interceptors.Add<ErrorInterceptor>();});
```

Now if we run the error request, we should see the following:

![grpc-error](/assets/images/jaeger-error.png)

Since we added the interceptor to the service, we can see the error in the span and easily debug our service.

Happy Coding!

[example project](https://github.com/Robooto/grpc-open-tel-exceptions)
