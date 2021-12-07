# 数据逐出策略

在redis中，允许用户设置最大使用内存大小`maxmemory`（需要配合`maxmemory-policy`使用），设置为0表示不限制(默认配置)。

生产环境中需要设置此值，最好不超过内存`60%-70%`。

当redis内存数据集快到达maxmemory时，redis会实行数据淘汰策略。

## Redis提供`6种`数据淘汰策略。

在逐出算法中，根据用户设置的逐出策略，选出待逐出的key，直到当前内存小于最大内存值为止。

### **可选逐出策略如下：**

1. `volatile-lru`：从已设置过期时间的数据集中挑选最近最少使用的数据淘汰
2. `volatile-ttl`：从已设置过期时间的数据集中挑选将要过期的数据淘汰
3. `volatile-random`：从已设置过期时间的数据集中任意选择数据 淘汰
4. `allkeys-lru`：从数据集中挑选最近最少使用的数据淘汰
5. `allkeys-random`：从数据集中任意选择数据淘汰
6. `no-enviction`（驱逐）：禁止驱逐数据

## 默认策略

在redis2.8中默认策略是volatile-lru

在redis3.2和redis4.0中默认策略是no-eviction

如果使用no-eviction时，当内存不足，Redis会返回OOM的错误信息(error) OOM command not allowed when used memory > ‘maxmemory’.

当cache中没有符合清除条件的key时，回收策略 volatile-lru, volatile-random 和volatile-ttl 将会和策略 noeviction 一样直接返回错误。

选择正确的回收策略是很重要的，取决于你的应用程序的访问模式。使用INFO命令输出来监控缓存命中和错过的次数，以调优Redis的配置。

## 通用规则如下：

- 如果期望用户请求呈现幂律分布(power-law distribution)，也就是，期望一部分子集元素被访问得远比其他元素多时，可以使用allkeys-lru策略。
- 如果期望是循环周期的访问，所有的键被连续扫描，
- 或者期望请求符合平均分布(每个元素以相同的概率被访问)，可以使用allkeys-random策略。
- 如果期望是让redis使用缓存对象设置的TTL值，确定哪些对象应该是较好的清除候选项，可以使用volatile-ttl策略。