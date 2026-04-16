# oom-skill 是什么？
OOM dump 分析能力封装为 skill，写入 知识库/oom-skill/SKILL.md。

这个 skill 包含了完整的分析流程（7 步）、每步的关注点和输出模板、常见 OOM 场景速查表，以及你之前指出的关键注意事项——特别是日志和代码方法必须严格对应、分页查询和全量查询要区分清楚这些经验教训。

# 案例
> 根据 工单/OOM/tmp-dump 分析为什么 test-app oom  ,大概时间： 2026-04-16 10:08, ip :172.19.78.53



## 1、提供 dump文件
gc.txt
jmap.txt
‘memory_dump.hprof
pod.log
threadCount.txt
threads.txt
top.txt

## 文件说明
| 文件 | 采集命令 | 作用 | 能回答什么问题 |
|------|---------|------|--------------|
| `memory_heap.txt` | `jmap -heap <pid>` | JVM 堆配置（最大堆、新生代、老年代大小）和各区域使用率 | 堆是不是真的满了？配了多大内存？哪个区域满了？ |
| `jmap.txt` | `jmap -histo <pid>` | 堆中所有对象的实例数和占用字节数，按大小降序排列 | 哪些类的对象吃掉了内存？是 POI 对象、业务 DTO、还是集合类？ |
| `gc.txt` | `jstat -gcutil <pid> 1000` | GC 统计快照：Young GC / Full GC 次数、耗时，各区域使用率百分比 | GC 是否在 thrashing（疯狂回收但回收不了）？Full GC 频率和耗时？ |
| `top.txt` | `top -H -p <pid> -bn1` | 进程内各线程的 CPU、内存占用，以及系统整体负载 | CPU 是不是被 GC 线程吃光了？系统负载多高？线程数正常吗？ |
| `threadCount.txt` | `jstack <pid> \| grep -c "nid="` 或类似 | 当前 JVM 线程总数 | 是否存在线程泄漏？线程数是否异常（正常几百，异常可能上千） |
| `threads.txt` | `jstack <pid>` | 完整线程 dump，每个线程的堆栈和状态 | 线程在干什么？是否有死锁？哪些线程在等待/阻塞？ |
| `pod.log` | `kubectl logs <pod>` | 应用运行日志 | OOM 前发生了什么请求？哪个接口触发的？有没有异常堆栈？ |
| `memory_dump.hprof` | `jmap -dump:format=b,file=xxx.hprof <pid>` | 完整堆转储二进制文件，包含所有对象及其引用关系 | 用 MAT/VisualVM 打开可以分析对象引用链、找到 GC Root、定位内存泄漏的精确路径 |

简单理解它们的分工：
- `memory_heap.txt` + `gc.txt` 回答"是不是 OOM、GC 状态如何"
- `jmap.txt` 回答"谁吃了内存"
- `top.txt` + `threadCount.txt` + `threads.txt` 回答"系统资源和线程状态"
- `pod.log` 回答"OOM 前发生了什么"
- `memory_dump.hprof` 是终极武器，能做最深入的引用链分析，但需要用专门工具（MAT）打开，文件通常很大



# oom-skill 执行流程

好，按照 SKILL.md 的流程，从第一步开始严格执行。

## 第一步：定位并读取 dump 文件

8 个文件全部存在。按 skill 优先级，先读 P0 文件。

## 第二步：分析 memory_heap.txt — 确认 OOM 事实

**第二步输出：**
```
堆配置：MaxHeap = 2560MB, NewSize = 1024MB, OldSize = 1536MB
堆使用：Eden 100% used, Old Gen 99.98% used, Survivor 0% used (From 0%, To 0%)
结论：确认 OOM。Old Gen 仅剩 0.36MB 可用，Eden 满，Survivor 全空（对象直接晋升 Old Gen）
```

## 第三步：分析 jmap.txt — 定位内存大户

**第三步输出：**

内存占用 TOP 5：

