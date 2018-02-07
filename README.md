These are the C# coding standards and practices for new (non-legacy) code. This is a living document. Please discuss on #coding-standards and send a pull request if you would like to contribute.

## Overall philosophy: KISS

_Keep It Simple, Stupid_

Systems work best if they are kept simple. Therefore, simplicity should be a key goal in design, and unnecessary complexity should be avoided.

## Exceptions

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
  - Either: don't `catch` the exception in the first place.
  - Else: rethrow the exception so it can be handled further up the stack by something that can.
- There should always be a global exception handler that logs the exception and shows a "sorry, something went wrong" message to the user. If all you want to do is log the error, then don't bother - keep your code clean and let the global exception handler do its thing.
- When _rethrowing_ an exception simply `throw;` it. Do not `throw ex;` as this loses the call stack information, making it look like the exception originated inside your `catch` block.

**Note that simply logging the exception does not usually count as gracefully recovering. If in doubt, always rethrow.**

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