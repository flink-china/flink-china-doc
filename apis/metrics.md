---
title: "Metrics"
# Top-level navigation
top-nav-group: apis
top-nav-pos: 12
top-nav-title: Metrics
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

Flink 提供了一个metric系统帮助收集和提供给其他组件.

* TOC
{:toc}

## 注册 metrics

你可以通过任何继承 RichFunction 类的方法来得到 metric, 只需要调用 getRuntimeContext().getMetricGroup(). 这个方法可以返回一个创建和注册新 metrics 的 MetricGroup 对象.

### Metric 类型

Flink 提供给 Counters, Gauges 和 Histogram 类型.

#### Counter

Counter 使用来提供统计一些东西. 当前的 Counter 值可以使用 inc()/inc(long n) 或者 dec()/dec(long n) 来增加或减少. 你可以通过使用 MetricGroup 的counter(String name)方法创建或者注册一个 Counter.

~~~java
public class MyMapper extends RichMapFunction<String, Integer> {
  private Counter counter;

  @Override
  public void open(Configuration config) {
    this.counter = getRuntimeContext()
      .getMetricGroup()
      .counter("myCounter");
  }

  @public Integer map(String value) throws Exception {
    this.counter.inc();
  }
}
~~~

或者你也可以使用你自己的 Counter 来实现:

~~~java
public class MyMapper extends RichMapFunction<String, Integer> {
  private Counter counter;

  @Override
  public void open(Configuration config) {
    this.counter = getRuntimeContext()
      .getMetricGroup()
      .counter("myCustomCounter", new CustomCounter());
  }
}
~~~

#### Gauge

Gauge 可以提供任何类型的值. 如果你要使用 Gauge 你必须通过继承 org.apache.flink.metrics.Gauge 接口创建自己的类. 对于返回的类型没有严格的限制. 可以通过调用 MetricGroup 的 gauge(String name, Gauge gauge) 来注册一个 gauge.

~~~java
public class MyMapper extends RichMapFunction<String, Integer> {
  private int valueToExpose;

  @Override
  public void open(Configuration config) {
    getRuntimeContext()
      .getMetricGroup()
      .gauge("MyGauge", new Gauge<Integer>() {
        @Override
        public Integer getValue() {
          return valueToExpose;
        }
      });
  }
}
~~~

注意 reporter 会把这个对象转为 String, 所以有必要实现 toString() 方法.

#### Histogram

Histogram 可以用来表示值的分布情况. 你可以通过 MetricGroup 的 histogram(String name, Histogram histogram) 来注册.

~~~java
public class MyMapper extends RichMapFunction<Long, Integer> {
  private Histogram histogram;

  @Override
  public void open(Configuration config) {
    this.histogram = getRuntimeContext()
      .getMetricGroup()
      .histogram("myHistogram", new MyHistogram());
  }

  @public Integer map(Long value) throws Exception {
    this.histogram.update(value);
  }
}
~~~

