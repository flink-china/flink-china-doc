---
title: "Elasticsearch Connector"

# Sub-level navigation
sub-nav-group: streaming
sub-nav-parent: connectors
sub-nav-pos: 2
sub-nav-title: Elasticsearch
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

这个连接器提供了 Sink (接收器:以下统称为 Sink )可以向 [Elasticsearch](https://elastic.co/) 的索引中写数据。要使用这个连接器，需要在工程中添加以下依赖。

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-elasticsearch{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

注意，目前 streaming 连接器不是二进制发行包的一部分。
关于如何打包程序和依赖库，并在集群执行，请参考 [这里]({{site.baseurl}}/apis/cluster_execution.html#linking-with-modules-not-contained-in-the-binary-distribution) 

#### 安装 Elasticsearch

创建 Elasticsearch 集群可以参见 [这里](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html).
保证创建并牢记集群的名字，当创建一个 Sink 向集群中写入数据时，需要配置集群的名字。

#### Elasticsearch Sink
Elasticsearch Sink 提供了一个可以向 Elasticsearch 索引写入数据的接收器。

这个接收器可以使用两个不同的方式与 Elasticsearch 通信。

1. 嵌入式的集群节点(embedded Node)
2. 客户端连接(TransportClient)

参考 [这里](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/client.html)
查看这两种模式的区别。

下方代码演示了如何使用嵌入式的节点创建 Sink ，与集群进行通信。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<String> input = ...;

Map<String, String> config = Maps.newHashMap();
// This instructs the sink to emit after every element, otherwise they would be buffered
config.put("bulk.flush.max.actions", "1");
config.put("cluster.name", "my-cluster-name");

input.addSink(new ElasticsearchSink<>(config, new IndexRequestBuilder<String>() {
    @Override
    public IndexRequest createIndexRequest(String element, RuntimeContext ctx) {
        Map<String, Object> json = new HashMap<>();
        json.put("data", element);

        return Requests.indexRequest()
                .index("my-index")
                .type("my-type")
                .source(json);
    }
}));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[String] = ...

val config = new util.HashMap[String, String]
config.put("bulk.flush.max.actions", "1")
config.put("cluster.name", "my-cluster-name")

text.addSink(new ElasticsearchSink(config, new IndexRequestBuilder[String] {
  override def createIndexRequest(element: String, ctx: RuntimeContext): IndexRequest = {
    val json = new util.HashMap[String, AnyRef]
    json.put("data", element)
    println("SENDING: " + element)
    Requests.indexRequest.index("my-index").`type`("my-type").source(json)
  }
}))
{% endhighlight %}
</div>
</div>

注意，如何使用 String 类型的 Map 对象来配置 Sink。配置的 keys 记录在 Elasticsearch 的文档中 [这里](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html).
特别重要的是 `cluster.name` 参数，必须和集群配置的名字一致。

底层实现中， sink 使用 `BulkProcessor` 方式向集群批量发送索引请求。
这会在发送请求之前缓存每个元素。`BulkProcessor` 操作可以使用以下这些参数进行配置：
 * **bulk.flush.max.actions**: 最大缓存数
 * **bulk.flush.max.size.mb**: 最大缓存数据大小(单位MB)
 * **bulk.flush.interval.ms**: 不考虑以上两个配置,集群刷新数据的间隔时间（ms）

下方的示例代码使用 `TransportClient` 方式连接集群，实现功能同 embedded Node 方式。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<String> input = ...;

Map<String, String> config = Maps.newHashMap();
// This instructs the sink to emit after every element, otherwise they would be buffered
config.put("bulk.flush.max.actions", "1");
config.put("cluster.name", "my-cluster-name");

List<TransportAddress> transports = new ArrayList<String>();
transports.add(new InetSocketTransportAddress("node-1", 9300));
transports.add(new InetSocketTransportAddress("node-2", 9300));

input.addSink(new ElasticsearchSink<>(config, transports, new IndexRequestBuilder<String>() {
    @Override
    public IndexRequest createIndexRequest(String element, RuntimeContext ctx) {
        Map<String, Object> json = new HashMap<>();
        json.put("data", element);

        return Requests.indexRequest()
                .index("my-index")
                .type("my-type")
                .source(json);
    }
}));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[String] = ...

val config = new util.HashMap[String, String]
config.put("bulk.flush.max.actions", "1")
config.put("cluster.name", "my-cluster-name")

val transports = new ArrayList[String]
transports.add(new InetSocketTransportAddress("node-1", 9300))
transports.add(new InetSocketTransportAddress("node-2", 9300))

text.addSink(new ElasticsearchSink(config, transports, new IndexRequestBuilder[String] {
  override def createIndexRequest(element: String, ctx: RuntimeContext): IndexRequest = {
    val json = new util.HashMap[String, AnyRef]
    json.put("data", element)
    println("SENDING: " + element)
    Requests.indexRequest.index("my-index").`type`("my-type").source(json)
  }
}))
{% endhighlight %}
</div>
</div>

这两种方式的不同之处在于，使用 `TransportClient` 方式连接集群，我们需要为 Sink 提供 Elasticsearch 集群的机器列表。

更多 Elasticsearch 信息请参考 [这里](https://elastic.co) 。