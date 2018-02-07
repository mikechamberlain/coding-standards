These are the C# coding standards and practices for new (non-legacy) code. This is a living document. Please discuss on #coding-standards and send a pull request if you would like to contribute.

## Overall philosophy: KISS

_Keep It Simple, Stupid_

Systems work best if they are kept simple. Therefore, simplicity should be a key goal in design, and unnecessary complexity should be avoided.

## Services, interfaces and DI

- Only create an interface for your class if at least one of the following is true:
  1. Multiple implementations exist in application code.
  2. The service needs to be mocked to make its dependencies testable.
- A service only needs to be mocked if it is _impure_, eg:
  - it has a side effects such as making an external call (eg. http)
  - it is non-deterministic - that is, when called multiple times with the same input, different results may be returned (eg. `DateTime.Now`).
- An impure service is a good candidate for dependency injection. This promotes testability and keeps components loosely coupled.
- A pure service doesn't need to be injected. Prefer to create and access it as a static method instead.
- Keep each component's dependency count to a reasonable level.
  - Declaring only impure services as dependencies will help.
  - If a component's dependency count is getting out of control, consider refactoring the component into more narrowly focused responsibilities. 

For example, say we have a model that needs to be mapped to a view model:

```c#
public class MyModel
{
    public int Id { get; set; }
}

public class MyViewModel
{
    public int Id { get; set; }
}
```

### Don't

```c#
// ugh, so much code...

interface IViewModelMapper<TFrom, TTo>
{
    TTo Map(TFrom from);
}

public class MyViewModelMapper : IViewModelMapper<MyModel, MyViewMOdel>
{
    public MyViewModel Map(MyModel model)
    {
        return new MyViewModel
        {
            Id = model.Id
        };
    }
}

public class MyController
{
    private MyViewModelMapper mapper;
    private IMyService service;
    
    public MyService(MyViewModelMapper mapper, IMyService service)
    {
        this.mapper = mapper;
        this.service = service;
    }
    
    public MyViewModel Get()
    {
        var model = service.GetModel();
        var viewModel = mapper.Map(model);
        return Json(viewModel);
    }
```

### Do

```c#

// so much simpler

public class MyViewModel
{
    // this method is pure, doesn't need to be mocked, so just make it static
    public static MyViewModel From(MyModel model)
    {
        return new MyViewModel
        {
            Id = model.Id
        };
    }      
}

public class MyController : ApiController
{
    private IMyService service;
    
    // we removed an unneccesarry dependency!
    public MyService(IMyService service)
    {
        this.service = service;
    }
    
    public MyViewModel Get()
    {
        var model = service.GetModel();
        var viewModel = MyViewModel.From(model);
        return Json(viewModel);
    }
}
```

## Errors and exceptions

- Fail as quickly and as loudly as possible. This gives us the greatest chance of becoming aware of the problem or bug, meaning we can take steps to fix it. Do not be tempted to hide exceptions from the user simply to "improve" the UX. This just leads to long-term difficult-to-diagnose inconsistencies.
- Do not _catch_ an exception unless you have a good reason to do so. Such reasons might include one or more of:
   - recovering from the error
   - enriching the error message / wrapping the exception
   - retrying the operation
   - logging the error
- When catching an exception, be as specific as possible to the type of exception you are handling. Avoid catching `System.Exception` if you only plan to handle `System.IO.FileNotFoundException`. 
- Do not _swallow_ an exception unless you have handled the error and can gracefully continue. The hardest exceptions to troubleshoot are the ones that don't even exist, because someone upstream decided to swallow it.
- Only swallow an exception if you have taken steps to correct the problem and have brought the system back into a consistent state. For example, if an exception is thrown when writing to a readonly file then you can recover by removing the readonly attribute.
- If you cannot recover from an error, it's totally fine!
  - Either: don't catch the exception in the first place.
  - Else: rethrow the exception so it can be handled further up the stack by something that can.
- There should always be a global exception handler that logs the exception and shows a "sorry, something went wrong" message to the user. If all you want to do is log the error, then don't bother - keep your code clean and let the global exception handler do its thing.
- When _rethrowing_ an exception simply `throw;` it. Do not `throw ex;` as this loses the call stack information, making it look like the exception originated inside your `catch` block.

**Note that simply logging the exception does not count as graceful recovery. If in doubt, always rethrow.**

### Don't

```c#
try
{
    return myService.GetData();
}
catch (Exception ex)
{
    Log.Error(ex.ToString());
    return null;
    // Now the caller has to deal with a myserious null value, with no indication 
    // that something went wrong.
}

```

### Do

```c#
try
{
    return myService.GetData();
}
catch (MyServiceException ex)
{
    Log.Error(ex.ToString());
    throw;
    // This time, the caller is notified that something went wrong, and can take 
    // steps to recover, else allow the exception to propagate up the stack. 
}

```

## nulls

_"The billion dollar mistake"_

Avoid nulls where possible, because they:
- subvert types
- are sloppy
- present a special case
- make poor APIs
- exacerbate poor language decisions
- are difficult to debug
- are non-composable

Consider using an Option/Maybe type to represent the potential absence of a value. 

[Read more here (seriously)](https://www.lucidchart.com/techblog/2015/08/31/the-worst-mistake-of-computer-science/).

### Don't

```c#
public List<int> GetPropertyIds(int hostId)
{
    var properties = propertyService.GetProperties(hostId);
    
    if (properties == null || !properties.Any()) 
    {
        return null; 
        // NO! Now the caller has to somehow know to deal with this special case.
        // You are asking for a NullReferenceException in prod. Hope you like 
        // 3am wakeup calls.
    }
    
    return properties.Select(p => p.Id).ToList();
}
```

### Do

```c#
public List<int> GetPropertyIds(int hostId)
{
    var properties = propertyService.GetProperties(hostId);
    
    if (properties == null) 
    {
        // Just return an empty List and everything should just work.
        // Even better: fix propertyService.GetProperties() to not return null itself.
        return Enumerable.Empty<int>().ToList();
    }
    
    return properties.Select(p => p.Id).ToList();
}
```