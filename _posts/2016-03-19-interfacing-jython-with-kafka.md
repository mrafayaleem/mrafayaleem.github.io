---
layout: post
comments: true
title: Interfacing Jython with Kafka 0.8.x
---
With the release of Kafka 0.9.0, the consumer API was redesigned to remove the dependency between consumer and Zookeeper. Prior to the 0.9.0 release, Kafka consumer was depenedent on Zookeeper for storing its offsets and the complex rebalancing logic was built right into the "high level" consumer. This was causing a couple of issues which are discussed [here](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Detailed+Consumer+Coordinator+Design). Due to the issues invloving complex rebalance algoritihm within the consumer, some Kafka clients had no support for "coordinated consumption". Essentially, it means that without "coordinated consumption", you cannot have N consumers within the same consumer group consuming on the same topic which defeats one of the purpose of Kafka; to balance load across multiple consumers. I saw this problem while using `kafka-python 0.9.2` client library with `Kafka 0.8.x`. While I was trying to setup multiple consumers within the same consumer group, I noticed that every message to that topic would be reproduced on each and every consumer within that same group, defeating the purpose of Kafka being used as a traditional queing system with a pool of consumers. The issue has been discussed on [Github](https://github.com/dpkp/kafka-python/issues/199#issuecomment-53052761). 

One of the solution that I managed to figure out without moving away from Python was to use Jython for writing consumers for Kafka 0.8.x.

In this post, I would discuss about what you can do with Jython to resolve the Kafka 0.8.x issue and why it actually works. In a more traditional sense, this post would also work as a tutorial for interfacing Jython with Java.

## Setup

You need to have the following installed on your system. The tutorial assumes that you are familiar with these tools.

* Java SDK
* Python
* `virtualenv` and `virtualenvwrapper`
* Kafka 0.9.x or Kafka 0.8.x (assuming you know how to setup and run Kafka)

I have setup a bare-bones repositiory for working with this tutorial [here](https://github.com/mrafayaleem/kafka-jython/). To keep things at a minimum, `Kafka 0.9.1` binaries and `Jython 2.7.0` installer  are included within this repo. This means that you can directly run Kafa and Zookeeper after cloning this.

#### Setting up a virtualenv for Jython
Jython is fully compatible with `virtualenv` and tools such as `pip` and setting a virtualenv with Jython as the interpretter is pretty straightforward.

Jython can be installed using the GUI or console. For GUI, execute the jar and follow the steps. For installing it via console, you can use the following command to start with:

```shell
java -jar jython_installer-2.7.0.jar --console
```
Make a note of the location where you have installed Jython.

`cd` into the directory where you cloned the repo and create a virtualenv using Jython as your interpretter by using the following:

```shell
mkvirtualenv -p /jython-installation-path/jython2.7.0/bin/jython -a kafka-jython kafka-jython
```
You should be in the repo directory right now with your `virtualenv` already activated.

The project layout as follows:

```
.
├── bin
│   └── windows
├── build
├── config
├── examples
│   └── src
│       └── main
│           ├── java
│           │   └── kafkajython
│           └── python
│               └── consumers
├── libs
└── requirements
```
* **bin:** Contains helper scripts from Kafka and other binaries.
* **build:** This is where your compiled files would go.
* **config:** Various kafka configs.
* **examples:** Jython and Java code for this tutorial.
* **libs:** Kafka jars which we will use as dependencies.
* **requirements:** Python library dependencies.

#### Installing Python dependencies

Once in the repo directory, install all Python dependencies using:

```shell
pip install -r requirements/development.txt
```

#### Compiling source code

Since one of our examples depends on calling Java class directly from Jython, we need to compile it first using:

```shell
javac -cp ".:/your-directory/kafka-jython/libs/*" -d build examples/src/main/java/kafkajython/Consumer*
```

We tell java compiler to include all the dependencies in the `lib` directory while compiling and put the compiled files in the build directory.

## A bit about interfacing Jython with Java

Essentially, there are two ways that you can write consumers for this case.  

1. Write everything in Java and call it directly from Jython.
2. Write everything in Jython by importing from Java standard library and Kafka directly in your source code.

We would cover both in a while.

#### 1 - Write everything in Java and call it directly from Jython

Lets look at some of the code for the consumers. This "high level" consumer example has been borrowed directly from [here](https://cwiki.apache.org/confluence/display/KAFKA/Consumer+Group+Example) so do check it out for a more elaborate explanation.

`ConsumerTest` class is a runnable that consumes messages from Kafka stream and waits (blocks) for new messages. `ConsumerGroupExample` is the entry point where we specify the number of concurrent consumers (within the same consumer group) to use when consuming a topic.

The `main` method on `ConsumerGroupExample` accepts an array of strings. In later part, you would see that we actually pass the array of strings through Jython.

**ConsumerGroupExample.java**

```java
package kafkajython;

import kafka.consumer.ConsumerConfig;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ConsumerGroupExample {
    private final ConsumerConnector consumer;
    private final String topic;
    private  ExecutorService executor;

    public ConsumerGroupExample(String a_zookeeper, String a_groupId, String a_topic) {
        consumer = kafka.consumer.Consumer.createJavaConsumerConnector(
                createConsumerConfig(a_zookeeper, a_groupId));
        this.topic = a_topic;
    }

    public void shutdown() {
        if (consumer != null) consumer.shutdown();
        if (executor != null) executor.shutdown();
        try {
            if (!executor.awaitTermination(5000, TimeUnit.MILLISECONDS)) {
                System.out.println("Timed out waiting for consumer threads to shut down, exiting uncleanly");
            }
        } catch (InterruptedException e) {
            System.out.println("Interrupted during shutdown, exiting uncleanly");
        }
    }

    public void run(int a_numThreads) {
        Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
        topicCountMap.put(topic, new Integer(a_numThreads));
        Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCountMap);
        List<KafkaStream<byte[], byte[]>> streams = consumerMap.get(topic);

        // now launch all the threads
        //
        executor = Executors.newFixedThreadPool(a_numThreads);

        // now create an object to consume the messages
        //
        int threadNumber = 0;
        for (final KafkaStream stream : streams) {
            executor.submit(new ConsumerTest(stream, threadNumber));
            threadNumber++;
        }
    }

    private static ConsumerConfig createConsumerConfig(String a_zookeeper, String a_groupId) {
        Properties props = new Properties();
        props.put("zookeeper.connect", a_zookeeper);
        props.put("group.id", a_groupId);
        props.put("zookeeper.session.timeout.ms", "400");
        props.put("zookeeper.sync.time.ms", "200");
        props.put("auto.commit.interval.ms", "1000");

        return new ConsumerConfig(props);
    }

    public static void main(String[] args) {
        String zooKeeper = args[0];
        String groupId = args[1];
        String topic = args[2];
        int threads = Integer.parseInt(args[3]);

        ConsumerGroupExample example = new ConsumerGroupExample(zooKeeper, groupId, topic);
        example.run(threads);

        try {
            Thread.sleep(90000);
        } catch (InterruptedException ie) {

        }
        example.shutdown();
    }
}
```

**ConsumerTest.java**

```java
package kafkajython;

import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;

public class ConsumerTest implements Runnable {
    private KafkaStream m_stream;
    private int m_threadNumber;

    public ConsumerTest(KafkaStream a_stream, int a_threadNumber) {
        m_threadNumber = a_threadNumber;
        m_stream = a_stream;
    }

    public void run() {
        ConsumerIterator<byte[], byte[]> it = m_stream.iterator();
        while (it.hasNext())
            System.out.println("Thread " + m_threadNumber + ": " + new String(it.next().message()));
        System.out.println("Shutting down Thread: " + m_threadNumber);
    }
}
```

Execute this "high level" java example using Jython:

```shell
jython -J-cp "/your-directory/Projects/kafka-jython/libs/*:/your-directory/Projects/kafka-jython/build:." examples/src/main/python/java_interfaced_jython_consumer.py
```
Note that we need to tell Jython about both dependencies; Kafka jars and `.class` files that we compiled earlier in this tutorial.

Now, when you produce messages on Kafka, you should see them printing in your shell.

Lets go through the Jython code in `java_interfaced_jython_consumer.py` now.

**java\_interfaced\_jython_consumer.py**

```python
from kafkajython import ConsumerGroupExample


def run():
    # List of arguments for initializing the consumer
    args = [
        "localhost:2181",
        "test-group",
        "another-replicated-topic",
        "3"
    ]
    ConsumerGroupExample.main(args)

if __name__ == '__main__':
    run()

```


Notice how the Java classes that we compiled earlier can be directly imported here. Since those Java classes were packaged in `kafkajython` namespace, it is necessary to do the import from the same package.

In the run function here, we pass a list of strings to the main method of ConsumerGroupExample that is written in Java.

Lets see how we can write this example purely in Jython.

#### 2 - Write everything in Jython by importing from Java standard library and Kafka directly in your source code


Following is the code for "high level" consumer written entirely in Jython:

**group.py**

```python
from concurrent.futures import ThreadPoolExecutor

from java.util import Properties
from java.util import HashMap
from java.lang import String

from kafka.consumer import ConsumerConfig
from kafka.consumer import Consumer


class HighLevelConsumer(object):

    def __init__(self, zookeeper, group_id, topic, thread_count=1, callback=lambda x, y: (x, y)):
        self.consumer = Consumer.createJavaConsumerConnector(
            self._create_consumer_config(zookeeper, group_id)
        )
        self.topic = topic
        self.thread_count = thread_count
        self.callback = callback

    def consume(self):
        topic_count_map = HashMap()
        topic_count_map.put(self.topic, self.thread_count)
        consumer_map = self.consumer.createMessageStreams(topic_count_map)
        streams = consumer_map.get(self.topic)

        with ThreadPoolExecutor(max_workers=self.thread_count) as executor:
            futures = []
            for i, stream in enumerate(streams):
                futures.append(executor.submit(self._decorate(self.callback, i, stream)))

            for future in futures:
                future.result()

    @staticmethod
    def _decorate(callback, thread, stream):
        def decorated():
            it = stream.iterator()
            while it.hasNext():
                callback(thread, String(it.next().message()))

        return decorated

    @staticmethod
    def _create_consumer_config(zookeeper, group_id):
        props = Properties()
        props.put("zookeeper.connect", zookeeper)
        props.put("group.id", group_id)
        props.put("zookeeper.session.timeout.ms", "400")
        props.put("zookeeper.sync.time.ms", "200")
        props.put("auto.commit.interval.ms", "1000")

        return ConsumerConfig(props)

```

You can see how trivial it is to use libraries and other Java language constructs in Jython. In fact, the code doesn't look any different than traditional Python. However, notice the explicit Java `String` import: `from java.lang import String`. In Java, you never need to do this because `java.lang` is auto-imported.

Following is the entry point for the Jython consumer.

**pure\_jython\_consumer.py**

```python
from consumers.group import HighLevelConsumer


def process_message(thread, message):
    print str(thread) + ': ' + str(message)


def run():
    consumer = HighLevelConsumer(
            'localhost:2181', 'unknown', 'another-replicated-topic', 3, callback=process_message)
    consumer.consume()

if __name__ == '__main__':
    run()
```

The Jython implementation is more or less the same as the one written in Java. One difference is that I have used a backport of Python 3.2 futures package to create thread pool instead of using concurrent utils from Java. The other difference is that the Jython consumer allows you to pass functions as callbacks so you can manipulate the incoming messages.

Lets run this using:

```shell
jython -J-cp "/Users/rafay/Projects/kafka-jython/libs/*" examples/src/main/python/pure_jython_consumer.py
```
You should notice incoming messages from the Kafka producer.

**Important note:** *One very important mention in context of this post is that Jython threads are always mapped to Java threads. Jython actually lacks the global interpreter lock (GIL), which is an implementation detail of CPython. This means that Jython can actually give you better performance on mult-threaded compute-intensive tasks written in Python. You can read more about it [here](http://www.jython.org/jythonbook/en/1.0/Concurrency.html#no-global-interpreter-lock).*


### Conslusion

I think using Jython for coordinated consumption with Kafka 0.8.x is a good idea when:

* You cannot move away from Python because of library dependencies.
* Your Kafka infrastructure cannot migrate to Kafka 0.9.x (which is a requirement if you want to use new Kafka consumer clients).
* You have updated your Kafka infrastrcutre to 0.9.x but for some reason, you cannot update your consumers to use the new Kafka consumer client.

In the long run, it would be better to just update your Kafka infrastructure to 0.9.x even when it might cause downtime. You would definitely get better support and more features; such as the fact that latest version of `kafka-python` implements the new Kafka consumer client which supports coordinated consumers.

Conclusively, Jython has worked well for this problem. However, I am not aware of how well it would perform	 in a huge scale production environment with several consumers, consumer groups, topics and partitions.

*Note: This post might contain some edits which can be tracked [here](https://github.com/mrafayaleem/mrafayaleem.github.io).*

---
