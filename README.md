# Mediator

**Welcome!**  

A lightweight implementation of the Mediator pattern to enable loosely coupled communication between components. Supports commands, queries, and events for clean, maintainable architecture. Easily extensible for various use cases.

This public repository is dedicated to discussions, issue tracking, and community support for the Mediator library.

> **Note:**  
> The source code for Mediator is not published in this repository. This space is intended for feedback, questions, bug reports, and feature suggestions.

## What You Can Do Here

- Open issues for bug reports or feature requests
- Ask questions or start discussions about usage, patterns, or best practices
- View updates on releases and planned features

## Where is the Source Code?

The source code is not hosted in this repository.  
If you are interested in contributing or accessing the source, please refer to the project documentation or contact the maintainer directly.

## Get in Touch

- [Open an issue](https://github.com/ilnairb/Mediator/issues)
- [Start a discussion](https://github.com/ilnairb/Mediator/discussions)

Thank you for being part of the Mediator community!

# Mediator Quick Start Guide

## 1. Installation & Registration

**Install this nuget package** in your solution.

### Automatic Handler Registration

Register handlers, behaviors, validators, and the mediator with assembly scanning:

```csharp
// Register everything in Startup.cs or Program.cs
services.AddMediator(assemblies: new[] { typeof(PingCommand).Assembly });
// Or, scan all loaded assemblies
services.AddMediator(AppDomain.CurrentDomain.GetAssemblies());
```

AddMediator will automatically find and register:

- Command/Query hanlders and Notification handlers
- Validators

For behaviors, you need to manually register them in your start up program 

```csharp
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>)); // Add logging behavior for Mediator
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>)); // Add validation behavior for Mediator
```

For decorators, you aslo need to register them manually in your start up program

```csharp
// the handler type needs to be registered separately so we can wrap it with the decorator
builder.Services.AddTransient<CreateOrderCommandHandler>();
// Register the decorator as the handler interface, wrapping the actual handler
builder.Services.AddTransient<IRequestHandler<CreateOrderCommand, int>>(sp =>
    new RetryDecorator<CreateOrderCommand, int>(
        sp.GetRequiredService<CreateOrderCommandHandler>(), // inner handler
        maxRetries: 3 // or your preferred retry count
    )
);
```

## 2. Defineing REquests, Handlers and Validators

### Commands, Queries, Notifications

```csharp
// Command (write)
public class PingCommand : ICommand<string>
{
    public string Message { get; set; }
}

// Query (read)
public class GetTimeQuery : IQuery<DateTime> { }

// Notification (event)
public class UserRegisteredNotification : INotification
{
    public string Email { get; set; }
}
```

### Handlers

```csharp
// Command handler
public class PingCommandHandler : ICommandHandler<PingCommand, string>
{
    public Task<string> Handle(PingCommand command, CancellationToken cancellationToken)
        => Task.FromResult($"Pong: {command.Message}");
}

// Query handler
public class GetTimeQueryHandler : IQueryHandler<GetTimeQuery, DateTime>
{
    public Task<DateTime> Handle(GetTimeQuery query, CancellationToken cancellationToken)
        => Task.FromResult(DateTime.UtcNow);
}

// Notification handler
public class UserRegisteredHandler : INotificationHandler<UserRegisteredNotification>
{
    public Task Handle(UserRegisteredNotification notification, CancellationToken cancellationToken)
    {
        Console.WriteLine($"Welcome email sent to {notification.Email}!");
        return Task.CompletedTask;
    }
}
```

### Validation

```csharp
public class PingCommandValidator : IValidator<PingCommand>
{
    public ValidationResult Validate(PingCommand command)
    {
        if (string.IsNullOrWhiteSpace(command.Message))
            return ValidationResult.Failed("Message cannot be empty.");
        return ValidationResult.Success;
    }
}

```
Or you can extend the ValidatorBase class, which the ValidateAsync method calls the Validate method

## 3. Using the Mediator

Inject IMediator via DI:

```csharp
public class MyService
{
    private readonly IMediator _mediator;
    public MyService(IMediator mediator) => _mediator = mediator;

    public Task<string> PingAsync(string msg)
        => _mediator.Send(new PingCommand { Message = msg });

    public Task<DateTime> GetTimeAsync()
        => _mediator.Send(new GetTimeQuery());

    public Task RegisterUserAsync(string email)
        => _mediator.Publish(new UserRegisteredNotification { Email = email });
}
```

## 4. Decorators & Pipeline Behaviors

Decorators:
Register your decorator as a handler or use DI container features. The mediator supports handler/behavior decoration.

Pipeline Behaviors:
Add cross-cutting logic (logging, validation, etc).

```charp
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        CancellationToken cancellationToken,
        RequestHandlerDelegate<TResponse> next)
    {
        Console.WriteLine($"[LOG] Handling {typeof(TRequest).Name}");
        var response = await next();
        Console.WriteLine($"[LOG] Handled {typeof(TRequest).Name}");
        return response;
    }
}

// Register manually (if not using scanning)
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
```

## 5. Example: Hello World

```charp
// Command, handler, and validator
public record HelloCommand(string Name) : ICommand<string>;

public class HelloCommandHandler : ICommandHandler<HelloCommand, string>
{
    public Task<string> Handle(HelloCommand cmd, CancellationToken ct)
        => Task.FromResult($"Hello, {cmd.Name}!");
}

public class HelloValidator : IValidator<HelloCommand>
{
    public ValidationResult Validate(HelloCommand cmd)
        => string.IsNullOrWhiteSpace(cmd.Name)
            ? ValidationResult.Failed("Name required.")
            : ValidationResult.Success;
}

// In DI setup:
services.AddMediator(new[] { typeof(HelloCommand).Assembly });

// Usage:
var result = await mediator.Send(new HelloCommand("Mediator")); // "Hello, Mediator!"
```

## 6. Advanced

- Serial notifications: Use ISerialNotification for sequential notification handling.
- Decorator chaining: Register multiple decorators for layered logic.
- Custom behaviors: Implement your own IPipelineBehavior<TRequest, TResponse>.
