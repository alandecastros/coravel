# Coravel

Inspired by all the awesome features that are baked into the Laravel PHP framework - coravel seeks to provide additional features that .Net Core lacks like:

- Task Scheduling
- Queuing
- Mailer [TBA]
- Command line tools integrated with coraval features [TBA]
- More???

## Features

- [Task Scheduling](https://github.com/jamesmh/coravel#task-scheduling)
- [Queuing](https://github.com/jamesmh/coravel#feature-task-queuing)

## Quick-Start

Add the nuget package `Coravel` to your .NET Core app. Done!

### 1. Scheduling Tasks

Tired of using cron and Windows Task Scheduler? Want to use something easy that ties into your existing code?

In `Startup.cs`, put this in `ConfigureServices()`:

```c#
services.AddScheduler(scheduler =>
    {
        scheduler.Schedule(
            () => Console.WriteLine("Run at 1pm utc during week days.")
        )
        .DailyAt(13, 00)
        .Weekday();
    }
);
```

For async tasks you may use `ScheduleAsync()`:

```c#
scheduler.ScheduleAsync(async () =>
{
    await Task.Delay(500);
    Console.WriteLine("async task");
})
.EveryMinute();
```

Easy enough? Look at the documentation to see what methods are available!

### 2. Task Queuing

Tired of having to install and configure other systems to get a queuing system up-and-running? Tired of using databases to issue queued tasks? Look no further!

In `Startup.cs`, put this in `ConfigureServices()`:

```c#
services.AddQueue();
```

Voila! Too hard?

To append to the queue, inject `IQueue` into your controller (or wherever injection occurs):

```c#
using Coravel.Queuing.Interfaces; // Don't forget this!

// Now inside your MVC Controller...
IQueue _queue;

public HomeController(IQueue queue) {
    this._queue = queue;
}
```

And then call:

```c#
this._queue.QueueTask(() =>
    Console.WriteLine("This was queued!")
);
```

Or, for async tasks:

```c#
this._queue.QueueAsyncTask(async () =>
{
    await Task.Delay(100);
    Console.WriteLine("This was queued!")
});
```

Now you have a fully functional queue!

## Task Scheduling

Usually, you have to configure a cron job or a task via Windows Task Scheduler to get a single or multiple re-occuring tasks to run. With coravel you can setup all your scheduled tasks in one place! And it's super easy to use!

### Initial Setup

In your .NET Core app's `Startup.cs` file, inside the `ConfigureServices()` method, add the following:

```c#
services.AddScheduler(scheduler =>
    {
        scheduler.Schedule(
            () => Console.WriteLine("Every minute during the week.")
        )
        .EveryMinute();
        .Weekday();
    }
);
```

This will run the task (which prints to the console) every minute and only on weekdays (not Sat or Sun). Simple enough?

### How Does It Work?

The `AddScheduler()` method will configure a new Hosted Service that will run in the background while your app is running.

A `Scheduler` is provided to you for configuring what tasks you want to schedule. You may use the `Schedule()` and `ScheduleAsync()` methods to schedule a task.

After calling `Schedule()` you can chain method calls further to specify:

- The interval of when your task should be run (once a minute? every hour? etc.)
- Specific times when you want your task to run
- Restricting which days your task is allowed to run on (Monday's only? etc.)

Example: Run a task once an hour only on Mondays.

```c#
scheduler.Schedule(
    () => Console.WriteLine("Hourly on Mondays.")
)
.Hourly()
.Monday();
```

Example: Run a task every day at 1pm

```c#
scheduler.Schedule(
    () => Console.WriteLine("Daily at 1 pm.")
)
.DailyAtHour(13); // Or .DailyAt(13, 00)
```

### Global Error Handling

Any tasks that throw errors __will just be skipped__ and the next task in line will be invoked.

If you want to catch errors and do something specific with them you may use the `OnError()` method.

```c#
services.AddScheduler(scheduler =>
    // Assign your schedules
)
.OnError((exception) =>
    doSomethingWithException(exception)
);
```

You can, of course, add error handling inside your specific tasks too.

### Scheduling Tasks

After you have called the `Schedule()` method on the `Scheduler`, you can begin to configure the schedule constraints of your task.

```c#
scheduler.Schedule(
    () => Console.WriteLine("Scheduled task.")
)
.EveryMinute();
```

### Scheduling Async Tasks

Coravel will also handle scheduling async methods by using the `ScheduleAsync()` method. Note that this doesn't need to be awaited - the method or Func you provide _itself_ must be async (as it will be invoked by the scheduler at a later time).

```c#
scheduler.ScheduleAsync(async () =>
{
    await Task.Delay(500);
    Console.WriteLine("async task");
})
.EveryMinute();
```

Note, that you are able to register an async method when using `Schedule()` by mistake. Always use `ScheduleAsync()` when registering an async method.

#### Intervals

First, methods to apply interval constraints are available.

##### Basic Intervals

These methods tell your task to execute at basic intervals.

Using any of these methods will cause the task to be executed immedately after your app has started. Then they will only be
executed again once the specific interval has been reached.

If you restart your app these methods will cause all tasks to run again on start. To avoid this, use an interval method with time constraints (see below).

- `EveryMinute();`
- `EveryFiveMinutes();`
- `EveryTenMinutes();`
- `EveryFifteenMinutes();`
- `EveryThirtyMinutes();`
- `Hourly();`
- `Daily();`
- `Weekly();`

##### Intervals With Time Contraints

These methods allow you specify an interval and a time constraint so that your scheduling is more specific and consistent.

_Please note that the scheduler is using UTC time. So, for example, using `DailyAt(13, 00)` will run your task daily at 1pm UTC time._

- `HourlyAt(int minute)`
- `DailyAtHour(int hour)`
- `DailyAt(int hour, int minute)`

#### Day Constraints

After specifying an interval, you can further chain to restrict what day(s) the scheduled task is allowed to run on.

All these methods are further chainable - like `Monday().Wednesday()`. This would mean only running the task on Mondays and Wednesdays. Be careful since you could do something like this `.Weekend().Weekday()` which basically means there are no constraints (it runs on any day).

- `Monday()`
- `Tuesday()`
- `Wednesday()`
- `Thursday()`
- `Friday()`
- `Saturday()`
- `Sunday()`
- `Weekday()`
- `Weekend()`

## Feature: Task Queuing

Coravel allows zero-configuration queues (at run time).

Every 30 seconds, if there are any available items in the queue, they will begin invocation.

_Note: The queue is a separate Hosted Service from the scheduler (i.e. they each run in isolation)_

### Setup

In your `Startup` file, in the `ConfigureServices()` just do this:

```c#
services.AddQueue();  
```

That's it! This will automatically register the queue in your service container.

### Global Error Handling

The `OnError()` extension method can be called after `AddQueue` to register a global error handler.

```c#
services
    .AddQueue()
    .OnError(e =>
    {
        //.... handle the error
    });
```

### How To Queue Tasks

In your controller that is using DI, inject a `Coravel.Queuing.Interfaces.IQueue`.

You use the `QueueTask()` method to add a task to the queue.

```c#
IQueue _queue;

public HomeController(IQueue queue) {
    this._queue = queue;
}

//... Further down ...

public IActionResult QueueTask() {
    // Call .QueueTask() to add item to the queue!
    this._queue.QueueTask(() => Console.WriteLine("This was queued!"));
    return Ok();
}
```

### Queue Async Task

Use the `QueueAsyncTask` to queue up an async Task (which will run async whenever the queue is consumed).

```c#
 this._queue.QueueAsyncTask(async() => {
    await Task.Delay(1000);
    Console.WriteLine("This was queued!");
 };
```

### On App Closing

When your app is stopped, coravel will attempt to gracefully wait until the last moment and:

- Run the scheduler once last time
- Consume any tasks remaining in the queue

You shouldn't have to worry about loosing any queued items.

If your server was shutdown in a non-graceful way etc. (unplugged... etc.) then you may lose active queued tasks. But under normal circumstances, even when forcefully shutting down your app, coravel will (in the background) handle this for you.

_Note: Queue persistance might be added in the future ;)_
