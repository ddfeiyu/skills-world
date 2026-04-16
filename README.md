# skills-world
用好skill 走遍天下都不怕


# OOM Dump 分析 Skill

## 触发条件

当用户提供 JVM OOM（OutOfMemoryError）相关的 dump 文件、日志文件，要求分析 OOM 原因时调用此 skill。

常见触发信号：
- 用户提到 "OOM"、"OutOfMemoryError"、"内存溢出"、"堆溢出"、"heap space"
- 用户提供了 dump 目录，包含 jmap、gc、heap、thread、pod.log 等文件
- 用户要求分析某个服务为什么挂了、为什么内存爆了