Flink 并没有一个默认的 Histogram 的实现, 但是提供了一个 [Wrapper](https://github.com/apache/flink/blob/master/flink-metrics/flink-metrics-dropwizard/src/main/java/org/apache/flink/dropwizard/metrics/DropwizardHistogramWrapper.java) 来使用 Codahale/DropWizard histograms. 使用这个 wrapper 你需要在 pom.xml里添加如下依赖:

~~~xml
<dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-metrics-dropwizard</artifactId>
      <version>1.1-SNAPSHOT</version>
</dependency>
~~~

你可以像这样注册一个 Codahale/DropWizard histogram:

~~~java
public class MyMapper extends RichMapFunction<Long, Integer> {
  private Histogram histogram;

  @Override
  public void open(Configuration config) {
    com.codahale.metrics.Histogram histogram =
      new com.codahale.metrics.Histogram(new SlidingWindowReservoir(500));

    this.histogram = getRuntimeContext()
      .getMetricGroup()
      .histogram("myHistogram", new DropWizardHistogramWrapper(histogram));
  }
}
~~~

## Scope

每个 metric 的标识符是由3个部分组成的: 用户注册 metric 时提供的名字, 一个可选择的用户定义 scope 和 系统提供的 scope. 例如, 如果 A.B 是系统 scope, C.D 为用户scope 而 E 是它的名字, 那么它的标识符见鬼标记为 A.B.C.D.E.

你可以通过设置 conf/flink-conf.yaml 里的 metrics.scope.delimiter 值来设置分隔符(默认:,).

### 用户 Scope

你可以通过调用 MetricGroup#addGroup(String name) 或者 MetricGroup#addGroup(int name) 来定义用户 scope.

~~~java
counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetrics")
  .counter("myCounter");
~~~

### 系统 Scope

系统 scoper 将包含此 metric 的上下文信息, 例如哪一个 task 被注册了或者 task 属于哪一个 job.

在 conf/flink-conf.yaml 里可以设置添加哪些上下文信息. 每个 key 都可以包含一些常量(例如 "taskmanager")和一些变量(例如 "\<task_id\>"), 变量将会在运行时被替换.

* metrics.scope.jm
    * 默认: \<host\>.jobmanager
    * Applied to all metrics that were scoped to a job manager.
* metrics.scope.jm.job
    * 默认: \<host\>.jobmanager.\<job_name\>
    * Applied to all metrics that were scoped to a job manager and job.
* metrics.scope.tm
    * 默认: \<host\>.taskmanager.\<tm_id\>
    * Applied to all metrics that were scoped to a task manager.
* metrics.scope.tm.job
    * 默认: \<host\>.taskmanager.\<tm_id\>.\<job_name\>
    * Applied to all metrics that were scoped to a task manager and job.
* metrics.scope.tm.task
    * 默认: \<host\>.taskmanager.\<tm_id\>.\<job_name\>.\<task_name\>.\<subtask_index\>
    * Applied to all metrics that were scoped to a task.
* metrics.scope.tm.operator
    * 默认: \<host\>.taskmanager.\<tm_id\>.\<job_name\>.\<operator_name\>.\<subtask_index\>
    * Applied to all metrics that were scoped to an operator.

没有严格的定义变量的数量和顺序, 变量对大小写敏感.

一个 operator 默认的 scope 类似于 localhost.taskmanager.1234.MyJob.MyOperator.0.MyMetric

如果你希望包含 task 名字 但是希望忽略 task manager 的信息, 你可以按如下格式:

metrics.scope.tm.operator: \<host\>.\<job_name\>.\<task_name\>.\<operator_name\>.\<subtask_index\>

最终你会看到这样的标识符 localhost.MyJob.MySource_->_MyOperator.MyOperator.0.MyMetric.

这样的标识符会在同一 job 多次运行时引发冲突. 为避免这一情况最好还是提供唯一的 job 标识(例如 \<job_id\>) 或者给定 jobs 或者 operators 唯一的名字.

### 所有变量列表

* JobManager: \<host\>
* TaskManager: \<host\>, \<tm_id\>
* Job: \<job_id\>, \<job_name\>
* Task: \<task_id\>, \<task_name\>, ]<task_attempt_id\>, \<task_attempt_num\>, \<subtask_index\>
* Operator: \<operator_name\>, \<subtask_index\>

## Reporter

通过设置 conf/flink-conf.yaml 配置文件, 可以把 Metrics 传给外部的组件.

* metrics.reporters: The list of named reporters.
* metrics.reporter.\<name\>.\<config\>: Generic setting \<config\> for the reporter named \<name\>.
* metrics.reporter.\<name\>.class: The reporter class to use for the reporter named \<name\>.
* metrics.reporter.\<name\>.interval: The reporter interval to use for the reporter named \<name\>.

每个 reporter 都必须包含 class 属性, 一些还可以设置 internal. 下面将会列出各个 reporter 的详细设置.

设置多个 reporter 的例子:

~~~
metrics.reporters: my_jmx_reporter,my_other_reporter

metrics.reporter.my_jmx_reporter.class: org.apache.flink.metrics.jmx.JMXReporter
metrics.reporter.my_jmx_reporter.port: 9020-9040

metrics.reporter.my_other_reporter.class: org.apache.flink.metrics.graphite.GraphiteReporter
metrics.reporter.my_other_reporter.host: 192.168.1.1
metrics.reporter.my_other_reporter.port: 10000
~~~

你可以通过实现 org.apache.flink.metrics.reporter.MetricReporter 接口来写出自己的 reporter. 如果 reporter 需要定时执行的话还得实现 Scheduled 接口.

下面列出了目前支持的 reportre.

### JMX (org.apache.flink.metrics.jmx.JMXReporter)

你不需要额外添加依赖, 因为 JMX 已经默认提供但是没有被开启.

参数:

* port - JMX 监听端口. 也可以是一个端口范围. 当设定这个值后它将会在 job 或者 task manager 的 log 里进行显示. 如果没有指定端口那么将不会有 JMX 服务开启. Metrics 在本地 JMX 接口仍然可用.

### Ganglia (org.apache.flink.metrics.ganglia.GangliaReporter)

依赖:

~~~xml
<dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-metrics-ganglia</artifactId>
      <version>1.1-SNAPSHOT</version>
</dependency>
~~~

参数:

* host - the gmond host address configured under udp_recv_channel.bind in gmond.conf
* port - the gmond port configured under udp_recv_channel.port in gmond.conf
* tmax - soft limit for how long an old metric should be retained
* dmax - hard limit for how long an old metric should be retained
* ttl - time-to-live for transmitted UDP packets
* addressingMode - UDP addressing mode to use (UNICAST/MULTICAST)

### Graphite (org.apache.flink.metrics.graphite.GraphiteReporter)

依赖:

~~~xml
<dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-metrics-graphite</artifactId>
      <version>1.1-SNAPSHOT</version>
