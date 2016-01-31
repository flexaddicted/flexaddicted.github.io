---
layout: post
title:  "Having fun with NSOperations in iOS"
date:   2016-01-05 00:00:00 +0000
---

Keeping your app responsive could be a very challenging task. If you are performing long running operations in the main thread you could end up with a sluggish and unresponsive user interface. In iOS a powerful way to move work off the main thread can be achieved through [`NSOperation`](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/NSOperation_class/) and [`NSOperationQueue`](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/NSOperationQueue_class/) classes. I used them a lot during my previous development and I will continue to use them.

So, what is an `NSOperation`? A good definition can be found in [NSHipster](http://nshipster.com/nsoperation/):

>NSOperation represents a single unit of work. It’s an abstract class that offers a useful, thread-safe structure for modeling state, priority, dependencies, and management.


And, why use `NSOperation`s? Because you are able to abstract in a very good way a self-contained task your application needs to accomplish.

An `NSOperation` can be described through a finite state machine. Possible state are: pending, ready, executing, finished and cancelled. The relationships among these states can be represented with the following diagram. As you can notice, the finished state is a final one.

![image](/assets/2016-01-05-state-machine.png)

There are many ways to create `NSOperation`s: subclassing an `NSOperation`, using `NSInvocationOperation` or creating `NSBlockOperation`. I prefer the subclass way. Why? Because you can create an instance of it and reuse it across all the application.

An `NSOperation` can run in two different flavors: non-asynchronous (or non-concurrent) and asynchronous (or concurrent).
Before explaining the difference between the two, I would highlight that starting from iOS 8 the property `concurrent` has been replaced with the more clear `asynchronous` one. In this manner it is possible to determinate if an `NSOperation` executes synchronously in the main method (see below) or asynchronously.

So, what is the difference between a non-asynchronous and an asynchronous operation?

In order to kick-off a non-asynchronous operation, you need to add it to an `NSOperationQueue` (just a data structure that contains all your operations). If you decided to subclass, your subclass must override the `main` method. This method will run your code in a background thread for you. Since `NSOperation` and `NSOperationQueue` are run on top on GCD (Grand Central Dispatch), you do not need to create your own threads. An operation is then removed from its current queue if the main method has finished or the operation has been cancelled.
As a good rule (within the `main` method) of thumb it's important to check the `isCancelled` property in order to verify if the operation has been cancelled or not. In this case, you should be responsible to rollback the state of its execution in a valid state.

A simple non-asynchronous operation subclass can look like the following.

	class NonConcurrentOperation : NSOperation {

	    override func main() {

	        if(cancelled) {
	            return;
	        }

            // Execute your long task here. The task will execute in background task for you
            // As a suggestion, check the cancelled property
            // in order to rollback an invalid state introduced by the operation

	    }

	}

Such type of operation can be used like the following.

	let nonConcurrentOperation = NonConcurrentOperation()
	let queue = NSOperationQueue()
	queue.addOperation(nonConcurrentOperation)

Using an asynchronous operation, instead, requires more work since in order to kick-off it, you
need to override the `start` method (without calling the superclass implementation) and manage the state of the operation yourself.
What does it mean? `NSOperation` class is KVC (Key Value Coding) and KVO (Key Value Observing) compliant for several of its properties. Monitoring the ongoing state of your task and reporting changes in that state using KVO notifications allow you to manage in the correct way the state of the operation.

A simple concurrent `NSOperation` can look like the following.

	class ConcurrentOperation: NSOperation {

	    override var asynchronous: Bool {
	        return true
	    }

	    private var _executing = false {
	        willSet {
	            willChangeValueForKey("isExecuting")
	        }
	        didSet {
	            didChangeValueForKey("isExecuting")
	        }
	    }

	    override var executing: Bool {
	        return _executing
	    }

	    private var _finished = false {
	        willSet {
	            willChangeValueForKey("isFinished")
	        }

	        didSet {
	            didChangeValueForKey("isFinished")
	        }
	    }

	    override var finished: Bool {
	        return _finished
	    }

	    override func start() {
	        _executing = true
	        execute()
	    }

	    func execute() {
	        // Execute your async task here.
	    }

	    func finish() {
	        // Notify the completion of async task and hence the completion of the operation

	        _executing = false
	        _finished = true
	    }

	}

In order to use it, you can still use an `NSOperationQueue` (in order to leverage all the provided goodness of it).

	let concurrentOperation = ConcurrentOperation()
	let queue = NSOperationQueue()
	queue.addOperation(concurrentOperation)

Apple documentation encourages the usage of non-asynchronous operations in combination with an `NSOperationQueue`. This way is simpler than the asynchronous one since you don't need to do the dirty work yourself.
A possible problem that can arise is that if you override the `main` method and you start an async task within it, that task will not be completed. As soon the `main` method reaches the end, the operation is removed from its queue. So, for example, if you want to take advantage of `NSURLSession` API within an operation, you need to rely on an asynchronous operation.

So, for example, if you want to create a download operation you can use the approach below.

	class DownloadOperation : ConcurrentOperation {

	    var error: NSError?
	    var task: NSURLSessionDownloadTask?
	    var downloadCompleted : ((Bool, NSError?) -> Void) = { _ in }

	    lazy var session: NSURLSession = {
	        return NSURLSession.sharedSession()
	    }()

	    override func execute() {
	        task = session.downloadTaskWithURL(NSURL(string: "yourURL")!) {
	            (url, response, error) in

	            if error == nil {

	                // Notify the response by means of a closure or what you prefer
	                // Remember to run in the main thread since NSURLSession runs its
	                // task on background by default
	                self.downloadCompleted(true, nil)

	            } else {

	                // Notify the failure by means of a closure or what you prefer
	                // Remember to run in the main thread since NSURLSession runs its
	                // task on background by default
	                self.downloadCompleted(false, error!)

	            }

	            // Remember to tell the operation queue that the execution has completed
	            self.finish()
	        }
	        task!.resume()
	    }
	}

`NSOperationQueue` provides a nice way to control the number of concurrent operations that can be run "in parallel". In particular, the developer can set the `maxConcurrentOperationCount` to 1 in order to create a serial queue. At this point each operation will be executed in a FIFO (First In First Out) order.
In addition to that, you can also control the priority of each operation by using new Quality of Service APIs provided by iOS 8. This a pretty new concept that allows you to distinguish four different service levels: user-interactive, user-initiated, utility and background. If you are interested in more details, I suggest you watch [Power, Performance and Diagnostics: What's new in GCD and XPC](https://developer.apple.com/videos/play/wwdc2014-716/) talk at WWDC 2014.

The concept I like the most when using operations and queues is the one of creating dependencies. They allow you to execute operations in a specific order. The way to achieve this is done through the `addDependency` method. For example, if we want that a parsing operation should complete only after the download operation has finished its execution, our code should look like the following.

	let downloadOperation = // ...
	let parsingOperation = // ...
	[parsingOperation addDependency:downloadOperation];

But wait. There is also more. If you need to notify the user interface, for example, that both the operations have completed, you can create a third operation (in this case through `NSBlockOperation`) and say that this operation has a dependency with both the previous ones.

	let completionOperation = NSBlockOperation {             
	    print("All previous operations are finished")    
	}
	[completionOperation addDependency:downloadOperation];
	[completionOperation addDependency:parsingOperation];

That's all for me! Anyway, if you want to understand more of `NSOperation`s I strongly suggest to watch the [Advanced NSOperations](https://developer.apple.com/videos/play/wwdc2015-226/) talk at WWDC 2015.
