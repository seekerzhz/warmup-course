# 入场券自测

进 LSM 之前先过这一关。12 道题，答上 9 道、并且能当场演示对应的现象，就可以进 [LSM 主轨](https://docs.hanzel.top/lsm/)。

判定标准只有一条：**能跑出来给你看**。「我知道 fsync 比 flush 更狠」不算过；「我能让你看着我 `kill -9` 然后丢数据，加上 fsync 就不丢」才算过。

答不上来的题不用猜。每题后面标了回退位置，回去补那一个小节，比重刷整章快得多。

## 一、工程能力（构建与拆分）

**1. 你能独立写一个带 `CMakeLists.txt` 的 3 文件 C++ 工程吗？**
要求：`note.hpp` / `note.cpp` / `main.cpp`，`cmake -S . -B build && cmake --build build` 一条龙能跑，无警告。
答不上 → 回[轮子 1 · CMakeLists 是最小构建脚本](01-cli-tool.md)

**2. 看到 `undefined reference to 'foo()'`，你能在 30 秒内说出至少两种原因吗？**
答不上 → 回[轮子 1 · 声明和定义是两回事](01-cli-tool.md)

**3. 改了 `.cpp` 和改了 `CMakeLists.txt`，分别需要重跑哪条命令？**
答不上 → 回[轮子 1 · 完成标志第 1 条](01-cli-tool.md)（那里写了三步命令的差别）

## 二、持久化（格式与容错）

**4. 你能用 `fstream` 读写自定义二进制格式，并同时处理「文件不存在」和「文件损坏」两种情况吗？**
要求：删掉文件能正常启动；改坏一个字节能打印错误并退出，不是崩、不是输出乱码。
答不上 → 回[轮子 2 · 检查失败，而不是假设成功](02-persistence.md)

**5. 为什么二进制文件写入必须带 `std::ios::binary`？不加会怎样？**
答不上 → 回[轮子 2 · 二进制 IO 就这几行](02-persistence.md)

**6. 你的文件格式里有版本号吗？旧程序读到新版本文件，会发生什么？**
要求：能说出「明确报错」和「硬解析出垃圾」两种结果的区别，并说明你的实现属于哪种。
答不上 → 回[轮子 2 · 版本号怎么用](02-persistence.md)

**7. 为什么写入要用「临时文件 + rename」，而不是直接覆盖？**
答不上 → 回[轮子 2 · 写入要用「临时文件 + 改名」](02-persistence.md)

## 三、访问路径（分层与代价）

**8. 50 万条数据，你能把启动时间做到 100 ms 以内吗？靠的是什么？**
要求：能说清「磁盘存数据 + 内存存地图」这个结构，以及你的索引条目多少字节。
答不上 → 回[轮子 3 · 第一步：稠密索引](03-index.md)

**9. 稀疏索引比稠密索引省了什么、多花了什么？为什么 LSM 的 SSTable 也这么干？**
答不上 → 回[轮子 3 · 第二步：稀疏索引](03-index.md)

**10. 你的 offset 用什么类型？为什么不能用 `uint32_t`？**
答不上 → 回[轮子 3 · 两个必须想清楚的设计点](03-index.md)

## 四、崩溃恢复（这一组最硬）

**11. 你能解释 `fsync` 和 `flush` 的区别吗？**
要求：能说出三个层次（C++ 缓冲 / 页缓存 / 磁盘），并指出 `write` 返回成功只保证到哪一层。
答不上 → 回[轮子 4 · 竞赛生映射](04-wal-recovery.md) 和 [轮子 4 · `flush` 不等于 `fsync`](04-wal-recovery.md)

**12. 你能复现「不 fsync 就 `kill -9` 导致丢数据」这个现象吗？**
要求：当场演示对照组 —— 不 fsync 丢、加 fsync 不丢，条数差异看得见。
答不上 → 回[轮子 4 · 完成标志第 2 条](04-wal-recovery.md)

**13. 日志尾部有一条写了一半的记录，你的恢复流程怎么处理？**
要求：能说出「CRC 不过 / 长度不合理 → 停止扫描 → 截断到上一条完整记录」，并且真的截断了文件。
答不上 → 回[轮子 4 · 崩溃发生在任意位置](04-wal-recovery.md)

**14. 你的重放是幂等的吗？重放两次结果一样吗？**
答不上 → 回[轮子 4 · 幂等的关键](04-wal-recovery.md)

## 五、收尾（选做轮，但建议过）

**15. `kill` 之后进程为什么没有 `terminate called`？你怎么保证线程干净退出？**
答不上 → 回[轮子 5 · 优雅关闭](05-concurrency.md)

**16. `-fsanitize=thread` 跑过吗？输出是干净的吗？**
答不上 → 回[轮子 5 · `-fsanitize=thread` 是你的裁判](05-concurrency.md)

## 手把手自测流程

把这段照着做一遍，20 分钟，比看十遍文档有用。假设你的程序在 `./build/note`：

```bash
# 1. 干净构建
rm -rf build && cmake -S . -B build && cmake --build build -j
#   → 应该零警告。有警告就是 -Wall -Wextra 没开，回去加

# 2. 空文件启动
rm -f notes.bin wal.log
./build/note list          # 正常输出空列表，不报错

# 3. 写入并验证持久化
./build/note add "第一条"
./build/note add "第二条"
./build/note list | wc -l  # 应该是 2
# 退出程序，重新启动
./build/note list | wc -l  # 还是 2 → 过了第 4 题

# 4. 损坏检测
cp notes.bin notes.bin.bak
printf '\x00\x00\x00\x00' | dd of=notes.bin bs=1 seek=20 conv=notrunc 2>/dev/null
./build/note list          # 应该报错退出（退出码非 0），不是崩
mv notes.bin.bak notes.bin # 恢复

# 5. 崩溃恢复
rm -f wal.log
for i in $(seq 1 1000); do ./build/note add "n$i" > /dev/null; done &
PID=$!
wait_for=200
while [ $(./build/note list | wc -l) -lt $wait_for ]; do sleep 0.05; done
kill -9 $PID 2>/dev/null
./build/note list | wc -l  # 至少 200 条，且每条内容完整

# 6. 日志截断
truncate -s -50 wal.log
./build/note list | head -1  # 能正常起来，不崩
#   → 过了第 13 题

# 7. 并发（如果做了轮子 5）
cmake -S . -B build-tsan -DCMAKE_CXX_FLAGS="-fsanitize=thread -g -O1" \
  && cmake --build build-tsan -j
./build-tsan/note add "tsan" 2>&1 | grep -i "warning: ThreadSanitizer"
#   → 没有输出 = 过了第 16 题
```

## 通过之后

12 题（第 1–14 题里任选 12）答上 9 道，进 [LSM 阶段 0](https://docs.hanzel.top/lsm/)。

如果只答上 5–8 道，做**最短路径**就够进 LSM 阶段 0–2 了 —— 缺的那几块多半在轮子 3 和 5，等你到 LSM 阶段 3 之前再回来补。

如果答上不到 5 道，不要硬进。把轮子 1 和轮子 2 重做一遍（不是重读，是重写代码），把构建和持久化变成肌肉记忆。这两块是所有后续工作的地基，地基歪了，上面每一层都要还债。
