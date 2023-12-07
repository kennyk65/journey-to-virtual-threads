# journey-to-virtual-threads
A historical journey of how Java finally solved the blocking problem.  Java 21's Virtual Threads solve a primary issue faced by most Java developers.

It's taken nearly 30 years.  Java 1.21's introduction of Virtual Threads will finally make multitasking in Java almost effortless.  In order to fully appreciate their revolutionary nature, it is helpful to take a look at the various imperfect solutions offered by Java over the years to solve the "do useful work while we wait for something else" problem.

 If you'd like to see the coding examples presented in this article, see https://github.com/kennyk65/journey-to-virtual-threads

Java 1 
The Introduction of Java version 1 in 1995 was remarkable.  A strongly-typed, object-oriented, C-like-syntax language which offered many features, including easy-to-use Threads.  The Thread class represented an object that would run selected code in a separate thread from the main execution thread.  The Thread object itself was a wrapper for an actual OS-level thread known as a platform thread, a.k.a. kernel thread.  The logic to be executed was described by implementing a Runnable interface.  Java took care of all of the complexity of launching and managing this separate thread.  Now it will be almost trivial to perform multiple tasks simultaneously - or so it would seem.  Consider the following example:

Java
 
package example.java1;

public class Simple {

    public void threadExample() {
        Runnable r = new Runnable (){
            public void run() {
                System.out.println("do something in background");;
            }
        };
        Thread t = new Thread(r);
        t.start();
    }
}


The main thread describes an implementation of a Runnable interface, which is doing trivial work in this case.  A second thread - let's call it the background thread - is instantiated using this Runnable.  The start() method commands the background thread to work while the main thread resumes whatever work it needs to do. 

Excellent!  But an immediate challenge faced by first generation Java developers was how to pass a value from the background thread back to the main thread.  Consider this example where we wish to execute a long-running, blocking process in the background, then later use its returned value in the main thread:

Java
 
package example.java1;

import java.util.concurrent.TimeUnit;

public class PassingValues {

    // Variable shared between threads:
    private static String result;

    // Utility method to simulate a delay:
    private static void delay(int i) {
        try { TimeUnit.SECONDS.sleep(i); } 
        catch (InterruptedException e) {}
    }

    // Create an instance of a Runnable that updates the shared variable:
    private static Runnable workToDo = new Runnable (){
        public void run() {
            String s = longRunningProcess();  
            synchronized (this) {
                result = s;     //  Blocking!  Safely update shared variable.
            }
        }

        //  Simulate a long running, blocking process:
        private String longRunningProcess() {
            delay(2);               // blocking!
            return "Hello World!";         
        }
    };

    public static void main(String[] args) {
        // Create a Thread, pass the Runnable, and start it:
        Thread thread = new Thread(workToDo);
        thread.start();

	    // Do other work...

        // Wait for the thread to complete:
        try {
            thread.join();  //  blocking!
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // Use results
        String s = null;
        synchronized (workToDo) { 
            s = result;     //  Blocking! Safely acquire the shared variable
        } 
        System.out.println("Result from the thread: " + s);
    }
}


The Runnable implementation describes a long-running process, here simulated by calling sleep().  The main thread spawns a background thread and launches it to perform work.  The main thread is (hopefully) able to perform useful work while the background thread is busy doing separate work - fantastic!  However, at a certain point, the main thread must obtain the result from the background thread before it can continue, and may have no safe choice but to wait until the result is ready.

Notice the "// blocking!" comments in the code.  The background thread is blocked while waiting for a result.  Real-world examples of this blocking occur when calling a database, external web service, etc.  While blocked, all of the resources associated with a thread are effectively stuck waiting, unable to do any useful work.  The main thread is blocked while waiting for the background thread, unable to do any useful work.  Synchronization blocks are used to safely access variables used by multiple threads - a bit excessive in this case, but useful to illustrate other blocking points.  The platform threads seen here are relatively lightweight when compared to spawning separate OS-level processes, but large numbers of them can consume a lot of JVM resources. The threads above spend most of their time waiting for something to do.

In this example, the enemy is blocking, simulated by the sleep() method.  The imperfect solution offered by multithreading doesn't solve the problem, it merely parks the blocking in a background thread.  Eventually threads will block each other when coordinating activities, seen here in the join() and synchronization blocks.  Blocking is a terrible waste of resources whenever it happens.  For years Java treated blocking as an unsolvable obstacle, using the imperfect, resource-heavy solution of multithreading to address it.

Against this backdrop, one puzzle faced by Java developers is how a single-threaded scripting language like JavaScript can outperform multi-threaded, compiled Java code at certain tasks while using fewer resources.  Consider this year-2000 JavaScript code which achieves the same result as the previous example using a fraction of its memory:

JavaScript
 
var result;

// Function to simulate a long-running, blocking process 
// with an inline callback
function longRunningProcess(callback) {
    setTimeout(() => {
        callback("Hello World!");
    }, 2000); // blocking!
}

// Call the function with an inline callback
longRunningProcess((data) => {
    result = data;

    // Do other work...

    // Use results
    console.log("Result from the inline callback:", result);
});


JavaScript uses an event loop within a single thread.  Callbacks are function references used to identify work to be done when a task is completed.  Rather than tie up the one-and-only thread waiting for work to complete, JavaScript places the callback reference on its internal message queue and continues working.  The callback will be executed asynchronously by the event loop at a later time.  There is no blocking, or to be more precise, there is no blocking penalty; any waiting for results does not impact the execution thread.

Notice the "longRunningProcess()" in both examples.  The signatures are different; Java expects a value returned when the work is done, JavaScript expects a callback function to be invoked when the work is done.  Java demonstrates synchronous execution, JavaScript demonstrates asynchronous.  Both involve waiting for some work to be done, but the synchronous occupies a thread to do this.

To be fair, multiple Java threads will outperform JavaScript if 1) the threads are busy most of the time doing useful in-memory work 2) coordination between threads is absent or minimal. These are not the case in many real-world situations.

