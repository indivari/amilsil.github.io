---
layout: post
title: Unity Lifetime Managers, things devs should know about
tags:
- c#
- unity
- ioc
- patterns
---

Unity is one of the widely used IOC containers. While using any IOC container, it's always recommeneded to create only the instances that you need. But I have seen many below par unity configurations. In this article I hope to discuss different lifetime managers in order for us to better understand how they work.

I will use a small program I have written to demonstrate each.

    https://github.com/amilsil/unitytest.git


## The Default

{% highlight csharp %}
container.RegisterType<IMyObject, MyObject>()
{% endhighlight %}

When no lifetime manager is defined, unity defaults to `Transient`. That means the instance is **short lived** and an instance is created per evey object that wants one. This, however, is not the best in terms of performance, resource utilization or well, self satisfaction. But **it just works**. 


## 5 Lifetime Managers
Unity comes with 5 lifetime managers. 

1. **Transient** (default) - Creates an instance per injection
3. **PerResolve** - Creates an instance per resolve
4. **PerThread** - Creates an instance per thread
2. **Container Controlled** - Creates an instance per container. Lifetime of the instance is the same of the container.
5. **Externally Controlled** (not readymade for use) - Instance lifetime is maintained externally.

All of above has a dedicated usage. Let's just run through a code and the output, that demonstrates what each does. I will skip Externally Controlled lifetime managers here, for that it's a dedicated topic itself how & what usages it has.

The Unity registration below is setup with an object each with each different Lifetime Manager. 

{% highlight csharp %}
public class UnityConfig
{
    public static IUnityContainer RegisterComponents()
    {
        var container = new UnityContainer()
            .RegisterType<IObjectForTransient, ObjectForTransient>(new TransientLifetimeManager())
            .RegisterType<IObjectForContainerControlled, ObjectForContainerControlled>(new ContainerControlledLifetimeManager())
            .RegisterType<IObjectForPerResolve, ObjectForPerResolve>(new PerResolveLifetimeManager())
            .RegisterType<IObjectForPerThread, ObjectForPerThread>(new PerThreadLifetimeManager())

            .RegisterType<ICandidate, Candidate>()
            .RegisterType<ICandidateService, CandidateService>();

        return container;
    }
}
{% endhighlight %}

### Here is the dependency hierarchy for the code above

Both the `Candidate` and `CandidateService` classes each are injected with all of the object classes. The `Candidate` is additionally injected in to the `CandidateService` class.

![injection hierarchy]({{ site.baseurl }}/images/unity_dependency_brief.jpg "dependency hierarchy")
For simplicity I only used arrows for one object. But each object is injected alike.

The main `Program` class tries to resolve the `CandidateService` a couple of times. Bear in mind here, the `CandidateService` requires instances of all 5 Object classes and the `Candidate` class.

{% highlight csharp %}
static void Main(string[] args)
{
    var container = UnityConfig.RegisterComponents();
    var lockObject = new object();

    Action resolveOnce = () =>
    {
        lock (lockObject)
        {
            Console.WriteLine("Creating all on Thread {0}", Thread.CurrentThread.ManagedThreadId);
            container.Resolve<ICandidateService>();
            Console.WriteLine();
        }
    };

    // Resolve twice on the main thread.
    Console.WriteLine("MAIN THREAD");
    resolveOnce();
    resolveOnce();

    // Resolve twice on two new, separate threads.
    Console.WriteLine("TASK RUN");
    Task.Run(resolveOnce);
    Task.Run(resolveOnce);

    Console.Read();
}
{% endhighlight %}

## Output
Here's the output from running the code above.
{% highlight text%}
MAIN THREAD 
Creating all on Thread 8 
// first resolve
Creating ObjectForContainerControlled // once for container
Creating ObjectForTransient // for Candidate
Creating ObjectForPerResolve // for Candidate & CandidateService
Creating ObjectForPerThread // for Thread Main
Creating ObjectForTransient // for CandidateService

Creating all on Thread 8
// second resolve 
Creating ObjectForTransient // for Candidate
Creating ObjectForPerResolve // for Candidate & CandidateService
Creating ObjectForTransient // for CandidateService

TASK RUN 
Creating all on Thread 10 
// new thread A
Creating ObjectForTransient // for Candidate
Creating ObjectForPerResolve // for Candidate & CandidateService
Creating ObjectForPerThread // for ThreadA
Creating ObjectForTransient // for CandidateService

Creating all on Thread 9 
// new thread B
Creating ObjectForTransient // for Candidate
Creating ObjectForPerResolve // for Candidate & CandidateService
Creating ObjectForPerThread // for ThreadB
Creating ObjectForTransient // for CandidateService
{% endhighlight %}

### So, what does this mean?
1. **Transient**

    An instance is created every time it needs to be injected. 
3. **PerResolve**

    An instance is created every time it needs to be injected, per resolve. If two injections of the same class is required within the same resolve, one single instance is reused.
4. **PerThread**

    An instance is created per thread. Remember though, if you use the managed thread pool, and if a previous thread that had an instance is taken from the ThreadPool, the same instance is used next time.
2. **ContainerControlled**

    Only one instance is created per container. This is equal to the **Singleton** pattern.

    >   Remember that traditional Singletons are now dead. For we create hard dependencies with them and hard dependencies make less maintainable code.
5. **Externally Controlled**

    The instance creation is controlled externally. This is quite rarely used at least in my experience. One usage however is when you are dependent on a third party to create the instances for you. Additionally, those instance creation is too complex to be registered with unity.

## Take away
If we design unity configurations carefully, we can save quite a lot of resources. For example, in most cases, an instance per Resolve or per Container will do the task. I have found scenarios where a good configuration reduced the number of instances from 40s to just 16, yet doing the same thing robustly.
