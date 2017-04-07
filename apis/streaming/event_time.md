---
title: "事件时间"

sub-nav-id: eventtime
sub-nav-group: streaming
sub-nav-pos: 2
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

* toc
{:toc}

# 事件时间 / 处理时间 / 采集时间

Flink流处理中支持不同类型的 *时间*.

- **处理时间:** 处理时间指执行相应操作的机器的系统时间.

    如果流处理使用处理时间, 所有时间相关操作 (如时间窗口) 将使用执行相应操作的机器的系统时间.
    比如一个一小时的窗口,会包含在系统时间一小时内到达operator的记录。

    处理时间是最简单的时间模式,不需要各个流和各个机器之间协调时间。这种方式性能最好,延迟最低。然而,在分布式和异步的场景中,
    处理时间可能无法用于决策,因为处理时间容易受到很多因素影响,消息到达系统的速度(比如来自不同的消息队列),或者消息在系统的各个operator之间
    流动的速度不同。

- **事件时间:**
    事件时间是指每条记录在相应设备生成的时间。该时间通常包含在消息中,比进入Flink的时间早,*事件时间戳*可以从消息中获得。
    比如一个一小时的窗口使用事件时间时,会包含事件本身携带的时间落在该小时内的所有记录,无论记录何时到达,以什么顺序到达。

    事件时间可以正确处理乱序事件,延迟事件,或者从备份设备或持久记录重放的事件。使用事件时间时,时间进度取决于数据本身,而非
    任何墙钟。程序必须指定如何产生*事件时间水位*,水位机制用来标识时间进度,见后面描述。

    事件时间常引起固定延迟,因为它需要等待延迟事件和乱序事件一段固定的时间。因此使用事件时间时,常常和*处理时间*一起使用。

- **采集时间:**
    采集时间是事件进入Flink的时间。在source operator中,每个事件使用source的当前系统时间作为时间戳,基于时间的操作(比如时间窗口)
    使用该时间戳。

    概念上讲,*采集时间*位于*事件时间*和*处理时间*之间。它比*处理时间*稍微昂贵,但结果也准确一些:因为*采集时间*
    使用固定的时间戳(由source设置一次)。对同一条记录,不同的时间操作会使用相同的时间戳,而*处理时间*模式中,不同的窗口operator可能会把同一条
    记录划分到不同的窗口去(由于operator的系统时间不同和传输延迟)。

    与*事件时间*相比,*采集时间*模式无法处理乱序事件或者延迟事件,但是使用采集时间无需指定如何产生*水位*。

    从实现上讲,*采集时间*和事件时间很类似,它自动产生时间戳,自动产生水位。

<img src="fig/times_clocks.svg" class="center" width="80%" />


### 设置时间特性

Flink DataStream通常会在程序开头设置*时间特性*。该设置定义sources如何工作(如是否设置时间戳),它也会定义时间窗口的操作使用哪种
时间,比如 `KeyedStream.timeWindow(Time.secondss(30))`。


下面的例子使用1小时的窗口来聚合事件。窗口操作可以适应不同的时间特性。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.addSource(new FlinkKafkaConsumer09<MyEvent>(topic, schema, props));

stream
    .keyBy( (event) -> event.getUser() )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) -> a.add(b) )
    .addSink(...);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime)
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val stream: DataStream[MyEvent] = env.addSource(new FlinkKafkaConsumer09[MyEvent](topic, schema, props))

stream
    .keyBy( _.getUser )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) => a.add(b) )
    .addSink(...)
{% endhighlight %}
</div>
</div>

如果需要用*事件时间*运行该例子,需要使用事件时间source或指定*Timestamp Assigner & Watermark Generator*。时间戳和水位
让程序得知如何获取时间戳和如何处理乱序事件。


下面部分描述了*时间戳*和*水位*的基本机制。关于如何使用Flink DataStream API指定时间戳和生成水位,参考
[Generating Timestamps / Watermarks]({{ site.baseurl }}/apis/streaming/event_timestamps_watermarks.html)


# 事件时间和水位

*Note: 事件时间(Event Time)的深入介绍, 请参考论文: [Dataflow Model](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43864.pdf)*