One more consideration - the code shown here creates a Thread and allows it to be disposed of through garbage collection, wasteful of time and memory. Java 1 didn’t provide any Thread pooling capability, so early Java developers created their own with management logic to acquire and release threads from the pool.  

To summarize threading in Java 1:

Multithreading doesn't solve the blocking problem, and blocking is often the main problem.
Multithreading is resource intensive vs single-threaded, asynchronous models (i.e. JavaScript)
Threads are low-level constructs requiring code to manage and coordinate.
Later releases of Java would address all of these concerns, though it would take a while to get there.  Java 1.2 (1998) and 1.3 (2000) made only minor contributions to threading, the next batch of improvements would need to wait until the 2004 release of Java 5 (1.5).

Java 5
Java 1.5 (2004) introduced several features to improve thread management and coordination of work between threads, we'll focus on Future, ExecutorService, and Callable from the java.util.concurrent package.

ExecutorService abstracts away the mechanism by which something is executed.  Implementations exist for single thread, thread pools of fixed size, thread pools that grow and shrink, etc.  A Callable is like a Runnable except it can return a value.  To hold the value returned from a Callable, we need a special object which can live in one thread but hold the return value populated by the Callable.  This is the Future object, which represents a promise from one thread to populate a value for another thread.

Java
 
package example.java5;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

public class Demo {
    
    // Utility method to simulate a delay:
    private static void delay(int i) {
        try { TimeUnit.SECONDS.sleep(i); } 
        catch (InterruptedException e) {}
    }

    // Create a Callable that returns a result variable
    private static Callable<String> workToDo = new Callable<>() {
        public String call() throws Exception {
            return longRunningProcess();
        };

        // Simulate a long running, blocking process
        private String longRunningProcess() {
            delay(2);               // blocking!
            return "Hello World!";
        }
    };


    public static void main(String[] args) {
        // Create a singleThread ExecutorService, and submit the work:
        Future<String> future = 
            Executors
                .newSingleThreadExecutor()
                    .submit(workToDo);

        // Do other work...

        // Wait for the thread to complete:
        String result = "";
        try {
            result = future.get();  // blocking!
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        // Use results
        System.out.println("Result from the thread: " + result);
    }
}


Notice the differences from Java 1: 

The Callable is better than Runnable when the background thread needs to return a value to the main thread.  No more shared variable or synchronization between the two threads.
The Executors class contains factory methods for creating an ExecutorService.  We don't need to build our own thread pools.
The Future represents a promised result.  The get() method returns the result returned from the other thread, blocking the main thread if the result is not yet available.  We can even use it to cancel the other thread.
Notice what has not changed: the enemy here is still blocking.  The lines commented with "// blocking!" indicates the main points where it can occur, the long-running process simulated by the sleep() method, and the future.get() method, which may result in blocking if the background thread is not finished.  While Java 5 provides more syntactic elegance and ease-of-use, we are still using threads as an imperfect solution.

In fact, Java 5 may have made the situation worse!  By making multithreading easier to use, more developers explore it as the answer to various performance problems.  It is common sense to think that if we break up a single task into many parts worked on in parallel, the overall task will go faster.  Unless the various threads are busy a large percentage of the time and have little need to coordinate, Java programs generally consume more memory resources than other models and can even demonstrate worse performance due to the overhead of context switching.

Additionally, a need was growing to chain multiple pieces of work together, executing each part in the background.  Consider the following less-than-elegant solution:

Java
 
package example.java5;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

public class ChainingAsync {

