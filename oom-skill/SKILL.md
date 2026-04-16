# OOM Dump 分析 Skill

## 触发条件

当用户提供 JVM OOM（OutOfMemoryError）相关的 dump 文件、日志文件，要求分析 OOM 原因时调用此 skill。

常见触发信号：
- 用户提到 "OOM"、"OutOfMemoryError"、"内存溢出"、"堆溢出"、"heap space"
- 用户提供了 dump 目录，包含 jmap、gc、heap、thread、pod.log 等文件
- 用户要求分析某个服务为什么挂了、为什么内存爆了

## 分析流程

### 第一步：定位并读取 dump 文件

按优先级依次读取以下文件（不是所有文件都会存在）：

| 文件 | 作用 | 优先级 |
|------|------|--------|
| `memory_heap.txt` | JVM 堆配置和使用率，确认是否真的 OOM | P0 |
| `jmap.txt` | 对象实例直方图，定位哪些对象吃掉了内存 | P0 |
| `gc.txt` | GC 统计，判断是否 GC thrashing | P0 |
| `pod.log` | 应用日志，找到 OOM 前的请求和堆栈 | P0 |
| `top.txt` | 系统资源使用，确认 CPU 是否被 GC 线程占满 | P1 |
| `threadCount.txt` | 线程数，排除线程爆炸导致的 OOM | P1 |
| `threads.txt` | 线程 dump，分析线程在做什么 | P2 |
| `memory_dump.hprof` | 堆转储文件（通常太大无法直接读取，记录存在即可） | P2 |

### 第二步：分析 memory_heap.txt — 确认 OOM 事实

关注点：
- `MaxHeapSize`：最大堆大小
- `Eden Space` / `Old Generation` 的 used 百分比
- 如果 Old Gen 使用率 > 95% 且 Eden 100%，确认堆内存耗尽
- 记录 Survivor Space 使用情况，如果 Survivor 为 0% 说明对象直接晋升到 Old Gen

输出模板：
```
堆配置：MaxHeap = {X}MB, NewSize = {Y}MB, OldSize = {Z}MB
堆使用：Eden {A}% used, Old Gen {B}% used, Survivor {C}% used
结论：{是否确认 OOM}
```

### 第三步：分析 jmap.txt — 定位内存大户

关注点：
- 按 `#bytes` 降序排列的前 30 个类
- 重点识别以下几类对象：
  - **业务 DTO 类**（如 `com.xxx.dto.*`）：说明大量业务数据被加载到内存
  - **Apache POI 相关类**（`org.apache.xmlbeans.*`、`org.apache.poi.xssf.*`、`org.openxmlformats.*`）：说明 Excel 导出导致 OOM
  - **集合类**（`HashMap$Node`、`ArrayList`、`TreeMap$Entry`）：大量集合可能是数据容器
  - **byte[] / char[]**：大量原始数据缓冲
- 将业务类的实例数与业务场景关联（如 90,844 个 Form 对象 ≈ 导出的数据行数）

分析技巧：
- 如果 xmlbeans/POI 对象排名前列 → Excel 导出场景 OOM
- 如果业务 DTO 数量巨大 → 全量查询无分页
- 如果 byte[] 占比极高 → 可能是大文件/大报文加载
- 如果 Thread 对象数量异常 → 线程泄漏

输出模板：
```
内存占用 TOP 5：
1. {类名} - {实例数} 个, {内存}MB — {含义}
2. ...

关键发现：{哪类对象是内存大户，指向什么场景}
```

### 第四步：分析 gc.txt — 判断 GC 状态

关注点：
- `FGC`（Full GC 次数）和 `FGCT`（Full GC 总耗时）
- `O`（Old Gen 使用率）是否持续 99%+
- `E`（Eden）是否持续 100%
- `S0` / `S1`（Survivor）是否都是 0%
- 多行采样之间 FGC 是否在快速增长

判断标准：
- FGC 频繁增长 + Old Gen 99% + Eden 100% → **GC thrashing**，JVM 在疯狂 GC 但回收不了内存
- FGCT / FGC = 平均每次 Full GC 耗时，如果 > 2s 说明 GC 非常吃力

输出模板：
```
GC 状态：YGC={X}次/{Y}s, FGC={A}次/{B}s
Old Gen 使用率：{C}%, Eden：{D}%
结论：{是否 GC thrashing}
```

### 第五步：分析 pod.log — 还原事件时间线

关注点：
1. **搜索 OOM 关键字**：`grep -i "OutOfMemory\|oom\|GC overhead\|heap space"` 定位 OOM 发生的精确时间
2. **向前回溯**：从 OOM 时间点往前找触发请求，重点关注：
   - 导出类请求（`export`、`download`、`excel`、`xlsx`）
   - 大数据量查询（日志中出现大数量的记录）
   - 耗时长的请求（从请求开始到 OOM 之间的时间差）
