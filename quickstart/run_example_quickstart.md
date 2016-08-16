---
title: "快速起步: 监控维基百科编辑流"
# Top navigation
top-nav-group: quickstart
top-nav-pos: 2
top-nav-title: "例子: 维基百科编辑流"
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

* This will be replaced by the TOC
{:toc}

在这篇指南中，我们将从零开始创建一个Flink项目，然后在一个Flink集群上运行一个流分析程序。

维基百科提供了一个记录了所有词条编辑历史的IRC通道。我们将这个通道读取到Flink中，统计在一个窗口时间内每个用户编辑的字节数。这个程序很容易，以至于在几分钟之内就可以利用Flink实现。但是这个简单的程序却可以给我们建立更加复杂的数据分析程序打下一个很好的基础。

## 创建一个Maven工程

我们使用Maven命令创建我们的项目。更加详细的步骤请查看[Java API Quickstart]({{ site.baseurl }}/quickstart/java_api_quickstart.html)页面。执行下列命令行可以达到我们的目的：

{% highlight bash %}
$ mvn archetype:generate\
    -DarchetypeGroupId=org.apache.flink\
    -DarchetypeArtifactId=flink-quickstart-java\
    -DarchetypeVersion=1.0.0\
    -DgroupId=wiki-edits\
    -DartifactId=wiki-edits\
    -Dversion=0.1\
    -Dpackage=wikiedits\
    -DinteractiveMode=false\
{% endhighlight %}

如果你喜欢，可以修改参数`groupId`, `artifactId` 和 `package`。使用上面命令行中的参数，Maven会创建一个结构如下的项目：


{% highlight bash %}
$ tree wiki-edits
wiki-edits/
├── pom.xml
└── src
    └── main
        ├── java
        │   └── wikiedits
        │       ├── Job.java
        │       ├── SocketTextStreamWordCount.java
        │       └── WordCount.java
        └── resources
            └── log4j.properties
{% endhighlight %}


在根目录下有一个`pom.xml`文件，文件中包含了Flink依赖。同时在文件夹`src/main/java`下有几个Flink示例程序。因为我们是从零开始创建项目，可以删除这些示例程序。

