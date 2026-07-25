# Beam 流处理三种窗口：Fixed、Sliding、Session 到底怎么选？

最近在做一个 Beam Streaming 的 POC，研究的是跨境贸易实时风控的场景。说白了就是：银行盯着企业客户的交易流水，发现有问题的要实时报警。

做流处理绕不开的一个概念就是**窗口（Windowing）**。你从 Kafka 或者 Pub/Sub 读进来的数据是无穷无尽的，不可能等所有数据到了再算，所以得把数据切成一块一块的来处理。

Beam 提供了三种最基本的窗口：**Fixed Window**、**Sliding Window**、**Session Window**。这三种东西搞清楚了，流处理的基本功就算打牢了。

先说我模拟的数据长什么样：

```python
Transaction(
    account_id="CORP-ALPHA",   # 企业账号
    amount=40000.0,             # 金额（USD）
    counterparty="HK",          # 来源地区
    event_timestamp=...,        # 交易实际发生时间
)
```

四路数据源，分别模拟香港（低延迟 0~1s）、新加坡（中延迟 2~5s）、伦敦（高延迟 8~15s）和纽约（重度延迟 20~30s），目的是后面配合 Watermark 一起折腾。

下面一个一个说。

## Fixed Window：先学会走路

Fixed Window 是最直观的窗口。固定时长，互不重叠，每笔交易落入且只落入一个窗口。

```python
Window.into(FixedWindows.of(Duration.standardSeconds(30)))
```

拿风控场景来说，每 30 秒切一个窗口，统计每个账号的累计交易额，超 10 万美元就报警。逻辑很直白：

```text
窗口 [12:00:00 - 12:00:30)
  CORP-ALPHA: $245,000 -> 报警 (超 10 万)
  CORP-BETA:  $52,000  -> 正常

窗口 [12:00:30 - 12:01:00)
  CORP-ALPHA: $32,000  -> 正常（清零重算）
```

这就是典型的**时段结算**。每个窗口独立，互不干扰，适合做"这半小时谁超了"这种查询。

缺点也很明显：**窗口边界是硬切的**。比如 CORP-ALPHA 在 12:00:29 来了一笔 $90K，12:00:31 又来了一笔 $90K，分属两个窗口，每个都没超 10 万，单独看都正常。但直觉告诉我们这个账号有问题——只是运气好，刚好卡在窗口边界上。

这就引出了第二种窗口。

## Sliding Window：滚动着看才真实

Sliding Window 也有固定时长，但每隔一小段时间就开一个新窗口，所以窗口之间是**重叠**的。

```python
Window.into(SlidingWindows.of(Duration.standardSeconds(30))
              .every(Duration.standardSeconds(10)))
```

窗口大小 30 秒，每 10 秒滑动一次。同一笔交易会同时落入多个窗口：

```text
窗口 [12:00:00 - 12:00:30) -> CORP-ALPHA: $140K -> 报警
窗口 [12:00:10 - 12:00:40) -> CORP-ALPHA: $140K -> 又报警（同样的交易还在）
窗口 [12:00:20 - 12:00:50) -> CORP-ALPHA: $60K  -> 正常（前面的滑出去了）
```

这看起来像"重复报警"，但在金融风控里，这正是 Sliding Window 的价值所在。连续几个窗口都报同一个账号，说明它**持续处于高风险状态**，不是一过性的窗口边界效应。

Sliding Window 本质上是**滚动风险敞口**的监控——你每 10 秒拉一次过去 30 秒的快照，看到的是一个动态变化的风险轮廓。

那如果交易不是均匀分布的，而是短时间内密集爆发呢？

## Session Window：抓的就是你这种

Session Window 和前两个完全不是一个思路。它没有固定的窗口大小，窗口是**动态伸缩**的，完全由数据的行为模式决定。

它只有一个参数：**Gap Duration（会话间隙）**。

```python
Window.into(SessionWindows.withGapDuration(Duration.standardSeconds(15)))
```

逻辑很简单：同一个账号的两笔交易，如果时间差小于 15 秒，就归到同一个 Session 里；如果超过 15 秒没新交易来，Session 就关闭。

看这个例子：

```text
同一账号 CORP-ALPHA 的交易流：

[10:00:01]  $40K  ─┐
[10:00:05]  $30K  ─┤   间隙都 < 15s
[10:00:09]  $35K  ─┤
[10:00:12]  $50K  ─┘
──────────────────────── Session A: $155K  🔴 报警

         ↑ 安静了 5 分钟 ...

[10:05:30]  $10K  ─┐   间隙 > 15s → 新 Session
──────────────────────── Session B: $10K  ✅ 正常
```

这种窗口天然适合**反洗钱可疑交易检测**。正常企业的交易是稀疏的、分散在多个 Session 里的；可疑账户往往会在短时间内疯狂刷多笔交易，被合并成一个长 Session，累计金额自然就超标了。

Fixed Window 抓不住这种模式——它不管交易是均匀分布还是密集爆发，只看时间段内的总和。Session Window 则把"交易行为的密度"这个维度也考虑了进来。

## 三种窗口对比

实际做 POC 的时候，我把三种窗口都跑了一遍，自己的体会如下：

| 维度 | Fixed Window | Sliding Window | Session Window |
|------|-------------|---------------|----------------|
| 窗口大小 | 固定 | 固定 | 动态变化 |
| 窗口关系 | 不重叠 | 重叠 | 按 Key 独立 |
| 适合场景 | 时段结算、报表 | 趋势监控、滚动敞口 | 行为检测、反欺诈 |
| 状态开销 | 低 | 中（重叠越多越高） | 高（每个 Key 都要维护） |
| 窗口边界效应 | 明显 | 平滑 | 无边界概念 |

没有哪种窗口是万能的，关键看你想要解决什么问题。

做报表对账 -> Fixed Window。
监控风险趋势 -> Sliding Window。
抓突击交易 -> Session Window。

实际问题往往是几种窗口组合使用。比如先用 Session Window 发现可疑会话，再用 Sliding Window 看该账号的总体敞口趋势，最后用 Fixed Window 做记录归档。

这就是我这个 POC 里折腾窗口的一点心得。代码已经放在 [beam-streaming-poc](https://github.com/nvd11/beam-streaming-poc) 里了，有兴趣的可以去看完整的实现。
