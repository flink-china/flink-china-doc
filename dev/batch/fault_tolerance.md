---
标题: "故障容错性"
上级导航: 批处理
导航序号: 2
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
Flink的故障容错性是指在Flink应用在出现故障时，主程序通过恢复机制方式仍然可继续运行。这些故障主要有机器硬件故障，网络故障、还有程序瞬时故障等。

* This will be replaced by the TOC (这些是将要被替换掉的TOC)
{:toc}

批处理环境下的故障容错性 (DataSet API)
----------------------------------------------
在DataSet API中的程序的故障容错性是通过重新尝试运行失败的任务方式实现的。
Flink会在任务定义的时候配置好故障的重试执行的次数，配置的参数为*execution retries*，如果配置的值为0，表示故FLINK的故障容错性是无效的。
要使Flink的故障容错性机制生效，必须使参数*execution retries*设定值大于0，此参数一般常设定为3。
下面的样例是表明如何为Flink DataSet程序配置故障发生时重新偿试的次数。
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setNumberOfExecutionRetries(3);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setNumberOfExecutionRetries(3)
{% endhighlight %}
</div>
</div>

如果你的Flink是在yarm环境下，你也可以通过`flink-conf.yaml`配置文件中指定*execution retries*参数值。
~~~
execution-retries.default: 3
~~~


重试延时机制
------------
我们在执行对故障事件进行重试操作时，其重试操作是可以延时执行的。延时重试的主要意思是，一旦我们发生故障要进行重试时，重试的操作是不一定要立即执行的，可以通过此参数配置一定延时时间后才执行。当我们flink程序与外部系统服务交互时，重试延时机制是很帮助的，例如在与外部系统产生连接或挂起事务时，都要有一个等待事务期，待等待超时后再进行重新尝试连接服务。
你可以为每个Flink应用独自设立一个专有重试延时时间参数，具体配置样例代码如下：
(这个例子显示流API-Dataset Api的简单配置):
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setExecutionRetryDelay(5000); // 5000 milliseconds delay
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.getConfig.setExecutionRetryDelay(5000) // 5000 milliseconds delay
{% endhighlight %}
</div>
</div>

你也可以在`flink-conf.yaml`指定一个重试延时时间的默认值，供其下的所有的flink程序共用，具体配置如下：
~~~
execution-retries.delay: 10 s
~~~

{% top %}
