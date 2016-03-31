---
layout: post
comments: true
title: Apache Kafka Producer Benchmarks - Java vs Jython vs Python
---
*v1.0*

*Note: This post is open to suggestions that can help achieve fairer results with these benchmarks. It is a versioned post and I would be incrementing the version if anything related to these benchmarks changes. Changes can be tracked [here](https://github.com/mrafayaleem/mrafayaleem.github.io) as well.*

In my [previous](http://mrafayaleem.com/2016/03/19/interfacing-jython-with-kafka/) post, I wrote about how we can interface Jython with Kafka 0.8.x and use Java consumer clients directly with Python code. As a followup to that, I got curious about what would be the performance difference between Java, Jython and Python clients. In this post, I am publishing some of the benchmarks that I have been doing with Java, Jython and Python *producers*. 

**Acknowledgement:** I was further inspired by [Jay Kreps's](https://twitter.com/jaykreps) amazing [blog post](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines) on Kafka 0.8.1 benchmarks and a lot of context around my own benchmakrs have been borrowed from his post. It would be a good idea to read his post first before going through the rest of it here. 

All of these benchmarks were run on AWS EC2 instances with the following specifications:

- Three `m4.large` **shared** tenancy Kafka broker servers as a cluster with 128 GiB EBS Magnetic storage each.
- One `m4.xlarge` **shared** tenancy Zookeeper server for the whole Kafka cluster with 128 GiB EBS Magnetic storage.

I am not aware of any reasons where a Zookeeper cluster would perform significantly better than a single Zookeper server particularly for this configuration, so I went for a single server. For these benchmarks, Zookeer server was also used to run the producers for each language choice.

I kept Kafka and Zookeeper server configurations to mostly the default ones defined in `server.properties` and `zookeeper.properties` respectively. In particular, none of the server configurations were tweeked for these benchmarks.

You can see the server configurations [here](https://github.com/mrafayaleem/kafka-jython/tree/master/config).

For each of these benchmarks, message size was kept small at 100 bytes for exactly the same reason as cited by Jay Kreps.

Following are the build versions for the tools used for running these benchmarks:

- Kafka: `0.9.0.1`
- Java: `Java(TM) SE Runtime Environment (build 1.7.0_80-b15)`
- Jython: `2.7.0`
- Python `2.7.6`
- kafka-python: `1.0.2`
- Python futures module: `3.0.5`

I divided the benchmarks into three different cases with each case recording results for all the runs. Before going into the results, lets be clear about the following:

- **Case**: Case is the configuration for running the benchmarks. For example, producing on a topic with no replication is a case.
- **Run**: The choice of language for running a benchmark is a run. For example, a Java producer benchmark is a run.
- **Pure Java**: Java code running official Apacha Kafka producer API. See [implementation](https://github.com/mrafayaleem/kafka-jython/blob/master/benchmarks/src/main/java/kafkajython/benchmarks/ProducerPerformance.java).
- **Java interfaced Jython**: Jython code is just used to call producer code. The actual implementation is still in Java. See [implementation](https://github.com/mrafayaleem/kafka-jython/blob/master/benchmarks/src/main/jython/java_interfaced_jython_prodcuer.py).
- **Pure Jython**: Where possible, everything is written in Python code using Python standard libraries, except the actual internal producer implementation which is imported from official Kafka libraries. See [implementation](https://github.com/mrafayaleem/kafka-jython/blob/master/benchmarks/src/main/jython/producer_performance.py).
- **Pure Python**: Pure Python code that uses `kafka-python` client for Kafka. See [implementation](https://github.com/mrafayaleem/kafka-jython/blob/master/benchmarks/src/main/python/producer_performance.py).

These benchmarks were run for the following cases:

1. Single producer (single thread) producing on a topic with no replication and 6 partitions.
2. Single producer (single thread) producing asynchronousyly on a topic with replication factor of 3 and 6 partitions.
3. Single producer (single thread) producing syncronously on a topic with replication factor of 3 and 6 partitions.

In this context, a producer running in **async** mode would mean that the number of acknowledgments the producer requires the leader to have received from its followers before considering a request complete is **0**. **Sync** mode is where the leader would wait on all the followers to acknowledge before acknowledging it to the the producer.

## Single producer producing on a topic with no replication and 6 partitions.
Results are tabulated below for this case. I was shocked to see such a stark difference between `kafka-python` and official Java producer clients. And as expected, *Java* and *Java interfaced  Jython* producers showed very similar results.


**Results:**

 | No. of records sent | records/sec | MB/sec | avg latency (ms) | max latency (ms) | 50th (ms) | 95th (ms) | 99th (ms) | 99.9th (ms)
--- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
Pure Java | 50000000 | 747417.671943 | 71.28 | 429.34 | 2828.00 | 265 | 1251 | 1817 | 2497
Java interfaced Jython | 50000000 | 753046.071359 | 71.82 | 310.54 | 2189.00 | 1 | 1416 | 1798 | 1798
Pure Jython | 50000000 | 117115.217951 | 11.1689775421 | 53.95983326 | 4088.0 | 2 | 328 | 1108 | 3262
Pure Python | 50000000 | 9707.5761892 | 0.92578660862 | 25.82151426 | 1334.0 | 26 | 44 | 54 | 226

## Single producer producing asynchronousyly on a topic with replication factor of 3 and 6 partitions.

Not surprisingly, *Java* and *Java interfaced Jython* producers fared similarly here as well. It is also interesting to note that results for *Pure Jython* and *Pure Python* producers are not much different from the previous respective runs. My assumption is that in this case, they are mostly limited by the speed of code execution rather than producing to a topic with 3x replication. What surprises me is the fact that Java producer in this case produced half as many records/sec compared to the run in previous case, although it was run in async mode. This in contrast with the results for Kafka 0.8.1 in Jay Kreps's post.

**Results:**

 | No. of records sent | records/sec | MB/sec | avg latency (ms) | max latency (ms) | 50th (ms) | 95th (ms) | 99th (ms) | 99.9th (ms)
--- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
Pure Java | 50000000 | 427051.126561 | 40.73 | 1160.60 | 7428.00 | 459 | 4282 | 6288 | 7266
Java interfaced Jython | 50000000 | 446791.589595 | 42.61 | 1032.90 | 5925.00 | 219 | 4602 | 5595 | 5852
Pure Jython | 50000000 | 118026.409589 | 11.2558755483 | 231.39137682 | 5179.0 | 6 | 982 | 1983 | 4590
Pure Python | 50000000 | 9845.42251444 | 0.938932658619 | 26.58918002 | 1138.0 | 25 | 44 | 59 | 539

## Single producer producing syncronously on a topic with replication factor of 3 and 6 partitions.
*For this case, I increased the `batch.size` from `8196` to `64000` for each run to accomodate for sync mode.* 

Again, *Pure Python* and *Pure Jython* producers showed similar throughput, although max latencies are slightly higher than the previous run. It does seem like throughput in this case is mostly limited by code execution or `kafka-python` producer client implementation. Throughput for *Pure Java* and *Java interfaced Jython* has almost halved compared to the previous case which makes sense here as we ran these producers in sync mode.

**Results:**

 | No. of records sent | records/sec | MB/sec | avg latency (ms) | max latency (ms) | 50th (ms) | 95th (ms) | 99th (ms) | 99.9th (ms)
--- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
Pure Java | 50000000 | 205475.511429 | 19.60 | 2502.79 | 11165.00 | 1769 | 6741 | 9492 | 10640
Java interfaced Jython | 50000000 | 192588.426976 | 18.37 | 2659.39 | 19267.00 | 751 | 9236 | 14518 | 18843
Pure Jython | 50000000 | 110700.535126 | 10.5572257162 | 1220.20796466 | 7115.0 | 861 | 4770 | 6052 | 6747
Pure Python | 49999231 | 9844.45746764 | 0.938840624584 | 39.2650510365 | 2359.0 | 28 | 57 | 452 | 1108


You can find everything related to these benchmarks [here](https://github.com/mrafayaleem/kafka-jython/tree/master/benchmarks). I would love to hear some feedback on this post while I am working on publishing results for consumer benchmarks in my upcoming post.

---