3. **识别请求链路**：同一个线程（如 `http-nio-8080-exec-122`）从接收请求到 OOM 的完整过程
4. **关联代码**：日志中的 Controller 类名 + 方法名 → 在代码库中找到对应源码

搜索策略（按顺序执行）：
```bash
# 1. 找 OOM 错误
grep -n -i "OutOfMemory\|heap space" pod.log

# 2. 找导出/下载相关请求
grep -n -i "export\|download\|excel\|xlsx" pod.log

# 3. 找大数据量相关日志
grep -n -i "总条数\|total\|count\|size" pod.log

# 4. 找耗时日志
grep -n -i "耗时\|elapsed\|cost\|duration" pod.log
```

输出模板：
```
事件时间线：
{时间} - {事件描述}
{时间} - {事件描述}
...
触发请求：{URL}，操作人：{用户}，线程：{线程名}
```

### 第六步：分析 top.txt — 确认系统资源状态

关注点：
- CPU 使用率：如果 `GC task` 线程占据了大部分 CPU → 确认 GC thrashing
- 内存使用：RES（常驻内存）是否接近容器限制
- load average：系统负载是否异常高

### 第七步：关联代码 — 定位根因

根据日志中的 Controller/Service 类名，在代码库中查找对应源码：
1. 找到触发请求对应的 Controller 方法
2. 分析方法中的数据查询逻辑：是否有分页？是否一次性加载全量数据？
3. 分析数据处理逻辑：是否使用了内存密集型操作（如 POI XSSFWorkbook）？
4. **严格区分不同方法的日志**：同一个 Controller 中可能有分页查询方法和全量导出方法，日志要和方法一一对应，不能混淆

代码分析要点：
- 分页查询方法（如 `getOrderAndTravellerOrder`）和全量查询方法（如 `getOrderAndTravellerOrderList`）是两个不同的接口
- 分页查询的日志（如"总条数：90834, 本次条数: 10"）只说明数据总量，本身不会导致 OOM
- 全量导出方法才是 OOM 的真正触发点，要找到它自己的日志（如"导出查询,操作人"、"导出查询列表数据耗时"）

## 输出格式

最终分析报告应包含以下部分：

### 1. 事件时间线
按时间顺序列出关键事件。

### 2. 直接原因
一句话说明 OOM 的直接原因（如：酒店采购单导出一次性加载 9 万条数据 + POI XSSF 全量内存模型）。

### 3. 内存占用分析
表格列出 jmap 中的关键对象及其含义。

### 4. GC 状态
说明 GC 是否处于 thrashing 状态。

### 5. 代码定位
指出具体的代码文件、方法、以及问题代码行。
**注意**：必须准确区分日志和代码方法的对应关系，不能将分页查询的日志归因于导出方法。

### 6. 修复建议
给出具体可操作的修复方案。

## 常见 OOM 场景速查

| 场景 | jmap 特征 | 日志特征 | 典型修复 |
|------|----------|---------|---------|
| Excel 导出 | xmlbeans/POI 对象排前列 | export/download 请求 | 改用 SXSSFWorkbook 或 EasyExcel |
| 全量查询无分页 | 业务 DTO 数量巨大 | 大数据量查询日志 | 加分页/加数据量上限 |
| 大文件加载 | byte[] 占比极高 | 文件上传/下载请求 | 流式读写 |
| 线程泄漏 | Thread 对象数量异常 | 线程数持续增长 | 修复线程池配置 |
| 缓存未清理 | HashMap/ConcurrentHashMap 巨大 | 无明显触发请求 | 加缓存淘汰策略 |
| Dubbo 大报文 | Hessian 反序列化对象多 | Dubbo decode 报错 | 限制返回数据量 |

## 注意事项

1. **日志和代码必须严格对应**：同一个 Controller 中不同方法打的日志要区分清楚，不能张冠李戴
2. **分页查询 ≠ 全量查询**：分页查询返回的总条数只说明数据规模，不代表这些数据都被加载到了内存
3. **jmap 实例数是关键证据**：jmap 中业务对象的实例数可以和日志中的数据量交叉验证
4. **时间线要完整**：从触发请求到 OOM 发生之间可能有几分钟的延迟（数据加载 + Excel 生成），不要遗漏中间环节
5. **hprof 文件通常无法直接读取**：记录其存在即可，建议用户使用 MAT（Memory Analyzer Tool）或 VisualVM 进一步分析