    public static void main(String[] args) {

        try {
            ExecutorService svc = Executors.newSingleThreadExecutor();

            Future<String> future1 = svc.submit(new WorkToDo1());
            // Do other work...
            String result1 = future1.get();  // blocking!
            Future<String> future2 = svc.submit(new WorkToDo2(result1));
            // Do other work...
            String result2 = future2.get();  // blocking!
            Future<String> future3 = svc.submit(new WorkToDo3(result2));
            // Do other work...
            String result3 = future3.get();  // blocking!

            System.out.println("Result from the thread: " + result3);

        } catch (Exception e) {}
    }

    private static class WorkToDo1 implements Callable<String> {
        public String call() throws Exception {
            delay(1);       // blocking!
            return "Hello";
        }
    }

    private static class WorkToDo2 implements Callable<String> {
        private final String prefix;
        public WorkToDo2(String prefix) {
            this.prefix = prefix;
        }
        public String call() throws Exception {
            delay(1);       // blocking!
            return prefix + " World";
        }
    }

    private static class WorkToDo3 implements Callable<String> {
        private final String prefix;
        public WorkToDo3(String prefix) {
            this.prefix = prefix;
        }
        public String call() throws Exception {
            delay(1);       // blocking!
            return prefix + "!!";
        }
    }

    // Utility method to simulate a delay:
    private static void delay(int i) {
        try { TimeUnit.SECONDS.sleep(i); } 
        catch (InterruptedException e) {}
    }
   
}


This is getting messy.  Chaining work requires custom callables to receive and return required values.  This code attempts to multitask as much as possible.  Its effectiveness depends on how much "other work" is performed in the main thread.  If it is insignificant, we are consuming more resources than are needed to simply call three functions in order:

Java
 
package example.java5;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

public class ChainingSync {

    public static void main(String[] args) {
        String result1 = doThing1();
        String result2 = doThing2(result1);
        String result3 = doThing3(result2);
        System.out.println("Result from a single thread: " + result3);
    }

    private static String doThing1() {
        delay(1);
        return "Hello";
    }

    private static String doThing2(String input) {
        delay(1);
        return input + " World";
    }

    private static String doThing3(String input) {
        delay(1);
        return input + "!!";
    }

    // Utility method to simulate a delay:
    private static void delay(int i) {
        try { TimeUnit.SECONDS.sleep(i); } 
        catch (InterruptedException e) {}
    }

}


Even programmers with very little experience could follow the second example, yet it runs with fewer resources than the first. Now consider a JavaScript equivalent, but with no blocking:

JavaScript
 
(function() {
    setTimeout(function doThing1() {
        const result1 = "Hello";

        setTimeout(function doThing2() {
            const result2 = result1 + " World";

            setTimeout(function doThing3() {
                const result3 = result2 + "!!";

                console.log("Result from JavaScript:", result3);
            }, 1000);
        }, 1000);
    }, 1000);
})();


(There is a reason you chose to work in Java rather than JavaScript.)  The horrifying JavaScript syntax you see here is referred to as "callback hell".  On the bright side, there is no blocking, only a single thread is used, and it consumes only a fraction of the resources of the previous Java examples. 

