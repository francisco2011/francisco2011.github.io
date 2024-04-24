
Hello there, mate! It's a quite common task to add some logging behavior to your solutions, but depending on constraints like time, budgets, or the type and culture of the company you are working on, sometimes it will be difficult to add characteristics that sum up to the overall quality of the solution, nevertheless not considered part of the functional design. That's why I've seen many times big projects that lack any logging characteristic and  I didn't have a clear view on how to solve this situation until I discovered AOP and interceptors (hopefully I will dive into AOP in another opportunity).

Today, I will present a simple and reusable logging interceptor that's easy to write and understand, at the same time it will fit with ease in any. Net/Core project. First things first...

## ¿What is an interceptor?

In simple terms an Interceptor is a component that's able to inspect a message before it is delivered to its objective, this kind of components usually are designed to handle logging, transactions, authorization, exception handling, caching, rules validations, basically any type of cross-cutting concern in your software.

## ¿What will be implemented?

An interceptor that tracks and logs the parameters and result from the execution of the methods of an assigned object, furthermore I will provide some useful structures

### ¿What tools will we need?

- Castle.Core
- Castle.Core.AsyncInterceptor
- Serilog 
- ASP.net Core MVC web API

I reckon that Serilog and the web API are not necessary, but useful because asp.net MVC already includes a good enough DI module,  Swagger integration, and other characteristics that will be used in prior entries. On the other hand, Serilog allows you to easily create structured logs and send them to a  wide range of platforms. 

## Basic usage

### 1.-For this instance I will declare a simple class with one method and an interface for that class:

 ```
  public interface ISomeLogicService
    {
        Task DoSomeStuff(string who);
    }
  
 public class SomeLogicService : ISomeLogicService
    {
        public async Task DoSomeStuff(string who)
        {
            return await Task.Run(() => { return $"Hello {who}";});
        }
    }

 ```
 
Later we will use the interface to limit the scope of interception.

### 2.-Download and install Castle.Core and Castle.Core.AsyncInterceptor from Nuget 

[https://www.nuget.org/packages/castle.core/]()
[https://www.nuget.org/packages/Castle.Core.AsyncInterceptor/]()

### 3.-Implement IAsyncInterceptor, this interface contains the needed methods to intercept asynchronous calls. A plain and straightforward implementation could look like this:
  
  		 ```
        public void InterceptAsynchronous(IInvocation invocation)
        {
            invocation.ReturnValue = InternalInterceptAsynchronous(invocation);
        }
        public async Task InternalInterceptAsynchronous<TResult>(IInvocation invocation)
        {
            try
            {
                var arguments = invocation.Arguments;

                write(arguments, invocation);

                invocation.Proceed();
                var result = await (Task<TResult>)invocation.ReturnValue;

                var invocationResultArg = (result != null ? new object[1] { result as object } : invocation.Arguments);

                write(invocationResultArg, invocation);

                return result;

            }
            catch(Exception ex)
            {
                Logger.Error(ex, LOGTEMPLATE);
                throw;
            }
        }
        private void write(Object[] parameters, IInvocation invocation)
        {
            var logModel = createLogModel(parameters, invocation.Method.Name);
            Logger.Information(LOGTEMPLATE, logModel);

        }

         ```
        
### 4.-An example of instantiation using the DI feature of .Net Core MVC. The code is just a convenience and not production-ready instructions:

 ```
  services.AddSingleton<ProxyGenerator>();
  
  services.AddScoped<ISomeLogicService>(c =>
  {
        var target = new SomeLogicService();
        return c.GetService().CreateInterfaceProxyWithTargetInterface(typeof(ISomeLogicService), new SomeLogicService(), new 	BasicParamLogger()) as ISomeLogicService;
  });

 ```

### Aside Notes:

- I noticed that the memory consumed by the constant creation of ProxyGenerator objects can be a problem, so the best option is to make the object singleton.
- Normally I would use CreateInterfaceProxyWithTargetInterface because it's simple, but if you want to intercept all the calls to the wrapped object other options are available, or even use IInterceptorSelector if you need better alternatives to define an interface.


## What can we do next?

At this moment we already defined the basics to nicely log the execution of some objects, but from the standpoint of view of logging we still must define the level for each case of execution, this functionality can be implemented directly in the interceptor code and always execute to the same level, nevertheless, this can reduce the number of cases in which your interceptor can operate, therefore I will extend the implementation with an attribute that will allow me to define the level and in the future new configuration parameters.

### 1.-Declare an attribute

 ```
  [AttributeUsage(AttributeTargets.Interface | AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
    public class LogOptionsAttribute : Attribute
    {
        public LogEventLevel Level { get; set; }
        public LogOptionsAttribute(LogEventLevel level)
        {
            Level = level;
        }
    }
 ```

### 2.-add the attribute to the ISomeLogicService method

 ```
 [LogOptionsAttribute(LogEventLevel.Debug)]
    public interface ISomeLogicService
    {
        Task DoSomeStuff(string who);
    }
 ```
 
### 3.-Create a helper and an extension method that will contain the instructions to establish the log level. 

The first method simply returns the configured minimum log level and the second one looks for the existence of LogOptionsAttribute, if the attribute wasn't used then the already selected level will be used.

 ```

public static class LoggerExtensions
    {
        public static LogEventLevel GetLevel(this ILogger logger)
        {
            if (logger.IsEnabled(LogEventLevel.Verbose)) return LogEventLevel.Verbose;
            if (logger.IsEnabled(LogEventLevel.Debug)) return LogEventLevel.Debug;
            if (logger.IsEnabled(LogEventLevel.Information)) return LogEventLevel.Information;
            if (logger.IsEnabled(LogEventLevel.Warning)) return LogEventLevel.Warning;
            if (logger.IsEnabled(LogEventLevel.Error)) return LogEventLevel.Error;
            return LogEventLevel.Fatal;
        }
    }

public class LogHelper
    {
        public LogEventLevel GetLevel(MemberInfo memberInfo, ILogger logger)
        {
            var attribute = memberInfo.DeclaringType.GetCustomAttribute<LogOptionsAttribute>(true);
            if (attribute != null) return attribute.Level;
            return logger.GetLevel();

        }
    }

 ```

### 4.-Finally, update the method write from the interceptor to make use of the new helper.

 ``` 
private void write(Object[] parameters, IInvocation invocation)
{
  var logModel = createLogModel(parameters, invocation.Method.Name);
  var level = Helper.GetLevel(invocation.Method, Logger);
  Logger.Write(level, LOGTEMPLATE, logModel);
}

 ```

## Aside notes:

- The attribute lookup can be further improved. I seriously doubt it will last complex use cases.
- Log level can be changed in execution time and this fact should be considered when designing the logs structure.
- Probably, you will want to design different interceptors for different levels of logging and flows, for instance, errors and exceptions deserve a little more respect than I showed here.


## Final thoughts

I wish I knew this 3 years ago and I wish to see more software developers considering this type of tools in their designs, mainly because as I stated in this entry implementing interceptors is not that hard and they will give you the power to quickly add new features to your systems without making big changes. Logging is a feature that can save your life!
