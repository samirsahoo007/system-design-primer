# What do you know about garbage collection?
As an interviewer, I’d like to start the discussion by asking “tell me about what you know about garbage collection”. This question can give me an idea of how familiar the candidate is with this topic.

Remember that to perform well in this interview, you don’t necessarily need to know a lot about garbage collection as interviewers care about analysis rather than knowledge. On the flip side, knowing a lot about garbage collection doesn’t mean you can easily pass the interview.

Back to the question, **garbage collection is a system that automatically recycles unused memory in programming languages. A most popular example is Java/Python. When writing Java/Python, you don’t need to control how memory is allocated and recycled. Everything happens in the background.**

## Describe Python's garbage collection mechanism in brief. memory management in python:
Python uses two strategies for memory allocation reference counting and garbage collection.
        * reference counting => Python uses reference counting to detect inaccessible objects. Python maintains a count of the number of references to each object in memory. If a reference count goes to zero
                                then the associated object is no longer live and the memory allocated to that object can be freed up for something else
                                Periodically executing a cycle detection algorithm which looks for inaccessible cycles and deletes the objects involved.
        * garbage collection => Occasionally things called "reference cycles" happen. The garbage collector periodically looks for these and cleans them up. An example would be if you have
                                two objects o1 and o2 such that o1.x == o2 and o2.x == o1. If o1 and o2 are not referenced by anything else then they shouldn't be live. But each of them has a reference count of 1.

•       Certain heuristics are used to speed up garbage collection. For example, recently created objects are more likely to be dead. As objects are created, the garbage collector assigns them to generations.
        Each object gets one generation, and younger generations are dealt with first.

## Pros & cons
So why do we need garbage collection? Many people can easily tell the advantages of garbage collection, but get confused when asked about the disadvantages. Again, if you have a good grasp of how program works, you can easily come up with good answers by analyzing the problem and this is not something you need to remember as facts.

The most obvious benefit of having garbage collection is that it makes programming much easier. Remember when we write C++, we need to be extremely careful about pointers and memory allocation. By handling all of these by the program itself, developers can focus more on the core logic.

More specifically, garbage collection helps developers avoid several memory problems. First, it prevents accessing **dangling pointers** that point to an object that no longer exists. Secondly, it prevents freeing a region of memory that is already freed (double free). Last, it avoids memory leak, *which means an unreachable region of memory that can never be freed*. All of them are common pitfalls when developers try to manage memory manually.

So what are the disadvantages? Apparently, there are many languages that don’t have built-in garbage collection like C++.

The biggest disadvantage is that garbage collection consumes computing resources. Think about this, not only does garbage collection need to implement logics to recycle memory, it also consumes memory to store the status of objects. In some naive garbage collection implementation, the recycle process may even block the program potentially.

Another way to think about this is that without garbage collection, the developer has the full control over how memory is used, which gives the program much more flexibility and much easier to optimize. That’s one of the reasons why C++ is more efficient. Of course, it’s also prone to error.


## Design a simple garbage collection system
So how would you design a garbage collection system? Try to think about this problem by yourself. It’s even better if you have no prior knowledge. Instead of jumping to solutions, I’d like to tell you how to analyze this problem step by step.

Since the essence of a garbage collection system is to recycle unused memory in the program, the key is to identify which piece of memory is unused. More specifically, we should search for variables that are no longer referenced.

If you think about all the objects (variables) in a program, it’s like a directional graph that each object references other objects and at the same time is also referenced by some objects. As a result, unreachable objects, which are those without any reference, should be recycled. As you can see, the big problem has been simplified to a graph problem – find unreachable nodes in a graph.


### Naive mark-and-sweep
In fact, the above solution is just the most naive approach, which is called mark-and-sweep. To begin with, the garbage collection system does a tree traversal following object references and mark all the visited objects. In the second phase, for all the unreachable objects, free their memory(objects are stored in heap).

But how does the system track those unreachable objects? One easy way is to keep a set of all the objects in the program. Whenever a new object is initialized, add it to the pool.

The idea of the naive mark-and-sweep approach is quite straightforward. The system does a tree traversal following object references and mark all the visited objects. In the second phase, for all the unreachable objects, free their memory.

Apparently, the approach is easy to understand and implement. So what are the disadvantages?

The most notable problem is that the entire system must be suspended during garbage collection. In other words, once in a while, the problem will be frozen when doing garbage collection and no mutation of the working set can be allowed. Thus, it’ll significantly affect the performance of time-critical applications.

#### Improvement
Given the performance issue of mark-and-sweep, one modern garbage collection system takes a slightly different approach – Tri-color making.

Let me briefly introduce the algorithm. In a nutshell, the system marks all the objects into three colors:

* White – objects that have no reference and should be recycled.
* Black – reachable objects that shouldn’t be recycled. Black objects have no reference to white objects.
* Gray – objects that are reachable from roots and yet to be scanned for references to white.

Initially, all the objects that are referenced from roots are in gray and the white sets include everything else (black is empty). Each time the system picks an object from gray to black and move all its references from white to gray. In the end, gray becomes empty and all white objects are recycled.

The most notable advantage is that the system can do garbage collection on the fly, which is accomplished by marking objects as they are allocated and during mutation. Thus, the program won’t be halted for long time and performance gets improved.


### Reference counting
So what are other ways to design a garbage collection system that won’t freeze the program?

A natural solution is reference counting and the idea is extremely simple. The core concept of garbage collection is when an object has zero reference, we should recycle it as soon as possible. So why not just keep track of the reference count for each object?

The reference counting system will keep a counter for each object that counts the number of references it has. The counter is incremented when a reference to it is created, and decremented when a reference is destroyed. When the counter is 0, the object should be recycled. Obviously, the system can do the garbage collection on the fly since it’s able to release the memory at the right time.


#### Disadvantage of reference counting
Apparently, the reference counter adds space overhead to the whole system. Since every single object needs additional storage for its reference count, the overall space needed can be increased significantly without any optimization.

Another problem is the speed overhead. Since the system needs to keep updating the counter, every operation in the program requires modification of one or more reference counters. Another way to understand this is that instead of freeze the program to recycle objects, reference counting system divides the overhead into every small operation. Since you can’t get everything for free, you need to decide if you want every operation becomes slightly slower or stop the entire program once in a while.

In addition, cycles can also be a problem of reference counting. If two objects reference each other, they will never be recycled. If you have experience with obj-c, you should already know the solution. Some languages introduce the concept of “weak reference” for the back pointers that creates the cycle. It is a special reference object whose existence does not increment the reference count of the referent object.


Ref: 
http://blog.gainlo.co/index.php/2016/07/25/design-a-garbage-collection-system-part-i/
http://blog.gainlo.co/index.php/2016/08/08/design-garbage-collection-system-part-ii/

