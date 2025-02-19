Bolts (Without App Links)
============
[![Build Status](https://img.shields.io/travis/BoltsFramework/Bolts-ObjC/master.svg?style=flat)](https://travis-ci.org/BoltsFramework/Bolts-ObjC)
[![Coverage Status](https://codecov.io/github/BoltsFramework/Bolts-ObjC/coverage.svg?branch=master)](https://codecov.io/github/BoltsFramework/Bolts-ObjC?branch=master)
[![Pod Platform](https://img.shields.io/cocoapods/p/Bolts.svg?style=flat)](https://cocoapods.org/pods/Bolts)
[![Pod License](https://img.shields.io/cocoapods/l/Bolts.svg?style=flat)](https://github.com/BoltsFramework/Bolts-ObjC/blob/master/LICENSE)
[![Reference Status](https://www.versioneye.com/objective-c/bolts/reference_badge.svg?style=flat)](https://www.versioneye.com/objective-c/bolts/references)

[![Pod Version](https://img.shields.io/cocoapods/v/Bolts.svg?style=flat)](https://cocoapods.org/pods/Bolts)
[![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)

**Note:**
Bolts framework uses UIWebView for App Links feature which is not required for Amazon Fling SDK used for FireTV Casting.
UIWebView is deprecated by Apple and customers are seeing warning about it when submitting their apps to Appstore.

Bolts is a collection of low-level libraries designed to make developing mobile
apps easier. Bolts was designed by Parse and Facebook for our own internal use,
and we have decided to open source these libraries to make them available to
others. Using these libraries does not require using any Parse services. Nor
do they require having a Parse or Facebook developer account.

Bolts includes:

* "Tasks", which make organization of complex asynchronous code more manageable. A task is kind of like a JavaScript Promise, but available for iOS and Android.
* An implementation of the [App Links protocol](http://applinks.org/), helping you link to content in other apps and handle incoming deep-links.

For more information, see the [Bolts iOS API Reference](http://boltsframework.github.io/docs/ios/).

# Tasks

To build a truly responsive iOS application, you must keep long-running operations off of the UI thread, and be careful to avoid blocking anything the UI thread might be waiting on. This means you will need to execute various operations in the background. To make this easier, we've added a class called `BFTask`. A task represents the result of an asynchronous operation. Typically, a `BFTask` is returned from an asynchronous function and gives the ability to continue processing the result of the task. When a task is returned from a function, it's already begun doing its job. A task is not tied to a particular threading model: it represents the work being done, not where it is executing. Tasks have many advantages over other methods of asynchronous programming, such as callbacks. `BFTask` is not a replacement for `NSOperation` or GCD. In fact, they play well together. But tasks do fill in some gaps that those technologies don't address.
* `BFTask` takes care of managing dependencies for you. Unlike using `NSOperation` for dependency management, you don't have to declare all dependencies before starting a `BFTask`. For example, imagine you need to save a set of objects and each one may or may not require saving child objects. With an `NSOperation`, you would normally have to create operations for each of the child saves ahead of time. But you don't always know before you start the work whether that's going to be necessary. That can make managing dependencies with `NSOperation` very painful. Even in the best case, you have to create your dependencies before the operations that depend on them, which results in code that appears in a different order than it executes. With `BFTask`, you can decide during your operation's work whether there will be subtasks and return the other task in just those cases.
* `BFTasks` release their dependencies. `NSOperation` strongly retains its dependencies, so if you have a queue of ordered operations and sequence them using dependencies, you have a leak, because every operation gets retained forever. `BFTasks` release their callbacks as soon as they are run, so everything cleans up after itself. This can reduce memory use, and simplify memory management.
* `BFTasks` keep track of the state of finished tasks: It tracks whether there was a returned value, the task was cancelled, or if an error occurred. It also has convenience methods for propagating errors. With `NSOperation`, you have to build all of this stuff yourself.
* `BFTasks` don't depend on any particular threading model. So it's easy to have some tasks perform their work with an operation queue, while others perform work using blocks with GCD. These tasks can depend on each other seamlessly.
* Performing several tasks in a row will not create nested "pyramid" code as you would get when using only callbacks.
* `BFTasks` are fully composable, allowing you to perform branching, parallelism, and complex error handling, without the spaghetti code of having many named callbacks.
* You can arrange task-based code in the order that it executes, rather than having to split your logic across scattered callback functions.

For the examples in this doc, assume there are async versions of some common Parse methods, called `saveAsync:` and `findAsync:` which return a `Task`. In a later section, we'll show how to define these functions yourself.

## The `continueWithBlock` Method

Every `BFTask` has a method named `continueWithBlock:` which takes a continuation block. A continuation is a block that will be executed when the task is complete. You can then inspect the task to check if it was successful and to get its result.

```objective-c
// Objective-C
[[self saveAsync:obj] continueWithBlock:^id(BFTask *task) {
  if (task.isCancelled) {
    // the save was cancelled.
  } else if (task.error) {
    // the save failed.
  } else {
    // the object was saved successfully.
    PFObject *object = task.result;
  }
  return nil;
}];
```

```swift
// Swift
self.saveAsync(obj).continueWithBlock {
  (task: BFTask!) -> BFTask in
  if task.isCancelled() {
    // the save was cancelled.
  } else if task.error != nil {
    // the save failed.
  } else {
    // the object was saved successfully.
    var object = task.result() as PFObject
  }
}
```

BFTasks use Objective-C blocks, so the syntax should be pretty straightforward. Let's look closer at the types involved with an example.

```objective-c
// Objective-C
/**
 * Gets an NSString asynchronously.
 */
- (BFTask *)getStringAsync {
  // Let's suppose getNumberAsync returns a BFTask whose result is an NSNumber.
  return [[self getNumberAsync] continueWithBlock:^id(BFTask *task) {
    // This continuation block takes the NSNumber BFTask as input,
    // and provides an NSString as output.

    NSNumber *number = task.result;
    return [NSString stringWithFormat:@"%@", number];
  )];
}
```

```swift
// Swift
/**
 * Gets an NSString asynchronously.
 */
func getStringAsync() -> BFTask {
  //Let's suppose getNumberAsync returns a BFTask whose result is an NSNumber.
  return self.getNumberAsync().continueWithBlock {
    (task: BFTask!) -> NSString in
    // This continuation block takes the NSNumber BFTask as input,
    // and provides an NSString as output.

    let number = task.result() as NSNumber
    return NSString(format:"%@", number)
  }
}
```

In many cases, you only want to do more work if the previous task was successful, and propagate any errors or cancellations to be dealt with later. To do this, use the `continueWithSuccessBlock:` method instead of `continueWithBlock:`.

```objective-c
// Objective-C
[[self saveAsync:obj] continueWithSuccessBlock:^id(BFTask *task) {
  // the object was saved successfully.
  return nil;
}];
```

```swift
// Swift
self.saveAsync(obj).continueWithSuccessBlock {
  (task: BFTask!) -> AnyObject! in
  // the object was saved successfully.
  return nil
}
```

## Chaining Tasks Together

BFTasks are a little bit magical, in that they let you chain them without nesting. If you return a BFTask from `continueWithBlock:`, then the task returned by `continueWithBlock:` will not be considered finished until the new task returned from the new continuation block. This lets you perform multiple actions without incurring the pyramid code you would get with callbacks. Likewise, you can return a `BFTask` from `continueWithSuccessBlock:`. So, return a `BFTask` to do more asynchronous work.

```objective-c
// Objective-C
PFQuery *query = [PFQuery queryWithClassName:@"Student"];
[query orderByDescending:@"gpa"];
[[[[[self findAsync:query] continueWithSuccessBlock:^id(BFTask *task) {
  NSArray *students = task.result;
  PFObject *valedictorian = [students objectAtIndex:0];
  [valedictorian setObject:@YES forKey:@"valedictorian"];
  return [self saveAsync:valedictorian];
}] continueWithSuccessBlock:^id(BFTask *task) {
  PFObject *valedictorian = task.result;
  return [self findAsync:query];
}] continueWithSuccessBlock:^id(BFTask *task) {
  NSArray *students = task.result;
  PFObject *salutatorian = [students objectAtIndex:1];
  [salutatorian setObject:@YES forKey:@"salutatorian"];
  return [self saveAsync:salutatorian];
}] continueWithSuccessBlock:^id(BFTask *task) {
  // Everything is done!
  return nil;
}];
```

```swift
// Swift
var query = PFQuery(className:"Student")
query.orderByDescending("gpa")
findAsync(query).continueWithSuccessBlock {
  (task: BFTask!) -> BFTask in
  let students = task.result() as NSArray
  var valedictorian = students.objectAtIndex(0) as PFObject
  valedictorian["valedictorian"] = true
  return self.saveAsync(valedictorian)
}.continueWithSuccessBlock {
  (task: BFTask!) -> BFTask in
  var valedictorian = task.result() as PFObject
  return self.findAsync(query)
}.continueWithSuccessBlock {
  (task: BFTask!) -> BFTask in
  let students = task.result() as NSArray
  var salutatorian = students.objectAtIndex(1) as PFObject
  salutatorian["salutatorian"] = true
  return self.saveAsync(salutatorian)
}.continueWithSuccessBlock {
  (task: BFTask!) -> AnyObject! in
  // Everything is done!
  return nil
}
```

## Error Handling

By carefully choosing whether to call `continueWithBlock:` or `continueWithSuccessBlock:`, you can control how errors are propagated in your application. Using `continueWithBlock:` lets you handle errors by transforming them or dealing with them. You can think of failed tasks kind of like throwing an exception. In fact, if you throw an exception inside a continuation, the resulting task will be faulted with that exception.

```objective-c
// Objective-C
PFQuery *query = [PFQuery queryWithClassName:@"Student"];
[query orderByDescending:@"gpa"];
[[[[[self findAsync:query] continueWithSuccessBlock:^id(BFTask *task) {
  NSArray *students = task.result;
  PFObject *valedictorian = [students objectAtIndex:0];
  [valedictorian setObject:@YES forKey:@"valedictorian"];
  // Force this callback to fail.
  return [BFTask taskWithError:[NSError errorWithDomain:@"example.com"
                                                   code:-1
                                               userInfo:nil]];
}] continueWithSuccessBlock:^id(BFTask *task) {
  // Now this continuation will be skipped.
  PFQuery *valedictorian = task.result;
  return [self findAsync:query];
}] continueWithBlock:^id(BFTask *task) {
  if (task.error) {
    // This error handler WILL be called.
    // The error will be the NSError returned above.
    // Let's handle the error by returning a new value.
    // The task will be completed with nil as its value.
    return nil;
  }
  // This will also be skipped.
  NSArray *students = task.result;
  PFObject *salutatorian = [students objectAtIndex:1];
  [salutatorian setObject:@YES forKey:@"salutatorian"];
  return [self saveAsync:salutatorian];
}] continueWithSuccessBlock:^id(BFTask *task) {
  // Everything is done! This gets called.
  // The task's result is nil.
  return nil;
}];
```

```swift
// Swift
var query = PFQuery(className:"Student")
query.orderByDescending("gpa")
findAsync(query).continueWithSuccessBlock {
  (task: BFTask!) -> BFTask in
  let students = task.result() as NSArray
  var valedictorian = students.objectAtIndex(0) as PFObject
  valedictorian["valedictorian"] = true
  //Force this callback to fail.
  return BFTask(error:NSError(domain:"example.com",
                              code:-1, userInfo: nil))
}.continueWithSuccessBlock {
  (task: BFTask!) -> AnyObject! in
  //Now this continuation will be skipped.
  var valedictorian = task.result() as PFObject
  return self.findAsync(query)
}.continueWithBlock {
  (task: BFTask!) -> AnyObject! in
  if task.error != nil {
    // This error handler WILL be called.
    // The error will be the NSError returned above.
    // Let's handle the error by returning a new value.
    // The task will be completed with nil as its value.
    return nil
  }
  // This will also be skipped.
  let students = task.result() as NSArray
  var salutatorian = students.objectAtIndex(1) as PFObject
  salutatorian["salutatorian"] = true
  return self.saveAsync(salutatorian)
}.continueWithSuccessBlock {
  (task: BFTask!) -> AnyObject! in
  // Everything is done! This gets called.
  // The tasks result is nil.
  return nil
}
```

It's often convenient to have a long chain of success callbacks with only one error handler at the end.

## Creating Tasks

When you're getting started, you can just use the tasks returned from methods like `findAsync:` or `saveAsync:`. However, for more advanced scenarios, you may want to make your own tasks. To do that, you create a `BFTaskCompletionSource`. This object will let you create a new `BFTask`, and control whether it gets marked as finished or cancelled. After you create a `BFTaskCompletionSource`, you'll need to call `setResult:`, `setError:`, or `cancel` to trigger its continuations.

```objective-c
// Objective-C
- (BFTask *)successAsync {
  BFTaskCompletionSource *successful = [BFTaskCompletionSource taskCompletionSource];
  [successful setResult:@"The good result."];
  return successful.task;
}

- (BFTask *)failAsync {
  BFTaskCompletionSource *failed = [BFTaskCompletionSource taskCompletionSource];
  [failed setError:[NSError errorWithDomain:@"example.com" code:-1 userInfo:nil]];
  return failed.task;
}
```

```swift
// Swift
func successAsync() -> BFTask {
  var successful = BFTaskCompletionSource()
  successful.setResult("The good result.")
  return successful.task
}

func failAsync() -> BFTask {
  var failed = BFTaskCompletionSource()
  failed.setError(NSError(domain:"example.com", code:-1, userInfo:nil))
  return failed.task
}
```

If you know the result of a task at the time it is created, there are some convenience methods you can use.

```objective-c
// Objective-C
BFTask *successful = [BFTask taskWithResult:@"The good result."];

BFTask *failed = [BFTask taskWithError:anError];
```

```swift
// Swift
let successful = BFTask(result:"The good result")

let failed = BFTask(error:anError)
```

## Creating Async Methods

With these tools, it's easy to make your own asynchronous functions that return tasks. For example, you can make a task-based version of `fetchAsync:` easily.

```objective-c
// Objective-C
- (BFTask *) fetchAsync:(PFObject *)object {
  BFTaskCompletionSource *task = [BFTaskCompletionSource taskCompletionSource];
  [object fetchInBackgroundWithBlock:^(PFObject *object, NSError *error) {
    if (!error) {
      [task setResult:object];
    } else {
      [task setError:error];
    }
  }];
  return task.task;
}
```

```swift
// Swift
func fetchAsync(object: PFObject) -> BFTask {
  var task = BFTaskCompletionSource()
  object.fetchInBackgroundWithBlock {
    (object: PFObject?, error: NSError?) -> Void in
    if error == nil {
      task.setResult(object)
    } else {
      task.setError(error)
    }
  }
  return task.task
}

```

It's similarly easy to create `saveAsync:`, `findAsync:` or `deleteAsync:`.

## Tasks in Series

`BFTasks` are convenient when you want to do a series of tasks in a row, each one waiting for the previous to finish. For example, imagine you want to delete all of the comments on your blog.

```objective-c
// Objective-C
PFQuery *query = [PFQuery queryWithClassName:@"Comments"];
[query whereKey:@"post" equalTo:@123];

[[[self findAsync:query] continueWithBlock:^id(BFTask *task) {
  NSArray *results = task.result;

  // Create a trivial completed task as a base case.
  BFTask *task = [BFTask taskWithResult:nil];
  for (PFObject *result in results) {
    // For each item, extend the task with a function to delete the item.
    task = [task continueWithBlock:^id(BFTask *task) {
      // Return a task that will be marked as completed when the delete is finished.
      return [self deleteAsync:result];
    }];
  }
  return task;
}] continueWithBlock:^id(BFTask *task) {
  // Every comment was deleted.
  return nil;
}];
```

```swift
// Swift
var query = PFQuery(className:"Comments")
query.whereKey("post", equalTo:123)
findAsync(query).continueWithBlock {
  (task: BFTask!) -> BFTask in
  let results = task.result() as NSArray

  // Create a trivial completed task as a base case.
  let task = BFTask(result:nil)
  for result : PFObject in results {
    // For each item, extend the task with a function to delete the item.
    task = task.continueWithBlock {
      (task: BFTask!) -> BFTask in
      return self.deleteAsync(result)
    }
  }
  return task
}.continueWithBlock {
  (task: BFTask!) -> AnyObject! in
  // Every comment was deleted.
  return nil
}
```

## Tasks in Parallel

You can also perform several tasks in parallel, using the `taskForCompletionOfAllTasks:` method. You can start multiple operations at once, and use `taskForCompletionOfAllTasks:` to create a new task that will be marked as completed when all of its input tasks are completed. The new task will be successful only if all of the passed-in tasks succeed. Performing operations in parallel will be faster than doing them serially, but may consume more system resources and bandwidth.

```objective-c
// Objective-C
PFQuery *query = [PFQuery queryWithClassName:@"Comments"];
[query whereKey:@"post" equalTo:@123];

[[[self findAsync:query] continueWithBlock:^id(BFTask *results) {
  // Collect one task for each delete into an array.
  NSMutableArray *tasks = [NSMutableArray array];
  for (PFObject *result in results) {
    // Start this delete immediately and add its task to the list.
    [tasks addObject:[self deleteAsync:result]];
  }
  // Return a new task that will be marked as completed when all of the deletes are
  // finished.
  return [BFTask taskForCompletionOfAllTasks:tasks];
}] continueWithBlock:^id(BFTask *task) {
  // Every comment was deleted.
  return nil;
}];
```

```swift
// Swift
var query = PFQuery(className:"Comments")
query.whereKey("post", equalTo:123)

findAsync(query).continueWithBlock {
  (task: BFTask!) -> BFTask in
  // Collect one task for each delete into an array.
  var tasks = NSMutableArray.array()
  var results = task.result() as NSArray
  for result : PFObject! in results {
    // Start this delete immediately and add its task to the list.
    tasks.addObject(self.deleteAsync(result))
  }
  // Return a new task that will be marked as completed when all of the deletes
  // are finished.
  return BFTask(forCompletionOfAllTasks:tasks)
}.continueWithBlock {
  (task: BFTask!) -> AnyObject! in
  // Every comment was deleted.
  return nil
}
```

## Task Executors

Both `continueWithBlock:` and `continueWithSuccessBlock:` methods have another form that takes an instance of `BFExecutor`. These are `continueWithExecutor:withBlock:` and `continueWithExecutor:withSuccessBlock:`. These methods allow you to control how the continuation is executed. The default executor will dispatch to GCD, but you can provide your own executor to schedule work onto a different thread. For example, if you want to continue with work on the UI thread:

```objective-c
// Create a BFExecutor that uses the main thread.
BFExecutor *myExecutor = [BFExecutor executorWithBlock:^void(void(^block)()) {
  dispatch_async(dispatch_get_main_queue(), block);
}];

// And use the Main Thread Executor like this. The executor applies only to the new
// continuation being passed into continueWithBlock.
[[self fetchAsync:object] continueWithExecutor:myExecutor withBlock:^id(BFTask *task) {
    myTextView.text = [object objectForKey:@"name"];
}];
```

For common cases, such as dispatching on the main thread, we have provided default implementations of `BFExecutor`. These include `defaultExecutor`, `immediateExecutor`, `mainThreadExecutor`, `executorWithDispatchQueue:`, and `executorWithOperationQueue:`. For example:

```objective-c
// Continue on the Main Thread, using a built-in executor.
[[self fetchAsync:object] continueWithExecutor:[BFExecutor mainThreadExecutor] withBlock:^id(BFTask *task) {
    myTextView.text = [object objectForKey:@"name"];
}];
```

## Task Cancellation

It's generally bad design to keep track of the `BFTaskCompletionSource` for cancellation. A better model is to create a "cancellation token" at the top level, and pass that to each async function that you want to be part of the same "cancelable operation". Then, in your continuation blocks, you can check whether the cancellation token has been cancelled and bail out early by returning a `[BFTask cancelledTask]`. For example:

```objective-c
- (void)doSomethingComplicatedAsync:(MYCancellationToken *)cancellationToken {
    [[self doSomethingAsync:cancellationToken] continueWithBlock:^{
        if (cancellationToken.isCancelled) {
            return [BFTask cancelledTask];
        }
        // Do something that takes a while.
        return result;
    }];
}

// Somewhere else.
MYCancellationToken *cancellationToken = [[MYCancellationToken alloc] init];
[obj doSomethingComplicatedAsync:cancellationToken];

// When you get bored...
[cancellationToken cancel];
```

**Note:** The cancellation token implementation should be thread-safe.
We are likely to add some concept like this to Bolts at some point in the future.

# Installation

You can download the latest framework files from our [Releases page](https://github.com/BoltsFramework/Bolts-ObjC/releases).

Bolts is also available through [CocoaPods](https://cocoapods.org/). To install it simply add the following line to your Podfile:

    pod 'Bolts'
