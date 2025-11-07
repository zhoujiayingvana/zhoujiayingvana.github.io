# Spark配置
## 基本概念
###  一、核心概念  
| 角色 | 作用 | 运行位置 |
| --- | --- | --- |
| **Driver** | 负责调度任务、生成执行计划、协调 executors | 通常在 YARN 的某个节点上（cluster 模式在core节点） |
| **Executor** | 真正执行 task 的进程（运行 SQL、算子等） | 分布在各个 worker 节点（core/task 节点）上 |


###  二、关键配置含义  
| 配置项 | 含义 | 示例 |
| --- | --- | --- |
| `spark.executor.memory` | 每个 executor 可用内存 | `4g`<br/> → 每个 executor 有 4GB 内存 |
| `spark.executor.cores` | 每个 executor 使用的 CPU 核数 | `2`<br/> → 每个 executor 用 2 个 vCPU |
| `spark.driver.memory` | Driver 进程的内存 | `4g`<br/> → 调度进程本身有 4GB 内存 |
| `spark.num.executors` | Executor 的数量（静态） | `N`<br/> → 启动 N 个 executor |
| spark.dynamicAllocation.enabled | 是否动态分配 | true：动态分配<br/>false：不动态分配<br/>于spark.num.executors互斥，指定了这个，则动态分配不生效 |


### 三、资源总量计算示例
假设：

```plain
.config("spark.executor.memory", "4g")
.config("spark.executor.cores", "2")
.config("spark.num.executors", "3")
.config("spark.driver.memory", "4g")
```

👉 那么：

+ **Driver** 占用资源：
    - 1 个进程
    - 4 GB 内存（由 `spark.driver.memory` 控制）
+ **Executors** 占用资源：
    - 3 个 executor（由 `spark.num.executors` 控制）
    - 每个 executor 有：
        * 4 GB 内存
        * 2 个 vCPU

📦 **集群总资源占用：**

```plain
CPU： 3 executors × 2 cores = 6 cores
内存：3 executors × 4G = 12G
外加 Driver 4G（总计约 16G）
```

---

### 四、与动态分配的关系
如果改为：

```plain
.config("spark.dynamicAllocation.enabled", "true")
```

而 **不设置**`spark.num.executors`，  
则 Spark 会在运行中动态调整 executor 数量（例如从 2 到 20），  
但每个 executor 的大小仍由：

```plain
spark.executor.memory, spark.executor.cores
```

决定。

---

✅ **一句话总结：**

Driver 负责调度；Executors 负责计算；  
每个 executor 的“规格”由 memory/cores 决定，  
executor 的“数量”由 num.executors（静态）或 dynamicAllocation（动态）决定。



## 推荐配置
### EMR配置
master（1）：m5.xlarge，4核16G，ebs_size_gb：64

core(3)（尝试）：m6i.xlarge，4核16G，ebs_size_gb: 128

并行step数量：2

可用cpu：3*4 =12

可用内存：16*3 = 48GB

每个step运行时的内存使用：4*4+4 = 20GB

实际每个executor会额外分配0.4G，总计：约22GB

```plain
.config("spark.executor.memory", "4g")
.config("spark.executor.cores", "2")
.config("spark.num.executors", "4")
.config("spark.driver.memory", "4g")
.config("spark.sql.shuffle.partitions", "200") \
.config("spark.network.timeout", "600s") \
.config("spark.executor.heartbeatInterval", "30s")
```

🧩 一、`spark.sql.shuffle.partitions`

| 项目 | 内容 |
| --- | --- |
| **默认值** | `200` |
| **含义** | Spark SQL 在执行 `join`<br/>、`groupBy`<br/>、`aggregate`<br/> 等操作时，**shuffle 阶段生成的分区数量**。 |
| **调整建议** | <ul><li>数据量**较小（百万级以下）** → 100~200</li><li>数据量**较大（千万级以上）** → 可调高到 400~1000</li></ul> |


✅ 优点（调大）

+ 提高并行度，更多 task 并发执行；
+ 降低单个 task 数据量，减少溢写压力。

⚠️ 缺点（调大）

+ Task 数太多会造成调度开销增加；
+ 生成更多 shuffle 文件，**Driver 调度压力上升**；
+ 可能导致小文件过多。

---

🧩 二、`spark.network.timeout`