</dependency>
~~~

参数:

* host - Graphite 主机名
* port - Graphite 端口

### StatsD (org.apache.flink.metrics.statsd.StatsDReporter)

依赖:

~~~xml
<dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-metrics-statsd</artifactId>
      <version>1.1-SNAPSHOT</version>
</dependency>
~~~

参数:

* host - StatsD 主机名
* port - StatsD 端口

## 系统 metrics

Flink 提供如下系统 metrics 给外部使用:

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left">Metrics</th>
      <th class="text-left">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <th rowspan="1"><strong>JobManager</strong></th>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <th rowspan="19"><strong>TaskManager.Status.JVM</strong></th>
      <td>ClassLoader.ClassesLoaded</td>
      <td>The total number of classes loaded since the start of the JVM.</td>
    </tr>
    <tr>
      <td>ClassLoader.ClassesUnloaded</td>
      <td>The total number of classes unloaded since the start of the JVM.</td>
    </tr>
    <tr>
      <td>GargabeCollector.&lt;garbageCollector&gt;.Count</td>
      <td>The total number of collections that have occurred.</td>
    </tr>
    <tr>
      <td>GargabeCollector.&lt;garbageCollector&gt;.Time</td>
      <td>The total time spent performing garbage collection.</td>
    </tr>
    <tr>
      <td>Memory.Heap.Used</td>
      <td>The amount of heap memory currently used.</td>
    </tr>
    <tr>
      <td>Memory.Heap.Committed</td>
      <td>The amount of heap memory guaranteed to be available to the JVM.</td>
    </tr>
    <tr>
      <td>Memory.Heap.Max</td>
      <td>The maximum amount of heap memory that can be used for memory management.</td>
    </tr>
    <tr>
      <td>Memory.NonHeap.Used</td>
      <td>The amount of non-heap memory currently used.</td>
    </tr>
    <tr>
      <td>Memory.NonHeap.Committed</td>
      <td>The amount of non-heap memory guaranteed to be available to the JVM.</td>
    </tr>
    <tr>
      <td>Memory.NonHeap.Max</td>
      <td>The maximum amount of non-heap memory that can be used for memory management.</td>
    </tr>
    <tr>
      <td>Memory.Direct.Count</td>
      <td>The number of buffers in the direct buffer pool.</td>
    </tr>
    <tr>
      <td>Memory.Direct.MemoryUsed</td>
      <td>The amount of memory used by the JVM for the direct buffer pool.</td>
    </tr>
    <tr>
      <td>Memory.Direct.TotalCapacity</td>
      <td>The total capacity of all buffers in the direct buffer pool.</td>
    </tr>
    <tr>
      <td>Memory.Mapped.Count</td>
      <td>The number of buffers in the mapped buffer pool.</td>
    </tr>
    <tr>
      <td>Memory.Mapped.MemoryUsed</td>
      <td>The amount of memory used by the JVM for the mapped buffer pool.</td>
    </tr>
    <tr>
      <td>Memory.Mapped.TotalCapacity</td>
      <td>The number of buffers in the mapped buffer pool.</td>
    </tr>
    <tr>
      <td>Threads.Count</td>
      <td>The total number of live threads.</td>
    </tr>
    <tr>
      <td>CPU.Load</td>
      <td>The recent CPu usage of the JVM.</td>
    </tr>
    <tr>
      <td>CPU.Time</td>
      <td>The CPU time used by the JVM.</td>
    </tr>
    <tr>
      <th rowspan="1"><strong>Job</strong></th>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <tr>
        <th rowspan="5"><strong>Task</strong></th>
        <td>currentLowWatermark</td>
        <td>The lowest watermark a task has received.</td>
      </tr>
      <tr>
        <td>lastCheckpointDuration</td>
        <td>The time it took to complete the last checkpoint.</td>
      </tr>
      <tr>
        <td>lastCheckpointSize</td>
        <td>The total size of the last checkpoint.</td>
      </tr>
      <tr>
        <td>restartingTime</td>
        <td>The time it took to restart the job.</td>
      </tr>
      <tr>
        <td>numBytesInLocal</td>
        <td>The total number of bytes this task has read from a local source.</td>
      </tr>
      <tr>
        <td>numBytesInRemote</td>
        <td>The total number of bytes this task has read from a remote source.</td>
      </tr>
      <tr>
        <td>numBytesOut</td>
        <td>The total number of bytes this task has emitted.</td>
      </tr>
    </tr>
    <tr>
      <tr>
        <th rowspan="3"><strong>Operator</strong></th>
        <td>numRecordsIn</td>
        <td>The total number of records this operator has received.</td>
      </tr>
      <tr>
        <td>numRecordsOut</td>
        <td>The total number of records this operator has emitted.</td>
      </tr>
      <tr>
        <td>numSplitsProcessed</td>
        <td>The total number of InputSplits this data source has processed.</td>
      </tr>
    </tr>
  </tbody>
</table>

