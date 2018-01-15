---
date: 2015-07-07T00:00:00Z
title: Async-await and WaitHandles
--- 

The TPL is great but sometimes you have to work within a framework which schedules background tasks for you 
without giving you access to a `Task` of some kind. 
I had a situation like the following

{{< highlight csharp >}}

private void DoSomeWork(CancellationToken token)
{
    token.ThrowIfCancellationRequested();
    // do lots of work 
}

// some other code that might happen in a UI thread
AsyncWorkScheduler.Schedule(()=> DoSomeWork(token)); 

{{< / highlight >}}

I didn't really have access to how the background task was scheduled, and to be honest I didn't really care.
What I needed was a way to wait for the work to finish, whether it finished by itself or if it was
cancelled via the `CancellationToken`. 
I wanted to use `await` to avoid blocking any UI threads as well.

I could use the `AutoResetEvent` as a way to perform cross-thread signaling to indicate the work is done. 
Simple enough

{{< highlight csharp >}}

private readonly AutoResetEvent _autoReset = new AutoResetEvent(false);

private void DoSomeWork(CancellationToken token) 
{
    var cancellation = (CancellationToken)obj;
    try
    {               
        token.ThrowIfCancellationRequested();
        // do lots of work 
    }
    finally
    { 
        // yay! All done
        _autoReset.Set();
    }
}

{{< / highlight >}}

So how would you use await with an `AutoResetEvent`?

{{< highlight csharp >}}

/// <summary>
/// Create a task based on the wait handle, which is the base class
/// for quite a few synchronization primitives
/// </summary>
/// <param name="handle">The wait handle</param>
/// <returns>A valid task</returns>
private static Task WaitHandleAsTask(WaitHandle handle)
{
    return WaitHandleAsTask(handle, Timeout.InfiniteTimeSpan);
}

/// <summary>
/// Create a task based on the wait handle, which is the base class
/// for quite a few synchronization primitives
/// </summary>
/// <param name="handle">The wait handle</param>
/// <param name="timeout">The timeout</param>
/// <returns>A valid task</returns>
private static Task WaitHandleAsTask(WaitHandle handle, TimeSpan timeout)
{
    var tcs = new TaskCompletionSource<object>();
    var registration = ThreadPool.RegisterWaitForSingleObject(handle, (state, timedOut) =>
    {
        var localTcs = (TaskCompletionSource<object>)state;
        if (timedOut)
        {   
            localTcs.SetCanceled();
        }   
        else
        {
            localTcs.SetResult(null);
        }
    }, tcs, timeout, true);

    // clean up the RegisterWaitHandle
    tcs.Task.ContinueWith((_, state) => ((RegisteredWaitHandle)state).Unregister(null), registration, TaskScheduler.Default); 
    return tcs.Task;
}

{{< / highlight >}}

`ThreadPool.RegisterWaitForSingleObject` performs a wait in a thread pool thread, and when the handle is signal, the delegate executes
executes in a worker thread. This will signal the task completion.

So your wait operation ends up looking like


{{< highlight csharp >}}
void async SomeAsyncMethod(){
// .. stuff and work
await WaitHandleAsTask(_autoReset);
// .. more stuff and work
}
{{< / highlight >}}

Pretty neat eh? 
