---
title: Flink 重启策略
date: "2020-04-10 14:30:00"
categories:
- 大数据
- Flink
tags:
- 大数据
- Flink
toc: true
typora-root-url: ..\..\..
---

## Flink的重启策略

Flink支持不同的重启策略，这些重启策略控制着job失败后如何重启。集群可以通过默认的重启策略来重启，这个默认的重启策略通常在未指定重启策略的情况下使用，而如果Job提交的时候指定了重启策略，这个重启策略就会覆盖掉集群的默认重启策略。

#### 概览

默认的重启策略是通过Flink的`flink-conf.yaml`来指定的，这个配置参数`restart-strategy`定义了哪种策略会被采用。如果checkpoint未启动，就会采用`no restart`策略，如果启动了checkpoint机制，但是未指定重启策略的话，就会采用`fixed-delay`策略，重试`Integer.MAX_VALUE`次。请参考下面的可用重启策略来了解哪些值是支持的。

每个重启策略都有自己的参数来控制它的行为，这些值也可以在配置文件中设置，每个重启策略的描述都包含着各自的配置值信息。

| 重启策略     | 重启策略值   |
| ------------ | ------------ |
| Fixed delay  | fixed-delay  |
| Failure rate | failure-rate |
| No restart   | None         |

除了定义一个默认的重启策略之外，你还可以为每一个Job指定它自己的重启策略，这个重启策略可以在`ExecutionEnvironment`中调用`setRestartStrategy()`方法来程序化地调用，主意这种方式同样适用于`StreamExecutionEnvironment`。

下面的例子展示了我们如何为我们的Job设置一个固定延迟重启策略，一旦有失败，系统就会尝试每10秒重启一次，重启3次。
 Java代码

```cpp
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // 尝试重启次数
  Time.of(10, TimeUnit.SECONDS) // 延迟时间间隔
));
```

Scala代码:

```kotlin
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // 重启次数
  Time.of(10, TimeUnit.SECONDS) // 延迟时间间隔
))
```

#### 重启策略

下面部分描述了重启策略特定的配置项

###### 固定延迟重启策略(Fixed Delay Restart Strategy)

固定延迟重启策略会尝试一个给定的次数来重启Job，如果超过了最大的重启次数，Job最终将失败。在连续的两次重启尝试之间，重启策略会等待一个固定的时间。

重启策略可以配置flink-conf.yaml的下面配置参数来启用，作为默认的重启策略:

```csharp
restart-strategy: fixed-delay
```

| 配置参数                              | 描述                                                         | 默认值                                       |
| ------------------------------------- | ------------------------------------------------------------ | -------------------------------------------- |
| restart-strategy.fixed-delay.attempts | 在Job最终宣告失败之前，Flink尝试执行的次数                   | 1，如果启用checkpoint的话是Integer.MAX_VALUE |
| restart-strategy.fixed-delay.delay    | 延迟重启意味着一个执行失败之后，并不会立即重启，而是要等待一段时间。 | akka.ask.timeout,如果启用checkpoint的话是1s  |

例子:

```css
restart-strategy.fixed-delay.attempts: 3
restart-strategy.fixed-delay.delay: 10 s
```

固定延迟重启也可以在程序中设置:
 Java代码:

```cpp
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // 重启次数
  Time.of(10, TimeUnit.SECONDS) // 重启时间间隔
));
```

Scala代码:

```kotlin
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // 重启次数
  Time.of(10, TimeUnit.SECONDS) // 重启时间间隔
))
```

###### 失败率重启策略

失败率重启策略在Job失败后会重启，但是超过失败率后，Job会最终被认定失败。在两个连续的重启尝试之间，重启策略会等待一个固定的时间。

失败率重启策略可以在flink-conf.yaml中设置下面的配置参数来启用:

```css
restart-strategy:failure-rate
```

| 配置参数                                                | 描述                                    | 默认值           |
| ------------------------------------------------------- | --------------------------------------- | ---------------- |
| restart-strategy.failure-rate.max-failures-per-interval | 在一个Job认定为失败之前，最大的重启次数 | 1                |
| restart-strategy.failure-rate.failure-rate-interval     | 计算失败率的时间间隔                    | 1分钟            |
| restart-strategy.failure-rate.delay                     | 两次连续重启尝试之间的时间间隔          | akka.ask.timeout |

```css
restart-strategy.failure-rate.max-failures-per-interval: 3
restart-strategy.failure-rate.failure-rate-interval: 5 min
restart-strategy.failure-rate.delay: 10 s
```

失败率重启策略也可以在程序中设置:
 Java代码:

```cpp
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // 每个测量时间间隔最大失败次数
  Time.of(5, TimeUnit.MINUTES), //失败率测量的时间间隔
  Time.of(10, TimeUnit.SECONDS) // 两次连续重启尝试的时间间隔
));
```

Scala代码:

```kotlin
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // 每个测量时间间隔最大失败次数
  Time.of(5, TimeUnit.MINUTES), //失败率测量的时间间隔
  Time.of(10, TimeUnit.SECONDS) // 两次连续重启尝试的时间间隔
))
```

###### 无重启策略

Job直接失败，不会尝试进行重启

```swift
restart-strategy: none
```

无重启策略也可以在程序中设置
 Java代码:

```undefined
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.noRestart());
```

Scala代码:

```kotlin
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.noRestart())
```