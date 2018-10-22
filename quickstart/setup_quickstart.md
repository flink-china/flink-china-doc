---
title: "快速起步"
nav-title: '<i class="fa fa-power-off title appetizer" aria-hidden="true"></i> 快速起步'
nav-parent_id: root
nav-pos: 2
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

通过以下步骤把一个Flink示例程序跑起来.

## Setup:下载和启动Flink

Flink可以再 __Linux, Mac OS X, and Windows__ 上运行. 为了运行Flink, 需要安装 __Java 7.x__ (或者更高版本的Java). Windows用户, 请参通过下步骤 来运行Flink[Flink on Windows]({{ site.baseurl }}/setup/flink_on_windows.html) .

通过以下命令来检测Java是否正确安装:

~~~bash
java -version
~~~

如果你安装的Java 8, 运行后会输出如下:

~~~bash
java version "1.8.0_111"
Java(TM) SE Runtime Environment (build 1.8.0_111-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.111-b14, mixed mode)
~~~

{% if site.is_stable %}
<div class="codetabs" markdown="1">

<div data-lang="Download and Unpack" markdown="1">
1. 通过 [downloads page](http://flink.apache.org/downloads.html)下载Flink的bin文件. Y你可以
选择任何你喜欢的Hadoop/Scala版本. 如果你只使用本地文件系统, 任何
版本的Hadoop都是OK的.
2. 进入到下载目录.
3. 解压文件.

~~~bash
$ cd ~/Downloads        # Go to download directory
$ tar xzf flink-*.tgz   # Unpack the downloaded archive
$ cd flink-{{site.version}}
~~~
</div>

<div data-lang="MacOS X" markdown="1">
对于MacOS X 用户, 可通过如下步骤安装Flink [Homebrew](https://brew.sh/).

~~~bash
$ brew install apache-flink
...
$ flink --version
Version: 1.2.0, Commit ID: 1c659cf
~~~
</div>

</div>

{% else %}
### 下载和编译
克隆源码 [repositories](http://flink.apache.org/community.html#source-code), e.g.:

~~~bash
$ git clone https://github.com/apache/flink.git
$ cd flink
$ mvn clean package -DskipTests # this will take up to 10 minutes
$ cd build-target               # this is where Flink is installed to
~~~
{% endif %}

### 启动Flink 本地模式集群

~~~bash
$ ./bin/start-local.sh  # Start Flink
~~~

 通过访问 [http://localhost:8081](http://localhost:8081)检查 __JobManager's 网页__ ， 确认是否已经启动. 网页端会有一个可用TaskManager 实例.

<a href="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-1.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-1.png" alt="JobManager: Overview"/></a>

你也可通过检查 `logs` 目录下的日志来确认是否已经启动:

~~~bash
$ tail log/flink-*-jobmanager-*.log
INFO ... - Starting JobManager
INFO ... - Starting JobManager web frontend
INFO ... - Web frontend listening at 127.0.0.1:8081
INFO ... - Registered TaskManager at 127.0.0.1 (akka://flink/user/taskmanager)
~~~

## 阅读代码

在GitHub上你可以在这找到完整的[scala](https://github.com/apache/flink/blob/master/flink-examples/flink-examples-streaming/src/main/scala/org/apache/flink/streaming/scala/examples/socket/SocketWindowWordCount.scala)语言的SocketWindowWordCount 例子in  和 [java](https://github.com/apache/flink/blob/master/flink-examples/flink-examples-streaming/src/main/java/org/apache/flink/streaming/examples/socket/SocketWindowWordCount.java) .

<div class="codetabs" markdown="1">
<div data-lang="scala" markdown="1">
{% highlight scala %}
object SocketWindowWordCount {

    def main(args: Array[String]) : Unit = {

        // the port to connect to
        val port: Int = try {
            ParameterTool.fromArgs(args).getInt("port")
        } catch {
            case e: Exception => {
                System.err.println("No port specified. Please run 'SocketWindowWordCount --port <port>'")
                return
            }
        }

        // get the execution environment
        val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment

        // get input data by connecting to the socket
        val text = env.socketTextStream("localhost", port, '\n')

        // parse the data, group it, window it, and aggregate the counts
        val windowCounts = text
            .flatMap { w => w.split("\\s") }
            .map { w => WordWithCount(w, 1) }
            .keyBy("word")
            .timeWindow(Time.seconds(5), Time.seconds(1))
            .sum("count")

        // print the results with a single thread, rather than in parallel
        windowCounts.print().setParallelism(1)

        env.execute("Socket Window WordCount")
    }

    // Data type for words with count
    case class WordWithCount(word: String, count: Long)
}
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
public class SocketWindowWordCount {

    public static void main(String[] args) throws Exception {

        // the port to connect to
        final int port;
        try {
            final ParameterTool params = ParameterTool.fromArgs(args);
            port = params.getInt("port");
        } catch (Exception e) {
            System.err.println("No port specified. Please run 'SocketWindowWordCount --port <port>'");
            return;
        }

        // get the execution environment
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // get input data by connecting to the socket
        DataStream<String> text = env.socketTextStream("localhost", port, "\n");

        // parse the data, group it, window it, and aggregate the counts
        DataStream<WordWithCount> windowCounts = text
            .flatMap(new FlatMapFunction<String, WordWithCount>() {
                @Override
                public void flatMap(String value, Collector<WordWithCount> out) {
                    for (String word : value.split("\\s")) {
                        out.collect(new WordWithCount(word, 1L));
                    }
                }
            })
            .keyBy("word")
            .timeWindow(Time.seconds(5), Time.seconds(1))
            .reduce(new ReduceFunction<WordWithCount>() {
                @Override
                public WordWithCount reduce(WordWithCount a, WordWithCount b) {
                    return new WordWithCount(a.word, a.count + b.count);
                }
            });

        // print the results with a single thread, rather than in parallel
        windowCounts.print().setParallelism(1);

        env.execute("Socket Window WordCount");
    }

    // Data type for words with count
    public static class WordWithCount {

        public String word;
        public long count;

        public WordWithCount() {}

        public WordWithCount(String word, long count) {
            this.word = word;
            this.count = count;
        }

        @Override
        public String toString() {
            return word + " : " + count;
        }
    }
}
{% endhighlight %}
</div>
</div>

## 运行例子

现在我们要运行Flink程序. 程序将会每5秒钟读取socket端的数据流，打印单词出现的频率.

* 首先, 我们启动命令 **netcat** 打开服务端口

  ~~~bash
  $ nc -l 9000
  ~~~

* 提交Flink 程序:

  ~~~bash
  $ ./bin/flink run examples/streaming/SocketWindowWordCount.jar --port 9000

  Cluster configuration: Standalone cluster with JobManager at /127.0.0.1:6123
  Using address 127.0.0.1:6123 to connect to JobManager.
  JobManager web interface address http://127.0.0.1:8081
  Starting execution of program
  Submitting job with JobID: 574a10c8debda3dccd0c78a3bde55e1b. Waiting for job completion.
  Connected to JobManager at Actor[akka.tcp://flink@127.0.0.1:6123/user/jobmanager#297388688]
  11/04/2016 14:04:50     Job execution switched to status RUNNING.
  11/04/2016 14:04:50     Source: Socket Stream -> Flat Map(1/1) switched to SCHEDULED
  11/04/2016 14:04:50     Source: Socket Stream -> Flat Map(1/1) switched to DEPLOYING
  11/04/2016 14:04:50     Fast TumblingProcessingTimeWindows(5000) of WindowedStream.main(SocketWindowWordCount.java:79) -> Sink: Unnamed(1/1) switched to SCHEDULED
  11/04/2016 14:04:51     Fast TumblingProcessingTimeWindows(5000) of WindowedStream.main(SocketWindowWordCount.java:79) -> Sink: Unnamed(1/1) switched to DEPLOYING
  11/04/2016 14:04:51     Fast TumblingProcessingTimeWindows(5000) of WindowedStream.main(SocketWindowWordCount.java:79) -> Sink: Unnamed(1/1) switched to RUNNING
  11/04/2016 14:04:51     Source: Socket Stream -> Flat Map(1/1) switched to RUNNING
  ~~~

  程序连接上socket，等待输入. 可以通过web端访问到运行的程序实例:

  <div class="row">
    <div class="col-sm-6">
      <a href="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-2.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-2.png" alt="JobManager: Overview (cont'd)"/></a>
    </div>
    <div class="col-sm-6">
      <a href="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-3.png" ><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-setup/jobmanager-3.png" alt="JobManager: Running Jobs"/></a>
    </div>
  </div>

* 单词每5秒钟被计数一次 并打印. 

  ~~~bash
  $ nc -l 9000
  lorem ipsum
  ipsum ipsum ipsum
  bye
  ~~~

  `.out` 文件打印出每次统计的结果:

  ~~~bash
  $ tail -f log/flink-*-jobmanager-*.out
  lorem : 1
  bye : 1
  ipsum : 4
  ~~~~

   如下操作**停止** Flink:

  ~~~bash
  $ ./bin/stop-local.sh
  ~~~

## 下一步

Check out some more [examples]({{ site.baseurl }}/examples) to get a better feel for Flink's programming APIs. When you are done with that, go ahead and read the [streaming guide]({{ site.baseurl }}/dev/datastream_api.html).
