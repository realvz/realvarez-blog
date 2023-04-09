---
title: Weak References - A weapon against Out Of Memory Errors
description: How to give and receive constructive feedback 
date: 2022-05-28
layout: layouts/post.njk
---

One of the key features that containers offer us is the ability to apply resource control through [Control Groups](https://man7.org/linux/man-pages/man7/cgroups.7.html), generally referred to as cgroups. Containers, which build on top of Linux cgroups, allow us to run process that have limited access to host's total capacity.

This ability allows us to co-locate containers on the same host without being concerned that a container may impact other containerized apps, the so called *noisy neighbor problem*.

While cgroups allow us to control many resources, the most used controls are CPU and memory. For scalable, multi-tenant clusters like Kubernetes or Amazon ECS, resource limiting is a best practice.

ECS and Kubernetes allow us to limit resource consumption at container and ECS task or Kubernetes pod level. Below are the requests and limits a pod can have:

- `spec.containers[].resources.limits.cpu`
- `spec.containers[].resources.limits.memory`
- `spec.containers[].resources.limits.hugepages-`
- `spec.containers[].resources.requests.cpu`
- `spec.containers[].resources.requests.memory`
- `spec.containers[].resources.requests.hugepages-`

Similarly, Docker desktop allows configuring memory and CPU limits. Should a container try to use more memory than allocated, it gets *killed*.

```python
[336546.736392] Out of memory: Kill process 9218 (java) score 330 or sacrifice child
[336546.738454] Killed process 9218 (java) total-vm:7543324kB, test-mem:1060364kB, test-mem2:0kB, test-mem-rar:0kB
[336547.071919] oom_reaper: reaped process 9218 (java), now test-mem:0kB
```

## Memory Limits

Many enterprises are in the process of moving legacy applications to containers clusters. Some of these applications ran on dedicated hardware. Their business requirements didn’t include operating with resource constraints.

Having dedicated resources meant that traditional applications were free to consume as much system resources as available. Although Java allows us to control the heap size, it is rarely used to restrict memory usage for production applications.

When these applications migrate to containers and a limit is put upon their memory consumption, new out of memory errors may emerge. A common remedy is to allocate more memory until the application stops crashing.

In many cases, that might be the best solution. Pay a bit more of RAM, and move on. Code rot makes changes expensive with each passing day. But if you do have the resources to refactor code, optimizing memory consumption will improve the reliability of your applications.

## 🧹 Garbage Collection

One question we may want to ask ourselves is: What are some of the common ways of using memory inefficiently? Once you’ve fixed all the memory leaks, what’s next?

C programmers had to manage memory manually using malloc() and free(). Thankfully, higher-level languages automatically allocate memory when programs create objects and free memory when the object is not required.

When a process creates a new object, the system will allocate it memory and return the pointer to the program. Typically, a program’s variables (or objects) will remain in memory as long as the code is running. In programming languages like Python, objects include a reference count field, which counts how many objects are referencing it. JavaScript object has a reference to its prototype (implicit reference) and to its properties values (explicit reference). As long as an object has a non-zero reference, it’s immune to garbage collection.

Once the process terminates, the garbage controller is free to remove the associated objects from memory as they don’t have any references. Objects created like these have a strong reference, which is the default behavior in all programming languages. They remain in memory until the process terminates.

### 👉🏼 Weak Referencing

One way to reduce memory consumption is by decreasing the amount and size of the objects stored in memory.

Many languages also support creating objects with weak reference. Garbage collectors can remove any variable stored with weak reference even while the process that created it is still running. This provides an excellent way to store objects in memory that can be safely removed if the system comes under memory pressure.

Weak Reference objects allow applications to safely build caches without risking running out of memory.

### 🏁 Downsides of weak referencing

Weak referencing forms the foundation for caching. And, similar to caching, it requires *getting and setting* a variable before usage. So, checking a variable if it exists before reading or writing to it is necessary for weak references.

Can you imaging what would happen if a process tries to read its variable that’s stored as a weak reference, and the garbage controller has deleted it? Nothing good.

### 🎚 Different levels of weak references

In addition to the weak references described above, languages like Java provide *Soft References* and *Phantom References.* Here's an [article](https://dzone.com/articles/weak-soft-and-phantom-references-in-java-and-why-they-matter) that goes over the differences. The TLDR is that the garbage collector removes weak references first, then soft references, and finally phantom references.

If your language provides different levels of weak references, consider choosing one that suits your application's requirements.

### 🫱🏽‍🫲🏼 External caching

If your application runs multiple replicas or if multiple processes are caching similar data, consider using Redis or Memcached as an external, distributed, and shared cache. These caching solutions are excellent for web services as they allow multiple replicas to share a cache, which allow us to store user’s sessions state without requiring sticky sessions.

## 🏆 Bonus tip

Another common cause for OOM errors in web applications is not keeping the payload size minimal. Web servers stores data for every session in memory until the session terminates. When a client requests a large payload, and the server doesn’t limit it, the systems stores the payload in memory before it is sent to the client.

We can limit the payload size by restricting the maximum amount of data sent to a particular client. For example, you can paginate results from a database instead of sending the entire dataset to the client.

## References

[weakref — Weak references — Python 3.10.4 documentation](https://docs.python.org/3/library/weakref.html)

[Life of a Container](https://indradhanush.github.io/blog/life-of-a-container/)

[Memory Management - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management)

