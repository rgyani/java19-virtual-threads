# Virtual Threads - Java 19 - Project Loom

* Virtual threads are a lightweight implementation of Java threads, delivered as a preview feature in Java 19.
* Virtual threads reduce the effort of writing, maintaining, and observing high-throughput concurrent applications.
* Virtual threads breathe new life into the familiar thread-per-request style of programming, allowing it to scale with near-optimal hardware utilization.
* Virtual threads are fully compatible with the existing `Thread` API, so existing applications and libraries can support them with minimal change.
* Virtual threads support the existing debugging and profiling interfaces, enabling easy troubleshooting, debugging, and profiling of virtual threads with existing tools and techniques.

Java 19 brings the first preview of **virtual threads** to the Java platform; this is the main deliverable of **OpenJDKs Project Loom**. 

Virtual threads fundamentally change how the Java runtime interacts with the underlying operating system, eliminating significant impediments to scalability -- but change relatively little about how we build and maintain concurrent programs.

There is almost zero new API surface, and virtual threads behave almost exactly like the threads we already know. 

### Threads
Threads are foundational in Java. When we run a Java program, its _main_ method is invoked as the first call frame of the "main" thread, which is created by the Java launcher.  
When one method calls another, the callee runs on the same thread as the caller, and where to return to is recorded on the threads stack. When a method uses local variables, they are stored in that methods call frame on the threads stack

Threads are the basic unit of scheduling in Java programs; when a thread blocks waiting for a storage device, network connection, or a lock, the thread is descheduled so another thread can run on that CPU.   
Java was the first mainstream language to feature integrated support for thread-based concurrency

### Platform threads

Most JVM implementations today implement Java threads as thin wrappers around heavyweight, OS-managed threads platform threads.  
This reliance on OS threads has a downside: because of how most OSes implement threads, thread creation is relatively expensive and resource-heavy.  
This implicitly places a practical limit on how many threads we can create, which in turn has consequences for how we use threads in our programs.  
OSs typically allocate thread stacks as monolithic blocks of memory at thread creation time that cannot be resized later. This means that threads carry with them megabyte-scale chunks of memory to manage the native and Java call stacks.

Limiting how many threads we can create is problematic because the simplest approach to building server applications is the thread-per-task approach: assign each incoming request to a single thread for the lifetime of the task.  
Thread-per-task scales well enough for moderate-scale applications -- we can easily service 1000 concurrent requests -- but we will not be able to service 1M concurrent requests using the same technique, even if the hardware has adequate CPU capacity and IO bandwidth.

### Virtual threads
Virtual threads are an alternative implementation of java.lang.Thread which store their stack frames in Javas garbage-collected heap rather than in monolithic blocks of memory allocated by the operating system.   
We don't have to guess how much stack space a thread might need, or make a one-size-fits-all estimate for all threads; the memory footprint for a virtual thread starts out at only a few hundred bytes, and is expanded and shrunk automatically as the call stack expands and shrinks.

The operating system only knows about platform threads, which remain the unit of scheduling.  
To run code in a virtual thread, the Java runtime arranges for it to run by mounting it on some platform thread, called a carrier thread.  
Mounting a virtual thread means temporarily copying the needed stack frames from the heap to the stack of the carrier thread, and borrowing the carriers stack while it is mounted.

When code running in a virtual thread would otherwise block for IO, locking, or other resource availability, it can be unmounted from the carrier thread, and any modified stack frames copied are back to the heap, freeing the carrier thread for something else (such as running another virtual thread.) 

Nearly all blocking points in the JDK have been adapted so that when encountering a blocking operation on a virtual thread, the virtual thread is unmounted from its carrier instead of blocking.

Mounting and unmounting a virtual thread on a carrier thread is an implementation detail that is entirely invisible to Java code. 
ThreadLocal values of the carrier thread are not visible to a mounted virtual thread; the stack frames of the carrier do not show up in exceptions or thread dumps for the virtual thread. 
During the virtual threads lifetime, it may run on many different carrier threads, but anything depending on thread identity, such as locking, will see a consistent picture of what thread it is running on.


### Virtual Threads vs Virtual Memory
Virtual threads are so-named because they share characteristics with virtual memory.   
With virtual memory, applications have the illusion that they have access to the entire memory address space, not limited by the available physical memory.   
The hardware completes this illusion by temporarily mapping plentiful virtual memory to scarce physical memory as needed, and when some other virtual page needs that physical memory, the old contents are first paged out to disk.  
Similarly, virtual threads are cheap and plentiful, and share the scarce and expensive platform threads as needed, and inactive virtual thread stacks are "paged" out to the heap.


### Performance Characteristics
Despite the difference in creation costs,
* **virtual threads are not faster than platform threads**; we cant do any more computation with one virtual thread in one second than we can with a platform thread. 
* **Nor can we schedule any more actively running virtual threads than we can platform threads**; both are limited by the number of available CPU cores.

So, what is the benefit? 
Because they are so lightweight, **we can have many more inactive virtual threads than we can with platform threads**. 

At first, this may not sound like a big benefit at all! 
But "lots of inactive threads" actually describes the majority of server applications. 
Requests in server applications spend much more time doing network, file, or database I/O than computation. So if we run each task in its own thread, most of the time that thread will be blocked on I/O or other resource availability.  
Virtual threads allow IO-bound thread-per-task applications to scale better by removing the most common scaling bottleneck -- the maximum number of threads -- which in turn enables better hardware utilization.   
Virtual threads allow us to have the best of both worlds: a programming style that is in harmony with the platform rather than working against it, while allowing optimal hardware utilization.
  
For CPU-bound workloads, we already have tools to get to optimal CPU utilization, such as the fork-join framework and parallel streams.   
Virtual threads offer a complementary benefit to these.  
Parallel streams make it easier to scale CPU-bound workloads, but offer relatively little for IO-bound workloads; virtual threads offer a scalability benefit for IO-bound workloads, but relatively little for CPU-bound ones.


### Virtual threads do not replace platform threads; they are complementary.
However, many server applications will choose virtual threads (often through the configuration of a framework) to achieve greater scalability.



### Virtual threads in action
The following example creates 100,000 virtual threads that simulate an IO-bound operation by sleeping for one second. It creates a virtual-thread-per-task executor and submits the tasks as lambdas.

```
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        });
    });
}  // close() called implicitly
```

* With virtual threads, running this program takes about 1.6 seconds in a cold start, and about 1.1 seconds after warmup. 
* If we try running this program with a cached thread pool instead, depending on how much memory is available, it may well crash with OutOfMemoryError before all the tasks are submitted.   
* And if we ran it with a fixed-sized thread pool with 1000 threads, it wont crash, but it will take 100 seconds to complete.




