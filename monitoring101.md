This post is part of a series on effective monitoring.  
这篇文章是关于有效监控的系列文章的一部分。  
Be sure to check out the rest of the series: <u>[Alerting on what matters][1]</u> and <u>[Investigating performance issues][2]</u>.  
请务必查看本系列的其余部分：<u>[提醒重要事项][1]</u>和<u>[调查性能问题][2]</u>  
Monitoring data comes in a variety of forms—some systems pour out data continuously and others only produce data when rare events occur.   
监控数据有多种形式——一些系统不断地倾倒数据，而另一些系统仅在发生罕见事件时才产生数据。  
Some data is most useful for identifying problems;  
有些数据对于识别问题最有用；  
some is primarily valuable for investigating problems.  
有些主要用于调查问题。  
More broadly, having monitoring data is a necessary condition for observability into the inner workings of your systems.  
更广泛的说，拥有监控数据是可观察到系统内部工作的必要条件。  
This post covers which data to collect, and how to classify that data so that you can:  
这篇文章介绍了要收集哪些数据，以及如何对这些数据进行分类，以便您可以：  
1.Receive meaningful, automated alerts for potential problems  
  接收有关潜在问题的有意义的自动报警  
2.Quickly investigate and get to the bottom of performance issues  
  快速调查并深入了解性能问题  
Whatever form your monitoring data takes, the unifying theme is this:  
无论您的监控数据采用何种形式，统一的主题是：  
> Collecting data is cheap, but not having it when you need it can be expensive, so you should instrument everything, and collect all the useful data you reasonably can.  
> 收集数据很便宜，但在你需要的时候没有它可能会很昂贵，所以你应该检测一切，并尽可能收集所有有用的数据。

This series of articles comes out of our experience monitoring large-scale infrastructure for <u>[our customers][3]</u>.  
本系列文章源自我们为<u>[客户][3]</u>监控大型基础设施的经验。  
It also draws on the work of <u>[Brendan Gregg][4]</u>, <u>[Rob Ewaschuk][5]</u>, and <u>[Baron Schwartz][6]</u>.  
它还借鉴了<u>[Brendan Gregg][4]</u>、<u>[Rob Ewaschuk][5]</u>和<u>[Baron Schwartz][6]</u>的工作。

<a name="Metrics">[__Metrics__](#Metrics)</a>  
__指标__  
Metrics capture a value pertaining to your systems at a specific point in time — for example, the number of users currently logged in to a web application.  
指标捕捉在特定时间点与您的系统相关的值——例如，当前登录到Web应用程序的用户数量。  
Therefore, metrics are usually collected once per second, one per minute, or at another regular interval to monitor a system over time.  
因此，指标通常每秒收集一次，每分钟收集一次，或以其他固定时间间隔收集，以随着时间的推移监控系统。  
There are two important categories of metrics in our framework: work metrics and resource metrics.  
我们的框架中有两类重要的指标：工作指标和资源指标。  
For each system that is part of your software infrastructure, consider which work metrics and resource metrics are reasonably available, and collect them all.  
对于作为软件基础架构一部分的每个系统，考虑哪些工作指标和资源指标是合理可用的，并将它们全部收集起来。  
![alt 示例图片1](https://imgix.datadoghq.com/img/blog/monitoring-101-collecting-data/alerting101-chart-1.png?auto=format&fit=max&w=847)

<a name="Work-metrics">[__Work metrics__](#Work-metrics)</a>  
__工作指标__  
Work metrics indicate the top-level health of your system by measuring its useful output.  
工作指标通过测量系统的有用输出来指示系统的顶级健康状况。  
When considering your work metrics, it’s often helpful to break them down into four subtypes:  
在考虑您的工作指标时，将它们分解为四个子类型通常会有所帮助：
- __throughput__ is the amount of work the system is doing per unit time. Throughput is usually recorded as an absolute number.  
  吞吐量是系统在单位时间内完成的工作量。吞吐量通常记录为绝对数字。
- __success__ metrics represent the percentage of work that was executed successfully.  
  指标表示成功执行的工作百分比。
- __error__ metrics capture the number of erroneous results, usually expressed as a rate of errors per unit time or normalized by the throughput to yield errors per unit of work. Error metrics are often captured separately from success metrics when there are several potential sources of error, some of which are more serious or actionable than others.  
  错误度量捕获错误结果的数量，通常表示为每单位时间的错误率或通过吞吐量标准化以产生每单位工作的错误。当存在多个潜在错误源时，错误度量通常与成功度量分开捕获，其中一些源比其他源更严重或更可操作。  
- __performance__ metrics quantify how efficiently a component is doing its work. The most common performance metric is latency, which represents the time required to complete a unit of work. Latency can be expressed as an average or as a percentile, such as “99% of requests returned within 0.1s”.  
  性能指标量化了一个组件的工作效率。最常见的性能指标是延迟，它表示完成一个工作单元所需的时间。延迟可以表示为平均值或百分比，例如"99%的请求在0.1秒内返回"。

These metrics are incredibly important for observability.   
这些指标对于可观察性非常重要。  
They are the big, broad-stroke measures that can help you quickly answer the most pressing questions about a system’s internal health and performance:  
它们是可以帮助您快速回答有关系统内部健康和性能的最紧迫问题的大型、广泛的措施：  
is the system available and actively doing what it was built to do? How fast is it producing work? What is the quality of that work?  
该系统是否可用并积极执行其构建目的？它生产工作的速度有多快？它工作的质量如何？

Below are example work metrics of all four subtypes for two common kinds of systems: a web server and a data store.  
以下是两种常见系统的所有四种子类型的示例工作指标：Web服务器和数据存储。  
__Example work metrics: Web server (at time 2015-04-24 08:13:01 UTC)__

| __Subtype(子类型)__ | __Description(描述)__ | __Value(值)__ |
| :-----| :---- | :---- |
| throughput(吞吐量) | requests per second(每秒请求数) | 312 |
| success(成功数) | percentage of responses that are 2xx since last measurement(自上次测量以来 2xx 的响应百分比) | 99.1 |
| error(错误数) | percentage of responses that are 5xx since last measurement(自上次测量以来 5xx 的响应百分比) | 0.1 |
| performance(性能) | 90th percentile response time in seconds(90% 响应时间（以秒为单位）) | 0.4 |














[1]:https://www.datadoghq.com/blog/monitoring-101-alerting/
[2]:https://www.datadoghq.com/blog/monitoring-101-investigation/
[3]:https://www.datadoghq.com/customers/
[4]:https://dtdg.co/use-method
[5]:https://dtdg.co/philosophy-alerting
[6]:https://dtdg.co/metrics-attention