| 项目 | 内容 |
| --- | --- |
| **默认值** | `120s`<br/>（2 分钟） |
| **含义** | Spark 网络通信的超时时间（包括 executor 与 driver 心跳、shuffle 数据传输等）。 |
| **你设置的值** | `600s`<br/>（10 分钟） |


✅ 调大的优点

+ 对长时间运行任务更稳定；
+ 避免 executor 因长时间 shuffle 或 GC 被错误地判定为“丢失”；
+ 特别适合大 SQL、长时间 GC 或 IO 慢的场景。

⚠️ 调大的缺点

+ 真正失联的 executor 需要更久才被回收；
+ 如果任务确实挂了，会延迟失败检测时间。

---

🧩 三、`spark.executor.heartbeatInterval`

| 项目 | 内容 |
| --- | --- |
| **默认值** | `10s` |
| **含义** | Executor 向 Driver 汇报心跳的间隔时间，用来维持活跃状态。 |
| **你设置的值** | `30s` |


✅ 调大的优点

+ 减少心跳频率，降低网络和 Driver 调度负载；
+ 对大集群、很多 executor 的情况更友好。

⚠️ 调大的缺点

+ 心跳太慢时，若 `spark.network.timeout` 设置太短，会误判 executor “失联”；
+ 不适合短任务密集、延迟敏感的 workload。

# 常见问题
## 为什么3个core的cpu总共12核，要给每个step设置8核cpu。当有2个step，cpu使用16核心
非常好的问题 👍，你抓到的是 **Spark/YARN资源调度的核心逻辑**。  
下面我给你简明解释一下为什么配置上“看似超过12核”，但实际上仍然能正常运行。

---

### 🧠 一、关键概念：Spark 配置 ≠ 物理CPU限制
在 **YARN 模式** 下，  
Spark 的 `.config("spark.executor.cores", N)`、`num.executors` 等参数只是**请求资源的上限**，  
真正的分配由 **YARN ResourceManager** 控制。

🧩 你告诉 Spark：“我想要 8 核”，  
但 Yarn 可能说：“我现在只有 6 核，给你 6 核先跑，等别的任务释放再补”。

所以：

+ Spark 配置的是**逻辑请求量（期望值）**；
+ Yarn 决定**实际分配量（可用量）**；
+ 超配（oversubscription）是常见做法，因为任务不会同时打满 CPU。

---

### ⚙️ 二、为什么可以“配置超出”总核数？
假设集群资源如下：

+ 3 core 节点 × 4 vCPU = **12 核**；
+ 每个 Step 申请 8 核；
+ 并发 2 个 Step，总共请求 16 核。

YARN 的行为：

| 状态 | YARN 行为 |
| --- | --- |
| 空闲资源 ≥ 8 核 | 直接分配，Step 同时运行 |
| 空闲资源 < 8 核 | 先分配部分资源（如 6 核），待有空闲时再补 executor |
| 资源不足太多 | 第二个 Step 处于 “ACCEPTED” 状态，等待第一个释放资源 |


💡 所以，**YARN 并不会真的启动超过 12 核的容器**，而是根据实际资源动态调度。  
Spark 的 `num.executors` 是上限，不是硬限制。

---

### ⚙️ 三、为什么建议配置略超
原因有三个：

1. **避免长时间 idle**  
如果 Spark 设置太保守（如每作业只申请 4 核），两个作业一起跑时资源可能空闲一半。
2. **YARN 资源切片粒度较粗**  
通常 YARN 最小分配单元是 1 核 / 1GB 内存。申请 8 核的应用比申请 6 核的更容易在资源空闲时立即分配。
3. **Spark 任务峰值不同时出现**  
实际上，大部分 Spark 阶段并不会所有 executor 都 100% 占用 CPU，超配利用率更高。

---

### 🧩 四、结论总结
| 项目 | 说明 |
| --- | --- |
| Core 总核数 | 12 |
| 每 Step 配置 | 8 核（请求上限） |
| 并行 2 Step | 总请求 16 核（逻辑超配） |
| YARN 实际分配 | 动态调度，不会超过物理 12 核 |
| 结果 | 合理利用资源，不会报错，也不会导致物理超载 |


---

✅ **一句话总结**

Spark 的配置是“希望值”，YARN 才是“裁判”。  
适度超配能提高资源利用率，不会真正突破物理限制。