| 排名 | 类名 | 实例数 | 内存 | 含义 |
|------|------|--------|------|------|
| 1 | xmlbeans AttrXobj | 4,759,106 | 436MB | POI xlsx XML DOM 属性节点 |
| 2 | xmlbeans ElementXobj | 3,741,238 | 343MB | POI xlsx XML DOM 元素节点 |
| 3 | [C (char[]) | 6,441,501 | 267MB | 字符串底层存储 |
| 4 | java.lang.String | 6,517,596 | 149MB | 字符串对象 |
| 5 | XSSFCell + STCellRefImpl + CTCellImpl | ~1,544,195 each | ~136MB | Excel 单元格对象 |

关键业务对象：
- `PurchaseOrderDto` — 139,786 个, 29MB（Dubbo 返回的采购单）
- `PurchaseOrderTravellerDto` — 149,074 个, 18MB（出行人数据）
- `HotelOrderShowForm` — 90,844 个, 7MB（导出用的展示对象）
- `XSSFRow` / `CTRowImpl` — 90,835 个（Excel 行数）

关键发现：**xmlbeans/POI 对象排名前列 → 命中"Excel 导出场景 OOM"**。XSSFRow 90,835 行 ≈ HotelOrderShowForm 90,844 个，说明是把全量数据写入了 XSSFWorkbook。

## 第四步：分析 gc.txt — 判断 GC 状态

**第四步输出：**
```
GC 状态：YGC=5763次/105.6s, FGC=294→299次/617→625s
Old Gen 使用率：99.98%, Eden：100%
平均每次 Full GC 耗时：617/294 ≈ 2.1s
采样期间 FGC 从 294 增长到 299（5 次），FGCT 增长 8.3s
结论：确认 GC thrashing。JVM 在疯狂 Full GC 但回收不了内存，每次 FGC 耗时 ~1.7s
```

## 第五步：分析 pod.log — 还原事件时间线

按 skill 要求的搜索策略依次执行：

这里有新发现。让我继续追踪。

重大发现 — 用户不止点了一次导出。让我确认第二次和第三次导出的耗时日志。

只有第一次导出打出了耗时日志，第二次（09:59:37）和第三次（10:03:48）都没有打出耗时日志，说明它们在数据拉取阶段就已经因为内存不足而卡住或失败了。

## 第五步输出

**事件时间线（修正版）：**

| 时间 | 线程 | 事件 | 对应代码方法 |
|------|------|------|------------|
| 09:56:59 | exec-125 | 分页查询，操作人 10614 | `getPurchaseOrder()` |
| 09:57:06 | exec-125 | 分页查询返回：总条数 90834，本次 10 条 | `getPurchaseOrder()` |
| 09:57:36 | exec-122 | **第1次导出**，操作人 10614，同样条件 | `exportHotel()` |
| 09:57:51 | exec-122 | 第1次导出数据拉取完成，耗时 14689ms | `exportHotel()` |
| 09:59:37 | exec-154 | **第2次导出**（用户可能等不及又点了一次） | `exportHotel()` |
| 10:02:38 | exec-152 | **第3次导出**（又点了一次） | `exportHotel()` |
| 10:04:11 | exec-192 | 首次 OOM：`Java heap space` | — |
| 10:05:27 | — | Dubbo 解码 OOM，系统不可用 | — |

关键发现：**用户连续点了 3 次导出**，第 2、3 次都没有打出"耗时"日志，说明在数据拉取或 Excel 生成阶段就已经内存不足。多次导出请求叠加，加剧了内存压力。

## 第六步：分析 top.txt — 确认系统资源状态

**第六步输出：**
```
CPU：85.1% us，3 个 GC task 线程占据 81.2% + 62.5% + 56.2% = 199.9% CPU → 确认 GC thrashing
内存：RES = 3.4GB，进程虚拟内存 6.4GB
线程：164 个，正常范围，排除线程泄漏
Load average：11.06（远超 CPU 核数），系统严重过载
```

## 第七步：关联代码 — 定位根因

**第七步输出 — 代码与日志严格对应：**

| 日志内容 | 对应方法 | 调用的 Dubbo 接口 | 说明 |
|---------|---------|-----------------|------|
| `酒店采购单分页查询, 操作人：10614` | `getPurchaseOrder()` | `getOrderAndTravellerOrder(queryDto)` — 分页接口 | 只取 10 条，无问题 |
| `酒店采购单分页查询总条数：90834, 本次条数: 10` | `getPurchaseOrder()` | 同上 | 揭示数据总量，本身不导致 OOM |
| `酒店采购单导出查询,操作人：10614` | `exportHotel()` | `getOrderAndTravellerOrderList(queryCondition)` — **全量接口，无分页** | 拉取全部 ~9万条 |
| `酒店采购单导出查询列表数据耗时：14689` | `exportHotel()` | 同上 | 数据拉取完成后开始 POI 生成 |

---

## 最终分析报告

### 1. 事件时间线

| 时间 | 事件 |
|------|------|
| 09:56:59 | 用户 10614 分页查询酒店采购单，返回总条数 90,834（只取 10 条，正常） |
| 09:57:36 | 用户点击导出（第1次），exec-122 线程，调用全量接口拉取 ~9万条数据 |
| 09:57:51 | 第1次导出数据拉取完成（14.7s），开始 POI 生成 xlsx |
| 09:59:37 | 用户又点了导出（第2次），exec-154 线程，再次拉取全量数据 |
| 10:02:38 | 用户又点了导出（第3次），exec-152 线程 |
| 10:04:11 | OOM：`java.lang.OutOfMemoryError: Java heap space` |
| 10:05:27 | Dubbo 解码也 OOM，服务完全不可用 |

### 2. 直接原因

`HotelPurchaseOrderController.exportHotel()` 调用无分页的 `getOrderAndTravellerOrderList()` 一次性加载 ~9万条数据到内存，再用 POI XSSFWorkbook 生成 xlsx。用户连续点了 3 次导出，多个请求叠加，内存压力成倍放大。

### 3. 内存占用分析

| 对象 | 实例数 | 内存 | 来源 |
|------|--------|------|------|
| xmlbeans AttrXobj | 4,759,106 | 436MB | POI xlsx DOM |
| xmlbeans ElementXobj | 3,741,238 | 343MB | POI xlsx DOM |
| XSSFCell | 1,544,195 | 47MB | Excel 单元格 |
| PurchaseOrderDto | 139,786 | 29MB | Dubbo 返回数据（可能含多次请求的叠加） |
| HotelOrderShowForm | 90,844 | 7MB | 导出转换对象 |
| XSSFRow | 90,835 | 2MB | Excel 行 |

### 4. GC 状态

确认 GC thrashing：FGC 294→299 次，每次 ~2s，Old Gen 99.98%，3 个 GC 线程吃掉 200% CPU。

### 5. 代码定位

文件：`test-app/src/main/java/com/test/app/action/TestController.java`

问题方法：`exportHotel()`

问题代码：
```java
// 全量接口，无分页、无数据量限制
List<PurchaseOrderDto> orderList = purchaseOrderService.getOrderAndTravellerOrderList(queryCondition).getData();
```

注意区分：`getPurchaseOrder()` 方法调用的是分页接口 `getOrderAndTravellerOrder(queryDto)`，日志 `总条数：90834, 本次条数: 10` 是这个方法打的，它本身没有问题。

### 6. 修复建议

1. 导出加数据量上限（如 1 万条），超出提示缩小查询范围
2. 前端加防重复点击，避免用户连续触发多次导出
3. 将 XSSFWorkbook 替换为 SXSSFWorkbook（流式写入）或 EasyExcel
4. 导出改为异步任务 + 分页拉取数据

---

### Skill 验证结论

按 SKILL.md 的 7 步流程走下来，这次比之前的分析多发现了一个关键事实：**用户连续点了 3 次导出**（09:57:36、09:59:37、10:02:38），而不是只有 1 次。第 2、3 次导出都没有打出"耗时"日志，说明在数据拉取阶段就已经受到内存压力影响。这个细节在之前的分析中被遗漏了。

同时，日志和代码的对应关系这次严格区分了：`总条数：90834` 是分页查询方法打的，不是导出方法打的。Skill 中"注意事项"第 1 条起到了纠偏作用。