 Java would need a better way to chain Futures (promises) from one to another without callback hell, and of course we still need to do something about the blocking.  (To be fair, modern JavaScript uses Promises and async/await to more clearly implement the logic you see here, but this article is a history lesson.) 

Java 7 
The next two releases of Java, 1.6 (2006) and 1.7 (2011) addressed neither concern.  Java 1.7 did introduce a novel improvement known as the ForkJoinPool.  This new feature provided a way to divide a set of work into chunks for parallel processing.  The ForkJoinPool together with RecursiveTask made it easy to drop off a large amount of work to be performed, say in a Collection, and have it broken down recursively into smaller and smaller chunks for parallel processing by different threads (forked), then later collect the results of all the finished work (joining).  

While interesting in its own right, a byproduct of this development was the introduction of a work-stealing algorithm, where idle threads in the ForkJoinPool can "steal" tasks from other busy threads. The ForkJoinPool is a good thread pool option for general use, but it doesn't help us with our blocking issue, since blocked threads are not considered "idle" in this context.  Interestingly, the work-stealing concept foreshadowed the concept that a blocked thread might be released to do other work, which is the essence of Java 21's Virtual Thread solution.

Java 8
Java 1.8 (2014) was a BIG release.  On the multitasking front, the star addition was the CompleteableFuture, a big improvement over the Future from Java 1.5.  To quickly and easily run work in a separate thread, the CompleteableFuture class provides a number of "*Async" methods which execute the desired code in another thread provided by a ForkJoinPool (by default).  The caller simply calls the method directly without fumbling with Threads or an ExecutorService.  Callback methods can register the actions to take place when the asynchronous work is done.  Finally, Lambda syntax reduces the heavy boilerplate seen in our Java 5 example:

Java
 
package example.java8;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;

public class Demo {

    // Utility method to simulate a delay
    private static void delay(int i) {
        try { TimeUnit.SECONDS.sleep(i); } 
        catch (InterruptedException e) {}
    }

    // Simulate a long running, blocking process
    private static CompletableFuture<String> longRunningProcess() {
        return CompletableFuture.supplyAsync(() -> {
            delay(2);   // blocking!
            return "Hello World!";
        });
    }

    public static void main(String[] args) {
        // Create a CompletableFuture
        longRunningProcess().thenAccept(result -> {
            System.out.println("Result from CompletableFuture: " + result);
        });

        // Do other work...

        // Introduce a delay to prevent the JVM 
        // from exiting before work is complete.
        delay(3);
    }
}


Notice:

The calling thread no longer has to fumble with Executors or ExecutorService.
The main thread does not block waiting for completion of the asynchronous execution.  Instead, work to be done is registered in a callback.
The final delay() is not needed in a real-world application.  It is used here only to prevent the main non-daemon thread from exiting before the callback in the daemon thread can be demonstrated.  
The callback shown above allows for the possibility to chain several actions together.  CompleteableFuture supports a fluent interface to allow composition of asynchronous workflows; a pipeline of asynchronous operations where each executes after the previous one completes.  Many methods are provided which accept Suppliers, Consumers, Functions, and other interfaces which can be replaced with Lambda expressions.  

Java
 
package example.java8;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class ChainDemo {

    // Utility method to simulate a delay
    private static void delay(int i) {
        try { TimeUnit.SECONDS.sleep(i); } 
        catch (InterruptedException e) {}
    }

    private static String doThing1() {
        delay(1);
        return "Hello";
    }

    private static String doThing2(String s) {
        delay(1);
        return s + " World";
    }

    private static String doThing3(String s) {
        delay(1);
        return s + "!!";
    }


    public static void main(String[] args) {

        CompletableFuture
            .supplyAsync(() -> doThing1())
            .thenApplyAsync(s -> doThing2(s))
            .thenApplyAsync(s -> doThing3(s))
            .thenAcceptAsync(System.out::println);

        System.out.println("Main thread continues, unhindered.");

        // Introduce a delay to prevent the JVM 
        // from exiting before work is complete.
        delay(4);
    }
}


Far simpler and cleaner than the Java 5 equivalent!  Each fluent method you see here returns a CompleteableFuture.  The "Async" methods run asynchronously in a thread provided by a ForkJoinPool.  This example establishes a chain of callbacks, each executed in a background thread, each executed when the previous callback completes.  In this case, the "Main thread..." message actually displays before the message produced by the CompleteableFuture since the context switching between threads in the chain consumes time.

CompleteableFuture is a great pragmatic improvement over earlier models; much easier to use, and addresses the practical concerns of most developers - executing long-running calls in the background - while adding the ability to assemble chains of actions.  However, CompletableFuture does nothing special when the background thread blocks, or in cases where we must wait for the ultimate result before returning to a caller.  

The chaining of multiple actions to be completed in other threads is also making our debugging, testing, and exception handling work more difficult.  Exceptions occurring within our own code - the doThing*() methods - will not pose a challenge.  However, if an exception is thrown within the asynchronous framework, say because one of our methods returned a null, the resulting NullPointerException would be very tough to track down.  The resulting stack trace might not contain any references to our code at all.

CompleteableFuture does raise an interesting possibility: what if every method in our entire application allows for execution in another thread, including calls to databases, web services, etc.?  If every method returned a CompleteableFuture, it might be possible to construct an entire application's logic as a chain of steps to be completed by any thread available in a pool.  If it were possible to replace every blocking call with a reference to a callback, it might be possible to free threads from blocking, and gain a tremendous amount of productive work out of one or two threads.

The stage was set for reactive programming...

Java 9
Reactive programming builds upon the paradigm seen with chaining and composing CompleteableFutures, but its origins are elsewhere.  Before the java.util.concurrent.Flow types were introduced in Java 9 (2017), Java developers were already using RxJava and Spring's Reactor.  The latter used the Reactive Streams library which was later incorporated into Java 9.  

Reactive programming is not about multithreading at all.  It is about reducing a workload to a stream of events that can be processed asynchronously.  In reactive programming, everything can be represented as a stream.  Streams contain events flowing in sequence.  For example, the results from a database call can be viewed as a stream of events, the rows that match our query.  The results of a web service call can be viewed as a stream, even if it is a stream of one thing.  The term "stream" here should not be confused with java.util.stream concepts from Java 8; while there are syntactical similarities, the reactive streams are inherently asynchronous, while Java 8 streams provide an alternative to complex loops with variables to hold state.

Reactive is tough to learn, so take your time.  I recommend starting with Dave Syer's excellent three part exploration and Andre Saltz's fantastic introduction.  Java 9's java.util.concurrent.Flow classes are low-level definitions of reactive constructs (Publisher and Subscriber) and require a lot of code to construct a simple stream.  For our journey, let's consider an RxJava alternative to the CompleteableFuture example shown earlier:

Java
 
package example.java9;

import java.util.concurrent.TimeUnit;

import io.reactivex.rxjava3.core.Single;

public class RxJavaDemo {

