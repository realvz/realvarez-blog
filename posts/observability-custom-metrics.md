---
title: Close observability gaps with custom metrics
date: 2022-05-06
layout: layouts/post.njk
---
> *Which application metrics should you collect?*

I frequently engage with customers that are amid breaking their monolithic applications into smaller microservices. Many teams with also see this migration as an opportunity to make applications more observable. As a result, customers inquire which metrics they should monitor for a typical cloud native application.

Previously, when a customer asked me how to instrument a service, I pointed them to the well known [USE and RED methods](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/). But, I felt the response wasn’t thorough. A list of specific metrics to monitor can be helpful for teams building cloud native applications. This post is an attempt to provide a list of metrics to collect in a typical application. Not all the metrics listed below apply to every application type. For example, batch-like workloads rarely serve traffic, and resultantly, don't need to keep a log of requests-served.

The goal of this document is to help developers come up with the golden signals for their applications.

> Golden Signals, a term used first in the [Google SRE handbook](https://sre.google/sre-book/monitoring-distributed-systems/). Golden Signals are four metrics that will give you a very good idea of the real health and performance of your application as seen by the actors interacting with that service, whether they are final users or another service in your microservice application.

## Observability

Cloud best practices recommend building systems that are observable. While the word observability (or “*O11y*” as it is popularly known) doesn’t have an official definition, it is the measure of a system’s ability to expose its internal state. The three pillars of observability are logs, metrics, and traces.

Modern systems are designed to produce logs, *emit* metrics, and provide traces to help developers and operators understand its internal state.

> [Push vs Pull](https://giedrius.blog/2019/05/11/push-vs-pull-in-monitoring-systems/)

> Emitting metrics by exposing them on an externally accessible HTTP endpoint is gaining wider adoption thanks to developers adopting Prometheus for monitoring. In this model, Prometheus pulls metrics by scraping the application’s `/metrics` endpoint.

> When you run Node Exporter, it publishes metrics at `http://localhost:9100/metrics`

Observability tools aggregate and analyze data from different sources to help you detect issues and identify bottlenecks. The goal is to use these system signals to improve its reliability and prevent downtime.

AIOps products like [Amazon DevOps Guru](https://aws.amazon.com/devops-guru/) can also detect anomalies using your application's logs, metrics, and traces (and other sources) and give you early signals to prevent a potential disruption.

## Metrics to collect

For an application to function as designed, the application and its underlying system have to be *healthy*. Host metrics inform the operator of the host’s and infrastructure resource usage, like CPU, memory, I/O, etc. If you use Prometheus, [Node Exporter](https://github.com/prometheus/node_exporter) collects this information automatically for you.

Host metrics rarely differ. Whether we run a process on an EC2 instance or a Raspberry Pi, we’re interested in the same metrics.

Unlike host metrics, application metrics are unique to each microservice. Application metrics are supposed to provide the operator the information so they can do these things:

1. Identify future areas of improvement by providing code-specific measurements. Application monitoring or APM tools provide measurements over a segment of time that developers can analyze.
2. When the system fails, provide information for troubleshooting and prevention.
3. In some cases, provide early signals to business. For example, if the application exposes, the *orders* it has processed in the last 60 minutes can be tracked using the monitoring system, rather than querying a relational database.

There are several companies like application monitoring or APM companies like New Relic, DataDog that have products to aggregate application metrics using SDKs or agents. However, what they will not collect are the business specific metrics that only your application cares about.

In order to create a list of relevant metrics for an application, its architects will need to determine a signal for its every key function. The hallmark of a microservice is that it does *one thing well*, therefore it shouldn’t have many key functions. Start by white-boarding the functions implemented in the code and creating a list of metrics that would help you gauge its performance (or its availability at the least).

Most measurements you’ll do will fall under one of these categories:

#### Counter

As the name suggests, this value is incremented when a function runs. Example: total requests served

#### [Histogram](craftdocs://open?blockId=969CA66C-2955-405E-874A-E23D5F98AA26&spaceId=9d54cc03-adfe-f72f-3389-565eb7356d1d)

Histograms are charts that show the frequency of the occurrence of several ranges of values. A histogram samples observations (usually things like request durations or response sizes) and counts them in configurable buckets.

#### Gauge

This type is metric tracks a value that increases or decreases over a period. Example: number of threads.

---

With that background, let’s go through the list of common custom metrics developers use.

### Network activity

These are the obvious metrics to track for any application that serves traffic. Network metrics tell you how much load is placed on the system. Over the time, these data points assist you when devising the scaling strategy for the system.

Things you should include are:

- Request count by API type or page
- Requests total
- Transactions
- Concurrent, expired, and rejected sessions
- A watermark that records maximum concurrent sessions
- Average processing time
- A count by error type

### Resource usage

It is a best practice to monitor a systems *saturation*, which is a measure of your systems resource consumption. Every resource has a *breaking point*, beyond which additional stress causes performance degradation. Scalable and reliable systems are designed to never breach the breaking point.

However, simply collecting overall resource saturation at an application level is insufficient. You also need to look deeper at thread or resource pool level.

Consider collecting these metrics:

- Number of processors, system CPU load, process CPU load, available memory, used memory, available swap, used swap, open file descriptor count.
- Total resources consumed by connection pools, thread pools, and any other resource pools.
- Total started thread count, current thread count, current busy threads, keep alive count, poller thread count, and connection count.
- Objects created, destroyed, and checked out, high-water mark, number of times checked out,
- Number of threads blocked waiting for a resource, number of times a thread has blocked waiting

Common frameworks like Tomcat, Flask, etc. support exporting pre-defined metrics. For example, JMX already exposes a bunch of these metrics. See [AWS CloudWatch documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights-Prometheus-Sample-Workloads-javajmx.html).

### Users

Besides, serving the intended audience, bots or scripts flood internet facing web servers with requests. These automated requests can overload the system if unauthenticated requests are improperly handled (for example, not redirecting all unauthenticated requests to the authentication service and attempting to process an unauthenticated request).

Here are user related metrics to collect:

- Authenticated and unauthenticated requests
- Demographics, authenticated and unauthenticated requests, usage patterns,
- Unsuccessful login attempts

Some of these metrics may also come from your Load Balancer or ingress.

### Business transaction (for each type)

If your application follows the microservices approach, then the code fulfills one function, at least that’s the idea. What are the key performance indicators for your app’s function? Define them and track these metrics.

Should future releases cause performance regression, you’ll be able to detect it. Tracking these business metrics will help you track trends easily and avoid a cascading failure.

Here are common things that services care about:

- Orders, messages, requests, transactions processed
- Success and failure rates. For a retailer, this could be the conversion rate.
- Service level agreements (like average transaction response time)

If you still need help with identifying key metrics, ask yourself this question: In what ways can my application negatively affect the business even when it might appear to be healthy?

### Database connections

Along with monitoring your database instances using database monitoring tools, consider collecting database connection health metrics in your application. This is especially helpful if your application uses a shared database. If your application encounters database connection errors but the database remains operational for other application, you know the problem is on the application side, and not the database.

Consider recording these databases-related metrics:

- A count of `SQLException` thrown
- Number of (concurrent or maximum)queries
- Average query run time

### Data consumption

Wherever you’re persisting data, you need to ensure that you’re going to go over your quotas and run out of space. Besides, monitoring on disk and in-memory data volumes, don’t forget to monitor the data your application stores in databases and caches.

### Cache health

Speaking of cache, it is a best practice to monitor these metrics:

- Items in cache
- Get and set latency
- Hits and miss rates
- Items flushed

Also, consider using an external cache such as Redis or Memcached.

### External services

Keeping a track of how downstream services perform is also useful in understanding issues. Along with using timeouts, retries (preferably with [exponential backoff](https://en.wikipedia.org/wiki/Exponential_backoff)), and circuit breakers, consider monitoring these metrics for every external service your service's proper functioning depends on:

- Circuit breaker status
- Count of timeouts, requests
- Average response time or latency
- Responses by type
- Network errors, protocol errors
- Requests in flight
- A high watermark of concurrent requests.

### Granularity in metrics collection

The frequency at which you publish and collect metrics depends on your business requirements. For a retailer, knowing traffic patterns by the hour and day is useful in scaling capacity. Similarly, a travel company’s traffic pattern are influenced by holiday schedules.

Amazon EC2 provides instance metrics at 1-minute interval, which is a good start for critical metrics.

Remember that there’s a cost attached to exposing, collecting, and analyzing metrics. Collecting unnecessary information in metrics can put a strain on the system and slow down troubleshooting.

Consider giving the operator the control over the metrics your code should generate. This way, you can turn on specific metrics whenever needed.

## Conclusion

Finding out which metrics to collect is an answer that only the most familiar with the code can answer. This post provides a list of metrics for you to get started.

Are there any metrics that I have overlooked? Let me know at [@realz](https://twitter.com/realz).

## References

[Push Vs. Pull In Monitoring Systems – Giedrius Statkevičius](https://giedrius.blog/2019/05/11/push-vs-pull-in-monitoring-systems/)

[SRE Metrics: Four Golden Signals of Monitoring](https://www.splunk.com/en_us/blog/learn/sre-metrics-four-golden-signals-of-monitoring.html)

[Learning Modern Linux](https://www.oreilly.com/library/view/learning-modern-linux/9781098108939/)

[Release It!](https://www.oreilly.com/library/view/release-it/9781680500264/)

## Appendix

### Instrumentation

Instrumentation is the way to measure an application’s performance. It is highly useful in profiling and troubleshooting. There are two common strategies for instrumentation:

- Auto instrumentation. This is generally done using a library like OpenTelemetry API and SDK. For more see “[What Is Auto-Instrumentation?](https://www.honeycomb.io/blog/what-is-auto-instrumentation/)”.
- Custom instrumentation. Whenever your instrumentation needs are not met by auto instrumentation, you will also generate custom metrics.