{% highlight bash %}
$ rm wiki-edits/src/main/java/wikiedits/*.java
{% endhighlight %}


最后，项目中用到了Flink Wikipedia connector，我们需要把它作为依赖添加进来到项目总。编辑`pom.xml`文件的`dependencies` 模块，结果如下：

{% highlight xml %}
<dependencies>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-java</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-java_2.10</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-clients_2.10</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-connector-wikiedits_2.10</artifactId>
        <version>${flink.version}</version>
    </dependency>
</dependencies>
{% endhighlight %}

注意`flink-connector-wikiedits_2.10`依赖也添加了进来。（此例子和Wikipedia connector受到了Apache Samza
示例*Hello Samza*的启发）

## 编写Flink程序

是时候着手写代码了。打开你最喜欢的IDE并导入这个Maven项目，或者打开文字编辑，然后创建文件`src/main/java/wikiedits/WikipediaAnalysis.java`:

{% highlight java %}
package wikiedits;

public class WikipediaAnalysis {

    public static void main(String[] args) throws Exception {

    }
}
{% endhighlight %}

目前为止，我们只有一个空框架，随着接下来我们将一步步填充这个框架。注意，因为IDEs能够自动的添加import声明，所以我就不再提供。如果你想跳过这些步骤，直接在编辑器中输入代码，在这个章节的最后，我会展示带有import声明的完整代码。

在一个Flink程序中的第一步是创建一个`StreamExecutionEnvironment`（或者`ExecutionEnvironment`，如果你想编写批处理程序）。它可以用来设置执行参数和创建数据源从外部系统读取数据。现在，让我们添加到main方法里：

{% highlight java %}
StreamExecutionEnvironment see = StreamExecutionEnvironment.getExecutionEnvironment();
{% endhighlight %}

接下来我们将创建一个数据源来从维基百科IRC日志中读取数据

{% highlight java %}
DataStream<WikipediaEditEvent> edits = see.addSource(new WikipediaEditsSource());
{% endhighlight %}

这行代码创建一个由 `WikipediaEditEvent` 组成的`DataStream`。在这个例子中，我们感兴趣的是统计在一个窗口时间内每个用户添加和删除词条的字节数，在这我们设定窗口时常为5秒。首先必须指出我们在这把用户名设置成流的键，也就是说在这个流上的操作需要将键考虑进去。在我们这个用例中，需要统计的是每个特定的用户在窗口时间内所编辑的字节数总和。在一个流上设置键，我们需要提供一个`KeySelector`，如下所示：

{% highlight java %}
KeyedStream<WikipediaEditEvent, String> keyedEdits = edits
    .keyBy(new KeySelector<WikipediaEditEvent, String>() {
        @Override
        public String getKey(WikipediaEditEvent event) {
            return event.getUser();
        }
    });
{% endhighlight %}

This gives us a Stream of WikipediaEditEvent that has a String key, the user name.
上面的代码，让我们得到一个由`WikipediaEditEvent`组成的Stream，这个Stream拥有一个`String`类型的键（用户名）。现在我们可以在这个流上添加窗口，并根据这些窗口计算结果。每个窗口表示流上的一个切割片段，计算就应用在这些切片上。当需要在一个拥有无限数据元素的Stream上执行聚合操作的时候，窗口是必须的。在我们的例子中，我们想计算每5秒钟每个用户的总编辑字节数：

{% highlight java %}
DataStream<Tuple2<String, Long>> result = keyedEdits
    .timeWindow(Time.seconds(5))
    .fold(new Tuple2<>("", 0L), new FoldFunction<WikipediaEditEvent, Tuple2<String, Long>>() {
        @Override
        public Tuple2<String, Long> fold(Tuple2<String, Long> acc, WikipediaEditEvent event) {
            acc.f0 = event.getUser();
            acc.f1 += event.getByteDiff();
            return acc;
        }
    });
{% endhighlight %}


第一个调用语句，`.window()`，表明我们想要一个5s的翻滚窗口(非重叠)。第二个调用表明一个应用于每个独特的键的各个窗口切片之上的*Fold transformation*操作。在我们这个示例中，我们从一个初始化值`("", 0L)` 开始，然后加上一个用户在特定窗口时间内每次撰写的词条字节数。最后得到的是一个由`Tuple2<String, Long>`组成的流，每个`Tuple2<String, Long>`对应一个用户，且每5s为每个用户生成一个新的Tuple。


现在唯一剩下的事就是将结果Stream打印到控制台上并开始执行设定的操作：
{% highlight java %}
result.print();

see.execute();
{% endhighlight %}

为了启动Flink任务，最后一行代码是必须要有的。所有的操作，例如 creating sources, transformations 和 sinks
仅仅是构建了一个由内部操作组成的图。只有当`execute()`调用的时候，这个有内部操作组成的图才会被提交到一个集群，或
在你的本机上执行。


到目前为止，完整的代码如下：
{% highlight java %}
package wikiedits;

import org.apache.flink.api.common.functions.FoldFunction;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.connectors.wikiedits.WikipediaEditEvent;
import org.apache.flink.streaming.connectors.wikiedits.WikipediaEditsSource;

public class WikipediaAnalysis {

  public static void main(String[] args) throws Exception {

    StreamExecutionEnvironment see = StreamExecutionEnvironment.getExecutionEnvironment();

    DataStream<WikipediaEditEvent> edits = see.addSource(new WikipediaEditsSource());

    KeyedStream<WikipediaEditEvent, String> keyedEdits = edits
      .keyBy(new KeySelector<WikipediaEditEvent, String>() {
        @Override
        public String getKey(WikipediaEditEvent event) {
          return event.getUser();
        }
      });

    DataStream<Tuple2<String, Long>> result = keyedEdits
      .timeWindow(Time.seconds(5))
      .fold(new Tuple2<>("", 0L), new FoldFunction<WikipediaEditEvent, Tuple2<String, Long>>() {
        @Override
        public Tuple2<String, Long> fold(Tuple2<String, Long> acc, WikipediaEditEvent event) {
          acc.f0 = event.getUser();
          acc.f1 += event.getByteDiff();
          return acc;
        }
      });

    result.print();

    see.execute();
  }
}
{% endhighlight %}

你可以用Maven命令在你的IDE中或者命令行中运行这个例子：

{% highlight bash %}
$ mvn clean package
$ mvn exec:java -Dexec.mainClass=wikiedits.WikipediaAnalysis
{% endhighlight %}

第一个命令build我们的项目，第二个命令执行Main类。输出应该如下所示：

{% highlight bash %}
1> (Fenix down,114)
6> (AnomieBOT,155)
8> (BD2412bot,-3690)
7> (IgnorantArmies,49)
3> (Ckh3111,69)
5> (Slade360,0)
7> (Narutolovehinata5,2195)
6> (Vuyisa2001,79)
4> (Ms Sarah Welch,269)
4> (KasparBot,-245)
{% endhighlight %}

在每行开始的数字告诉我们这行输出是哪一个并行的实例产生的。

这些应该已经可以帮助你开始编写自己的Flink程序了。如果你想进一步学习Flink和Kafka，你可以查看我们的关于[基本概念]({{ site.baseurl }}/apis/common/index.html)和[DataStream API]({{ site.baseurl }}/apis/streaming/index.html)的指南。如果你想学习如何搭建在本地搭建一个Flink集群，并将结果写入[Kafka](http://kafka.apache.org)，请继续下面的额外练习。


## 额外练习: 在集群上运行程序并将结果写到Kafka

在我们开始之前，请跟着我们[快速起步：安装](setup_quickstart.html)的步骤搭建一个Flink集群，参考[Kafka quickstart]
(https://kafka.apache.org/documentation.html#quickstart)安装Kafka。

作为第一步，为了使用Kafka sink, 我们必须先将Flink Kafka connector作为一个依赖添加到pop.xml的dependencies section:
{% highlight xml %}
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.8_2.10</artifactId>
    <version>${flink.version}</version>
</dependency>
{% endhighlight %}


接下来我们需要修改我们的程序。我们将`print()` sink替换成Kafka sink。更新后的代码如下：
{% highlight java %}

result
    .map(new MapFunction<Tuple2<String,Long>, String>() {
        @Override
        public String map(Tuple2<String, Long> tuple) {
            return tuple.toString();
        }
    })
    .addSink(new FlinkKafkaProducer08<>("localhost:9092", "wiki-result", new SimpleStringSchema()));
{% endhighlight %}


注意我们是怎样利用一个MapFunction把`Tuple2<String, Long>`流转换成`String`流的。这样做的原因是向Kafka中写入明文
字符串相对要容易一些。然后我们创建了一个Kafka sink。或许你还需要适应你设置的主机名称和端口。在我们开始运行我们的程序
之前，我们还需要创建一个名称为`"wiki-result"`的Kafka 流。因为在集群上运行这个程序需要jar包，所以我们用Maven来打包
这个项目：


{% highlight bash %}
$ mvn clean package
{% endhighlight %}

得到的jar文件将会位于子文件夹`target`下：`target/wiki-edits-0.1.jar`。 下面我们将会用到这个文件。

现在我们已经准备好启动一个Flink集群，并在该集群上执行一个向Kafka写数据的程序。到我们安装Flink的目录下，并启动一个
本地集群：

{% highlight bash %}
$ cd my/flink/directory
$ bin/start-local.sh
{% endhighlight %}

我们还必须创建一个Kafka主题，这样我们的程序才可以向这个Kafka主题写数据：

{% highlight bash %}
$ cd my/kafka/directory
$ bin/kafka-topics.sh --create --zookeeper localhost:2181 --topic wiki-results
{% endhighlight %}

现在我们可以在本地Flink集群上运行我们的jar文件了：
{% highlight bash %}
$ cd my/flink/directory
$ bin/flink run -c wikiedits.WikipediaAnalysis path/to/wikiedits-0.1.jar
{% endhighlight %}

如果一切按照我们的计划顺利进行，这个命令的输出应该类似于这样：

```
03/08/2016 15:09:27 Job execution switched to status RUNNING.
03/08/2016 15:09:27 Source: Custom Source(1/1) switched to SCHEDULED
03/08/2016 15:09:27 Source: Custom Source(1/1) switched to DEPLOYING
03/08/2016 15:09:27 TriggerWindow(TumblingProcessingTimeWindows(5000), FoldingStateDescriptor{name=window-contents, defaultValue=(,0), serializer=null}, ProcessingTimeTrigger(), WindowedStream.fold(WindowedStream.java:207)) -> Map -> Sink: Unnamed(1/1) switched to SCHEDULED
03/08/2016 15:09:27 TriggerWindow(TumblingProcessingTimeWindows(5000), FoldingStateDescriptor{name=window-contents, defaultValue=(,0), serializer=null}, ProcessingTimeTrigger(), WindowedStream.fold(WindowedStream.java:207)) -> Map -> Sink: Unnamed(1/1) switched to DEPLOYING
03/08/2016 15:09:27 TriggerWindow(TumblingProcessingTimeWindows(5000), FoldingStateDescriptor{name=window-contents, defaultValue=(,0), serializer=null}, ProcessingTimeTrigger(), WindowedStream.fold(WindowedStream.java:207)) -> Map -> Sink: Unnamed(1/1) switched to RUNNING
03/08/2016 15:09:27 Source: Custom Source(1/1) switched to RUNNING
```

你可以看到各个独立的操作是怎样开始运行的。在这里只有两个，因为处于性能方面的考虑窗口后的这些操作被合并成一个操作了。在Flink中，你可以把它叫做*chaining*。

你可以通过Kafka 控制台消费者检查Kafka主题，进而检查程序的输出。

{% highlight bash %}
bin/kafka-console-consumer.sh  --zookeeper localhost:2181 --topic wiki-result
{% endhighlight %}


你也可以检查运行在[http://localhost:8081](http://localhost:8081)的Flink 仪表盘。从那你可以获得
你的集群资源和正在运行的任务的总体概述：

<a href="{{ site.baseurl }}/page/img/quickstart-example/jobmanager-overview.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-example/jobmanager-overview.png" alt="JobManager Overview"/></a>

如果你单击正在运行的任务，你将会得到一个可以检查各个独立操作的详细情况的视图。比如，查看已经处理的单位数据总数的视图：
<a href="{{ site.baseurl }}/page/img/quickstart-example/jobmanager-job.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-example/jobmanager-job.png" alt="Example Job View"/></a>

这些仅仅包含了我们学习Flink的一点心得。 如果你有任何疑问，咨询我们的 [Mailing Lists](http://flink.apache.org/community.html#mailing-lists)。我们很高兴可以提供帮助。