    // Utility method to simulate a delay
    private static void delay(int i) {
        try { TimeUnit.SECONDS.sleep(i); } 
        catch (InterruptedException e) {}
    }

    private static String doThing1() {
        delay(1);
        return "Hello";
    }

    private static String doThing2(String input) {
        delay(1);
        return input + " World";
    }

    private static String doThing3(String input) {
        delay(1);
        return input + "!!";
    }
    
    public static void main(String[] args) {
        Single.fromCallable(() -> doThing1())
                .map(RxJavaDemo::doThing2)
                .map(RxJavaDemo::doThing3)
                .doAfterSuccess(System.out::println)
                .subscribe(); // Subscribe to start the reactive pipeline
    }

}


RxJava has two types of streams, Single and Observable.  Single is designed for streams expecting a single event, so it is appropriate to use here.  Our stream starts with one event containing the "Hello" String returned from the doThing1() method, but it is a stream.  The map() function is an operator.  It takes the event from one stream, processes it, and sends it out in a new, separate stream.  In this case, the doThing2() method merely concatenates " World" to the end of the incoming String.  The next map() operator takes in the event, calls doThing3(), which concatenates "!!" to the incoming String, and produces yet another output stream.  The doAfterSuccess() function operates on the only event it encounters in the Single stream by printing to the console.  The subscribe() call is critical - the earlier code merely defines the algorithm, the subscribe() call effectively commands it to begin running.

Here is another implementation, based on Spring's Reactor.  Note the similarities:

Java
 
package example.java9;

import java.util.concurrent.TimeUnit;

import reactor.core.publisher.Mono;

public class ReactorDemo {

    // Utility method to simulate a delay
    private static void delay(int i) {
        try { TimeUnit.SECONDS.sleep(i); } 
        catch (InterruptedException e) {}
    }

    private static String doThing1() {
        delay(1);
        return "Hello";
    }

    private static String doThing2(String input) {
        delay(1);
        return input + " World";
    }

    private static String doThing3(String input) {
        delay(1);
        return input + "!!";
    }

