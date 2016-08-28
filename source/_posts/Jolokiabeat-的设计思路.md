---
title: Jolokiabeat 的设计思路
date: 2016-08-27 19:46:48
tags: [写码,idea,go]
---
##背景
Jolokia真是个好东西，把本来只有JVM才能访问的JMX信息暴露成一个http服务，使得我们只需要操控REST API就能获得想要的效果。

Jolokiabeat是我在工作时遇到了需求打造的，基于elastic公司的libbeat，参考了influxdata家TICK技术栈中的Telegraf中jolokia插件的实现，并且借鉴了logstash的JMX插件的一些想法（比如模板化MBean名称提供更多的额外信息，简化查询规则等等）

##架构
整个程序和其他的beat系列程序一样，都采用了流水线形式的架构：

[JMX_collector] => [Metrics transformer] => [LibBeat.Dispatcher]

JMX_Collector负责从Jolokia服务中拉取metrics
Metrics transformer负责将取回的信息整理
LibBeat.Dispatcher就是将信息发送出去的机制

##配置
Jolokiabeat的alpha版本最初使用TOML作为JMX查询参数的配置文件，结果兜兜转转一大圈以后，发现还是TOML最符合要求（最完美的cson并没有golang的binding，残念）

除了TOML之外，JSON，YAML甚至CSON都曾经在我的考虑范围之内，但是首先JSON因为自身格式需要太多花括号而且其实不算太灵活（主要是我经常想写个注释防止自己忘记一些东西，但是JSON连最后一个逗号都不许你加，字符串只能双引号你还想双斜杠写注释？🙄）。

接下来的CSON其实是一个非常理想的格式，因为它就是一个可以写注释的JSON而且还有各种改进，但是无奈coffeescript自从ES6粉墨登场开始就在走下坡路，而且CSON的库也只有js的支持，所以只好作罢

YAML是个不错的语言，如果喜欢python那种缩进式语法的话，肯定能很喜欢
*但是我从来就觉得python的缩进有时候真的是…………*

尝试一下[hjson](https://hjson.org)格式之后（其实仔细看的话就会发现这货就是cson，而且没有hocon那么复杂，但是没有完美解决对象数组的格式嵌套），我还是乖乖地滚回到了TOML的怀抱…………

于是一个配置文件应该长这样：（每个配置都是以server&context为单位的）
```toml
[[jolokia]]
host = "localhost"
port = 9999
# 用户名和密码
#username:
#password:
#还可以设置proxy代理服务器
# context一定不能少，而且不可以有/在里面
context = "kafka"
# extra可以为beat提供额外的信息
  [jolokia.extra]
    type = "jolokia-beat"
    measurement = "MessageQueue.Kafka"
  [[jolokia.metrics]]
    name = "server.ReplicaManager.${name}"
    bean = "kafka.server:name=*,type=ReplicaManager"
    # 作为标签添加的属性名
    as_tag = ["type","name"]
  [[jolokia.metrics]]
    bean = "kafka.server:name=PartitionCount,type=ReplicaManager"

```

##数据整合变形
这是从jolokia返回的json：
```json
{
    "timestamp": 1472159044,
    "status": 200,
    "request": {
        "mbean": "kafka.server:name=*,type=ReplicaManager",
        "type": "read"
    },
    "value": {
        "kafka.server:name=IsrShrinksPerSec,type=ReplicaManager": {
            "OneMinuteRate": 0.0,
            "Count": 0,
            "RateUnit": "SECONDS",
            "MeanRate": 0.0,
            "FiveMinuteRate": 0.0,
            "EventType": "shrinks",
            "FifteenMinuteRate": 0.0
        },
        "kafka.server:name=PartitionCount,type=ReplicaManager": {
            "Value": 72
        },
        "kafka.server:name=IsrExpandsPerSec,type=ReplicaManager": {
            "OneMinuteRate": 0.0,
            "Count": 0,
            "RateUnit": "SECONDS",
            "MeanRate": 0.0,
            "FiveMinuteRate": 0.0,
            "EventType": "expands",
            "FifteenMinuteRate": 0.0
        },
        "kafka.server:name=UnderReplicatedPartitions,type=ReplicaManager": {
            "Value": 0
        },
        "kafka.server:name=LeaderCount,type=ReplicaManager": {
            "Value": 72
        }
    }
}
```

这是在jason库帮助下进行了变形之后的数据：

```yaml
timestamp : 1472159044
status : 200
result :
    - meta:
        name: "IsrShrinksPerSec"
        type: "ReplicaManager"
      data:
        OneMinuteRate: 0.0
        Count: 0
        MeanRate: 0.0
        FiveMinuteRate: 0.0
        FifteenMinuteRate: 0.0

    - meta:
        name: "PartitionCount"
        type: "ReplicaManager"
      data:
        Value: 72

    - meta:
        name: "IsrExpandsPerSec"
        type: "ReplicaManager"
      data:
        OneMinuteRate: 0.0
        Count: 0
        MeanRate: 0.0
        FiveMinuteRate: 0.0
        FifteenMinuteRate: 0.0

    - meta:
        name: "UnderReplicatedPartitions"
        type: "ReplicaManager"
      data:
        Value: 0

    - meta:
        name: "LeaderCount"
        type: "ReplicaManager"
      data:
        Value: 72
```

另一种情况就是没有使用*通配符的时候：

```json
{
  "timestamp": 1472162915,
  "status": 200,
  "request": {
    "mbean": "kafka.server:name=PartitionCount,type=ReplicaManager",
    "type": "read"
  },
  "value": {
    "Value": 72
  }
}
```

```yaml
timestamp : 1472159044
status : 200
result :
    - meta:
        name: "PartitionCount"
        type: "ReplicaManager"
      data:
        Value: 72
```

```json
{
  "timestamp": 1472167534,
  "status": 200,
  "request": {
    "mbean": "kafka.server:name=IsrExpandsPerSec,type=ReplicaManager",
    "type": "read"
  },
  "value": {
    "OneMinuteRate": 0.0,
    "Count": 0,
    "RateUnit": "SECONDS",
    "MeanRate": 0.0,
    "FiveMinuteRate": 0.0,
    "EventType": "expands",
    "FifteenMinuteRate": 0.0
  }
}
```

```yaml
timestamp : 1472167534
status : 200
result:
    - meta :
        name: "IsrExpandsPerSec"
        type: "ReplicaManager"
      data:
        OneMinuteRate: 0.0
        Count: 0
        MeanRate: 0.0
        FiveMinuteRate: 0.0
        FifteenMinuteRate: 0.0
```

TBC，未完待续……
