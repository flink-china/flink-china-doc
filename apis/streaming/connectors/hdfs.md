---
title: "HDFS Connector"

# Sub-level navigation
sub-nav-group: streaming
sub-nav-parent: connectors
sub-nav-pos: 3
sub-nav-title: HDFS
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

This connector provides a Sink that writes rolling files to any filesystem supported by
Hadoop FileSystem. To use this connector, add the
following dependency to your project:
这个连接器提供了一种接收器(Sink，以下统称为 Sink)可以将回滚文件(下文中统称为 rolling,例如log4j DailyRollingFileAppender)写入到任意文件系统，包括Hadoop FileSystem(Hadoop平台文件系统: Hdfs)。使用这个连接器需要在工程中加入以下依赖。

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-filesystem{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version}}</version>
</dependency>
{% endhighlight %}

Note that the streaming connectors are currently not part of the binary
distribution. See
[here]({{site.baseurl}}/apis/cluster_execution.html#linking-with-modules-not-contained-in-the-binary-distribution)
for information about how to package the program with the libraries for
cluster execution.
注意：目前 streaming 的连接器不是二进制发行包的一部分。关于如何打包程序和依赖库，并在集群执行，请参考 [这里]({{site.baseurl}}/apis/cluster_execution.html#linking-with-modules-not-contained-in-the-binary-distribution)

#### Rolling File Sink

The rolling behaviour as well as the writing can be configured but we will get to that later.
This is how you can create a default rolling sink:
rolling 的行为包括写文件可以在这里配置，稍后再做介绍。
以下代码介绍了如何创建一个默认的 rolling sink：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<String> input = ...;

input.addSink(new RollingSink<String>("/base/path"));

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[String] = ...

input.addSink(new RollingSink("/base/path"))

{% endhighlight %}
</div>
</div>

The only required parameter is the base path where the rolling files (buckets) will be
stored. The sink can be configured by specifying a custom bucketer, writer and batch size.
使用 rolling sink 唯一的配置是：rolling 文件（buckets：桶）所在的存储目录。这种 sink 可以指定用户 bucketer ，写以及批量大小。

By default the rolling sink will use the pattern `"yyyy-MM-dd--HH"` to name the rolling buckets.
This pattern is passed to `SimpleDateFormat` with the current system time to form a bucket path. A
new bucket will be created whenever the bucket path changes. For example, if you have a pattern
that contains minutes as the finest granularity you will get a new bucket every minute.
Each bucket is itself a directory that contains several part files: Each parallel instance
of the sink will create its own part file and when part files get too big the sink will also
create a new part file next to the others. To specify a custom bucketer use `setBucketer()`
on a `RollingSink`.
默认的 rolling sink 将会使用 `"yyyy-MM-dd--HH"` 的格式命名 rolling buckets 。
这种模式是通过 `SimpleDateFormat` 类获取当前系统时间形成 bucket 的路径。
当 bucket 的路径改变时会创建新的 bucket 。举例来说，如果使用的是一分钟为最小粒度的模式，那么每分钟都会创建一个新的 bucket 。
每个 bucket 本身是一个目录，这个目录包含几个部分的文件：sink 的每个并行实例都会创建自己的文件，并且当文件很大时，sink 会在该文件同目录下创建一个新文件代替原来的文件。
`RollingSink` 上可以使用 `setBucketer()` 方法指定用户的 bucketer 。

The default writer is `StringWriter`. This will call `toString()` on the incoming elements
and write them to part files, separated by newline. To specify a custom writer use `setWriter()`
on a `RollingSink`. If you want to write Hadoop SequenceFiles you can use the provided
`SequenceFileWriter` which can also be configured to use compression.
默认的输出类是 `StringWriter` 。它会在传入参数上调用 `toString()` 方法将他们写入文件中，按换行符进行分隔。用户可以在 `RollingSink` 中使用 `setWriter()` 方法自定义输出类。
当要写 Hadoop 的分布式存储系统中时，可以使用 `SequenceFileWriter` 来实现，它同时可以配置使用压缩方式。

The last configuration option is the batch size. This specifies when a part file should be closed
and a new one started. (The default part file size is 384 MB).
最后一个配置操作是批量操作。这个配置指定了一个文件达到多大的数据量时需要被关闭同时打开一个文件新文件。 （默认的文件大小是384 MB）。

Example:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<IntWritable,Text>> input = ...;

RollingSink sink = new RollingSink<String>("/base/path");
sink.setBucketer(new DateTimeBucketer("yyyy-MM-dd--HHmm"));
sink.setWriter(new SequenceFileWriter<IntWritable, Text>());
sink.setBatchSize(1024 * 1024 * 400); // this is 400 MB,

input.addSink(sink);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[Tuple2[IntWritable, Text]] = ...

val sink = new RollingSink[String]("/base/path")
sink.setBucketer(new DateTimeBucketer("yyyy-MM-dd--HHmm"))
sink.setWriter(new SequenceFileWriter[IntWritable, Text]())
sink.setBatchSize(1024 * 1024 * 400) // this is 400 MB,

input.addSink(sink)

{% endhighlight %}
</div>
</div>

This will create a sink that writes to bucket files that follow this schema:
按照以下约束可以创建一个写 bucket 文件的 sink 。

```
/base/path/{date-time}/part-{parallel-task}-{count}
```

Where `date-time` is the string that we get from the date/time format, `parallel-task` is the index
of the parallel sink instance and `count` is the running number of part files that where created
because of the batch size.
对于字符串 `date-time` ，我们可以获得日期/时间的格式， `parallel-task` 是并行的 sink 实例的索引，`count` 是按照批量的大小创建的可运行文件的数量。

For in-depth information, please refer to the JavaDoc for
[RollingSink](http://flink.apache.org/docs/latest/api/java/org/apache/flink/streaming/connectors/fs/RollingSink.html).
更新如的信息，请查阅 JavaDoc [RollingSink](http://flink.apache.org/docs/latest/api/java/org/apache/flink/streaming/connectors/fs/RollingSink.html)。 