    public static void main(String[] args) {
        Mono.fromCallable(() -> doThing1())
            .map(ReactorDemo::doThing2)
            .map(ReactorDemo::doThing3)
            .doOnNext(System.out::println)
            .subscribe(); 
    }

}


Reactor uses Mono and Flux instead of RxJava's Single and Observable.  Our Mono begins with one event containing the "Hello" String, just as the RxJava version.  The map() and subscribe() methods are equivalent to their RxJava counterparts, doOnNext() is roughly equivalent to doAfterSuccess().  

In both examples, the reactive streams are executed directly in the main thread when subscribe() is called.  Once the streams are exhausted, program execution continues.  There is no need for the delay() call seen in earlier examples because all work is done in the main thread.

Let me repeat that: in the examples shown here, all work is done in the main thread.  The essential difference from the CompleteableFuture chain constructed earlier, other than syntax, is that the reactive approach operates by breaking up the work into chunks (events) that will be processed asynchronously.  Asynchronous doesn't mean "at the same time" or "in the background", it means "NOT simultaneous or concurrent in time" (Merriam-Webster, emphasis mine).  The model of execution used is much closer to JavaScript's event loop than the  multithreading approach seen in earlier Java versions.  While we could ask either implementation to pool multiple threads to break up the event processing, it's not necessary in these simple examples.  Remember: the enemy in these examples is blocking, represented by the delay() / sleep() calls.  Earlier examples tried to address blocking by multithreading, reactive tries to address it by asynchronous event handling.

But do these reactive examples solve the blocking problem?  If the calls to doThing1(), 2, and 3 make blocking calls, they will still block the entire thread - the main thread in these examples.  The reactive approaches shown here each consume one thread instead of two so they are more frugal on machine resources.  We could use Schedulers to pool up separate threads to process the asynchronous events as they come in, but ultimately in the case shown here those separate threads would be blocked most of the time.  

Of course, this is a contrived example using sleep() to simulate an actual blocking call to a database or web service, all called from a public static void main() method.  An optimized example would utilize reactive HTTP clients (like WebClient) or database libraries (like R2DBC).  These return reactive types (Flux/Mono) which can flow into upstream objects which return reactive types themselves.  If we can construct a perfect reactive call stack to a modern HTTP server with non-blocking IO handlers, we can eliminate all blocking calls.  If this article were primarily about reactive programming, I'd provide such an example, and suffice it to say it would eliminate all blocking with bare minimum use of threads.

The essential difficulty for most developers is thinking in the reactive programming style.  It requires a departure from easy-to-understand, step-by-step imperative programming.  As with chains of CompleteableFutures, reactive programs make our debugging, testing, and exception handling work more difficult. To get the benefit of reactive programming, an application's entire call stack must deal in reactive types; if we introduce a single blocking call, we've eliminated the essential benefit of the model.  In such a case, we may achieve a result no better than CompleteableFuture, and give ourselves a migraine in the process.

But we've solved the blocking problem! ...At a cost measured in aspirin. To summarize where we are with reactive programming:

We must learn to think in the reactive programming style rather than imperative programming.
An application's entire call stack must deal in reactive types.  A single blocking call threatens the entire model.
Debugging is tough, exception handling is perplexing.
Let's return to what we really want: the ability to make long-running, possibly blocking calls from our code without incurring the penalty of holding up a thread.  Ideally, we'd like to tell the JVM it is ok for a thread to work on something else while we wait for a response.  We would like to do this in imperative, step-by-step instructions without struggling with an asynchronous event model, and definitely without the callback syntax we saw in JavaScript.

The stage is now set for Java 21 Virtual Threads...

Java 21 Virtual Threads
After version 9, Java switched to a six-month release cadence, so the next few years went by fast.  In terms of multithreading or reactive programming, the next major advance would occur with Java 21's (2023) introduction of Virtual Threads: 

Virtual Threads are a lightweight alternative to traditional Java platform threads backed by OS-level threads. Compared to platform threads they are wonderfully lightweight, it is trivial to launch thousands of them while your application is running.  Pooling is completely unnecessary. Large numbers of virtual threads can run within each platform thread.  All work is still ultimately done on platform threads but a special Continuation object is used to exchange which Virtual Thread is running based on when a blocking call is encountered.

Compared with earlier models based on CompleteableFuture or Reactive Streams, coding with Virtual Threads is amazingly simple.  Let's go back to some code we haven't seen since the Java 1 and 5 days when we would define all our work within simple Runnables or Callables:

Java
 
package example.java21;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class Demo {

    // Utility method to simulate a delay
    private static void delay(int i) {
        try { TimeUnit.SECONDS.sleep(i); } 
        catch (InterruptedException e) {}
    }

    private static String doThing1() {
        delay(1);
        return "Hello";
    }

    private static String doThing2(String input) {
        delay(1);
        return input + " World";
    }

    private static String doThing3(String input) {
        delay(1);
        return input + "!!";
    }


    // Create an instance of a Runnable that implements our logic
    private static Runnable myRunnable = () -> {
        String result1 = doThing1();        // Blocking, or so it seems...
        String result2 = doThing2(result1); // Blocking, or so it seems...
        String result3 = doThing3(result2); // Blocking, or so it seems...
        System.out.println("Result from the thread: " + result3);
    };


    public static void main(String[] args) {
        // Create a Virtual Thread, pass the Runnable, and start it:
        Executors.newVirtualThreadPerTaskExecutor().submit(myRunnable);

        // Do other work...

        // Introduce a delay to prevent the JVM 
        // from exiting before work is complete.
        delay(4);
    }
}


The Executors factory class from Java 5 is back with a new factory method, newVirtualThreadPerTaskExecutor().  This returns an ExecutorService which supplies a Virtual Thread rather than a standard platform thread.  The Runnable will execute within a Virtual Thread which itself runs within a platform thread.  The sleep delay has returned only because all work is being done in a daemon thread, which would be interrupted in our demo if the main thread exited.

So what about the blocking?  Virtual Threads run within platform threads served from a modified ForkJoinPool.  A special internal object called the Continuation manages multiple Virtual Threads within a platform thread.  When a blocking call is detected, code within the Virtual Thread makes a call to Continuation.yield() to allow another VirtualThread to have a turn running.  The yield() process unmounts the VirtualThread from the platform thread, copies its stack memory into heap memory, then begins running the next Virtual Thread on its wait list.  When an OS handler signals that a result from the blocking call is complete, it calls a Continuation.run() to place this VirtualThread back on the wait list for the platform thread.  If this platform thread is busy, the work-stealing capability of the ForkJoinPool comes into play, and any other available platform thread can resume the work.  The blocking penalty has been eliminated with a bit of memory management - much more efficient.

But how does the VirtualThread detect blocking?  Java 21 modified nearly every line of code in the JDK that could result in a blocking.  These now call the yield() process when blocking begins.  This was a massive change to the JDK, over one thousand files were part of the pull request.

The result is that we can run normal imperative-style code in a Virtual Thread and completely ignore blocking penalties, because their ill effects have been eliminated.  Virtual Threads allow the underlying platform thread to stay busy with other Virtual Threads while our code waits for results.  Fewer threads do more work, just as we saw in Reactive Streams or JavaScript, but we don't need to embrace advanced asynchronous techniques or callbacks to enjoy these benefits.  It is hard to overstate the revolutionary impact this has on how we can code in Java from this point forward, replacing the need for elegant but hard-to-grasp techniques like CompleteableFuture or Reactive Streams with simple-as-Sunday imperative programming.

Virtual Threads are not a silver bullet and are not appropriate for every situation.  Virtual Threads impose a slight operational burden vs running directly in a platform thread, mainly due to the unmounting and remounting of memory when yield()/run() calls occur.  Workloads which would not incur blocking, such as long running calculations, would be more appropriately implemented using the multithreading or reactive techniques described earlier in this article.  Native code and code with synchronization blocks can run in Virtual Threads, but these tasks are pinned to a single platform thread, thus eliminating the work-stealing benefit.

One possible danger to consider: the code in the example above runs efficiently only when running in a Virtual Thread vs a platform thread.  The modifications added throughout the JDK codebase only take effect when code is running within a Virtual Thread.  If this code were modified slightly, such as removing the Runnable or calling a different Executors factory method, the result would be the same inefficient blocking we've spent 25 years trying to avoid.  It can sometimes be difficult for a busy developer to notice such a subtle change.

To summarize Java Virtual Threads

The blocking penalty experienced since the early days of Java can be eliminated in most practical cases.
Developers can read, write, test, debug, and handle exceptions using easy-to-use imperative style programming.
Multithreading is still preferred in non-blocking situations, such as long-running calculations.
I hope you've enjoyed this walk down memory lane culminating in the current state of the art.  Be sure to explore the coding examples further if you like.





